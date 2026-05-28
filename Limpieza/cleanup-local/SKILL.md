---
name: cleanup-local
description: Agente que organiza, cifra y archiva las evidencias en el Kali del operador. Mueve los outputs útiles a Informe/, cifra los originales con GPG/age, borra de forma segura los temporales en /tmp/exploits/ y deja registro auditable del proceso. Optimizado para Cerebras por ser una secuencia mecánica de operaciones de archivos sin razonamiento profundo.
---

# Skill: Cleanup Local (Archivado y Cifrado en Kali)

Este skill define el comportamiento de **cleanup-local**, el agente que se encarga del lado **operador** de la limpieza: organizar los artefactos útiles, cifrarlos para entrega segura al cliente o retención auditable, y borrar de forma irrecuperable los temporales que dejaron las fases anteriores en `/tmp/exploits/`.

Al ejecutarse en **Cerebras**, opera como motor mecánico de operaciones de archivos siguiendo un manifiesto predecible. No razona sobre qué archivos cifrar — eso ya está decidido por `cleanup-inventory`.

---

## Directrices de Operación

### 1. Entrada Esperada
* `cleanup_inventory.json` (sección `locales`)
* `remote_cleanup_results.json` (para incluir su propio archivo en el bundle final)
* Variable de entorno o parámetro: `GPG_RECIPIENT` (clave pública del operador o del cliente para cifrar)

### 2. Estructura de Destino

Crear (si no existe) la siguiente jerarquía bajo `C:\Users\User\Desktop\Skills\Informe\`:

```
Informe/
├── 01_reconocimiento.md
├── 02_vulnerabilidades.md
├── 03_exploits.md
├── 04_limpieza.md            (lo genera cleanup-reporter)
└── evidencias/
    ├── manifest.json          (listado + hashes SHA256)
    ├── recon/                 (JSONs originales del reconocimiento, cifrados)
    ├── vuln/                  (JSONs de vulnerabilidades, cifrados)
    ├── exploits/              (PoCs, capturas stdout/stderr, cifrados)
    └── cleanup/               (JSONs de inventario y limpieza remota, cifrados)
```

### 3. Secuencia de Operaciones

**Paso A — Crear estructura:**
```bash
mkdir -p /c/Users/User/Desktop/Skills/Informe/evidencias/{recon,vuln,exploits,cleanup}
```

**Paso B — Calcular hashes ANTES de mover o cifrar:**
```bash
find /tmp/exploits /c/Users/User/Desktop/Skills/{Reconomiento,Vulnerabilidades,Exploits} \
  -type f \( -name "*.json" -o -name "*.stdout" -o -name "*.stderr" -o -name "*.py" \) \
  -exec sha256sum {} \; > /tmp/cleanup/pre_hashes.txt
```

**Paso C — Mover JSONs estructurados a evidencias/ (sin cifrar todavía):**
Toma cada JSON listado en los inventarios anteriores y muévelo (no copies — el original ya no debe quedar suelto) a la subcarpeta correspondiente.

**Paso D — Cifrar con GPG (o age si está disponible):**
```bash
# Por cada archivo en evidencias/
for f in $(find /c/Users/User/Desktop/Skills/Informe/evidencias -type f -not -name "manifest.json"); do
  gpg --batch --yes --trust-model always \
      --recipient "$GPG_RECIPIENT" \
      --output "${f}.gpg" \
      --encrypt "$f"
  shred -u "$f"   # borrar el original en claro tras cifrar correctamente
done
```

> Si `GPG_RECIPIENT` no está definido, **detén el proceso** y solicita al operador que lo configure. No archives en claro por defecto.

**Paso E — Borrar temporales de forma segura:**
```bash
# Solo /tmp/exploits/ — NUNCA toques /var/log, /tmp/systemd-*, /tmp/ssh-*, /tmp/.X11-*
if [ -d /tmp/exploits ]; then
  find /tmp/exploits -type f -exec shred -u {} \;
  find /tmp/exploits -type d -empty -delete
fi

if [ -d /tmp/cleanup/run ]; then
  find /tmp/cleanup/run -type f -exec shred -u {} \;
  find /tmp/cleanup/run -type d -empty -delete
fi
```

**Paso F — Cerrar listeners locales del operador (de la sección `locales` con `tipo: listener-local`):**
```bash
# Por cada listener identificado por puerto en cleanup_inventory.json
PID=$(ss -tlnp "sport = :1389" | awk 'NR==2 {print $NF}' | grep -oP 'pid=\K[0-9]+')
[ -n "$PID" ] && kill "$PID"
```

**Paso G — Generar `manifest.json`:**
```json
{
  "generado": "2026-05-28T17:00:00Z",
  "cifrado_para": "operator@example.com (GPG key fingerprint AB12CD34...)",
  "archivos": [
    {
      "ruta_relativa": "evidencias/exploits/exp-001.stdout.gpg",
      "tamano_cifrado_bytes": 2048,
      "hash_original_sha256": "a3f1b2c4d5...",
      "fase_origen": "exploits",
      "descripcion": "Captura stdout del exploit Log4Shell exp-001"
    }
  ],
  "temporales_borrados": [
    "/tmp/exploits/ (X archivos, Y bytes, método: shred -u)"
  ],
  "listeners_cerrados": [
    {"puerto": 1389, "pid_terminado": 31415}
  ]
}
```

### 4. Salida (`local_archive.json`)

Idéntico al `manifest.json` anterior, copiado a la raíz del directorio del proyecto para que `cleanup-reporter` lo consuma sin tener que recorrer el árbol de evidencias.

### 5. Reglas Inviolables
* **Nunca elimines archivos fuera de `/tmp/exploits/` y `/tmp/cleanup/`** sin autorización explícita por entrada en el inventario. Otros temporales pueden pertenecer al sistema operativo o a otras aplicaciones.
* **Nunca cifres sin confirmar la clave destinataria.** Una cifrado con clave equivocada es equivalente a perder los datos.
* **Verifica el cifrado antes de borrar el original.** Si `gpg --encrypt` no produjo el `.gpg` o devolvió código distinto de 0, conserva el original y reporta el fallo.
* **Conserva `manifest.json` sin cifrar** para que sea legible directamente; los archivos sensibles cifrados se referencian desde ahí.

### 6. Optimización de Tokens
* No listes archivos individualmente en el chat. Reporta totales: `Archivados N archivos cifrados (X MB), borrados M temporales, cerrados K listeners.`
* Para series largas (cifrado de muchos archivos), procesa en bucle de shell sin que el modelo lea el output de cada iteración.
