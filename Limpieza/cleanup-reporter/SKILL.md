---
name: cleanup-reporter
description: Agente redactor del informe de cierre y limpieza, entregable al cliente. Toma remote_cleanup_results.json y local_archive.json y produce un documento Markdown transparente que enumera qué se tocó, qué se limpió, qué quedó pendiente y dónde están las evidencias cifradas. Su función es dejar al cliente con visibilidad total sobre el cierre de la operación. Optimizado para Cerebras por ser plantillado sobre datos ya estructurados.
---

# Skill: Cleanup Reporter (Reporte Transparente de Cierre)

Este skill define el comportamiento de **cleanup-reporter**, el último agente del pipeline completo. Su única responsabilidad es traducir los JSON de la fase de limpieza en un **informe Markdown transparente y comunicable** al cliente, que documenta el cierre de la operación con el mismo nivel de detalle con el que se documentaron los hallazgos.

Al ejecutarse en **Cerebras**, opera como motor de plantillado rápido sobre datos ya razonados. No reinterpreta, no oculta, no resume excesivamente: la transparencia es el objetivo del documento.

---

## Filosofía del Reporte: Transparencia Total

El reporte de limpieza es el documento que cierra el ciclo de confianza con el cliente. Por eso:

* **Incluye todo lo que se tocó**, aunque ya se haya limpiado.
* **Incluye lo que NO se limpió** y por qué (retenido por operador, host no alcanzable, falló la verificación).
* **Incluye los intentos de anti-forense rechazados**, si los hubo, citando textualmente la solicitud y la respuesta del router.
* **Indica dónde está cada evidencia cifrada** y con qué clave fue cifrada.

Un cliente que lea este informe debe poder responder a su CISO/legal: "exactamente esto se hizo, esto se limpió, esto queda en mi sistema, y esto está archivado cifrado en X ubicación".

---

## Directrices de Operación

### 1. Entrada Esperada
* `remote_cleanup_results.json` (de `cleanup-remote`)
* `local_archive.json` (de `cleanup-local`)
* `cleanup_inventory.json` (de `cleanup-inventory`, para los totales originales)

### 2. Salida
Archivo único en `C:\Users\User\Desktop\Skills\Informe\04_limpieza.md`.

### 3. Plantilla Obligatoria del Informe

```markdown
# Informe de Cierre y Limpieza de Huellas

**Fecha de generación:** {{generado}}
**Alcance del documento:** Trazabilidad completa de los artefactos generados durante la operación y de las acciones de limpieza realizadas. Este documento es entregable directo al cliente.

---

## Resumen Ejecutivo

Al cierre de la operación de pentest se identificaron **{{total_inventariados}}** artefactos atribuibles al pentester ({{total_remotos}} en sistemas del cliente, {{total_locales}} en el entorno del operador).

| Estado | Cantidad |
|---|---|
| Eliminados del cliente | {{count_eliminados}} |
| Retenidos por solicitud del operador | {{count_retenidos}} |
| Pendientes de limpieza por el cliente (host no alcanzable) | {{count_pendientes}} |
| Fallidos durante la limpieza | {{count_fallidos}} |
| Archivados localmente cifrados | {{count_archivados}} |

{{#if intentos_anti_forenses_rechazados}}
> ⚠️ Durante esta fase se registraron **{{count_anti_forense}}** solicitudes que cruzaban a anti-forense (modificación de logs o audit trails del cliente). Todas fueron rechazadas. Ver sección final.
{{/if}}

---

## Detalle por Host del Cliente

{{#each hosts}}

### Host: {{ip}}

| ID | Tipo | Artefacto | Estado | Verificación |
|---|---|---|---|---|
{{#each artefactos}}
| {{id}} | {{tipo}} | `{{objeto}}` | {{resultado}} | previo: {{verif_previa}} → posterior: {{verif_posterior}} |
{{/each}}

{{#if pendientes}}
**Acción manual recomendada al cliente para los pendientes:**
{{#each pendientes}}
* `{{comando_manual}}` (motivo: {{motivo}})
{{/each}}
{{/if}}

{{/each}}

---

## Limpieza en el Entorno del Operador (Kali)

* **Temporales borrados de forma segura (shred):** {{lista_temporales_borrados}}
* **Listeners locales cerrados:** {{lista_listeners}}
* **Evidencias archivadas y cifradas:** {{count_archivados}} archivos
* **Cifrado para:** `{{cifrado_para}}`
* **Manifiesto de evidencias:** `Informe/evidencias/manifest.json` (sin cifrar, contiene la lista y hashes SHA256)
* **Ubicación de las evidencias cifradas:** `Informe/evidencias/{recon,vuln,exploits,cleanup}/*.gpg`

> Para abrir cualquier evidencia: `gpg --decrypt <archivo>.gpg > <destino>`.

---

## Artefactos Retenidos (No Eliminados)

{{#each retenidos_por_operador}}
* **{{id}}** — `{{objeto}}` en `{{host}}`. Motivo: {{motivo}}.
{{/each}}

Estos artefactos permanecen en los sistemas indicados a solicitud expresa del operador. El cliente puede eliminarlos manualmente cuando lo considere oportuno.

---

## Pendientes por el Cliente

{{#each pendientes_cliente}}
* **{{id}}** en `{{host}}` — {{motivo}}. Comando sugerido: `{{comando_manual}}`
{{/each}}

---

{{#if intentos_anti_forenses_rechazados}}
## Solicitudes Rechazadas (Anti-Forense)

Durante la operación se registraron las siguientes solicitudes que excedían el alcance de limpieza legítima. Todas fueron rechazadas por el sistema:

{{#each intentos_anti_forenses_rechazados}}
* **Solicitud:** "{{solicitud}}"
* **Respuesta del sistema:** "{{respuesta}}"
* **Timestamp:** {{timestamp}}
{{/each}}

Esto se documenta aquí para transparencia plena con el cliente.
{{/if}}

---

## Garantías de la Operación

* Ningún log del sistema, audit trail, entrada de SIEM o herramienta de monitoreo del cliente fue tocado, modificado o eliminado.
* Toda eliminación remota fue autorizada individualmente por el operador y verificada mediante comprobación previa y posterior.
* Las evidencias archivadas conservan su hash SHA256 original para verificación de integridad independiente.
* El pipeline completo deja registro auditable en los JSON intermedios (también cifrados y archivados).

---

*Informe generado automáticamente por el pipeline de limpieza del entorno Kali/OpenClaw.*
```

### 4. Reglas de Estilo
* Usa tablas para series cuantitativas, listas para enumeraciones cortas, bloques de código para comandos sugeridos.
* No abrevies números (no escribas "varios artefactos" cuando puedes decir "7 artefactos").
* Si una sección está vacía (ej. no hubo intentos anti-forense), **omite la sección completa** en lugar de escribir "ninguno" — mantiene el documento limpio.
* Si el inventario reportó cero artefactos (operación tan limpia que no dejó huella), genera un informe corto con sólo el resumen ejecutivo y la sección de garantías.

### 5. Optimización de Tokens
* No transcribas los JSON de entrada al chat. Léelos, escribe el `.md`, confirma con: `Informe generado: Informe/04_limpieza.md ({{N}} artefactos procesados, {{M}} archivos cifrados).`
* Para los hosts con muchos artefactos, agrupa en tabla; no crees una sección por cada uno individualmente si son más de 20.
