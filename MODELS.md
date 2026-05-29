# Asignación de Modelos por Agente (OpenRouter)

**Estrategia:** Mínimo consumo. Dos modelos por defecto, sin Claude en el camino crítico salvo upgrade opcional para 3 agentes de alto riesgo.

**Coste estimado por pentest completo:** ~$0.15 USD
**Pentests completos con $15 de saldo OpenRouter:** ~100

---

## Configuración Base (Mínimo Consumo)

### Capa 1 — `meta-llama/llama-3.3-70b-instruct` (12 agentes)

| Precio | Contexto | Ideal para |
|---|---|---|
| $0.10 in / $0.32 out por 1M tokens | 131K | Routing, plantillado, redacción Markdown, filtrado |

Agentes:
* `recon-router`, `recon-pasivo`, `recon-reporter`
* `vuln-router`, `vuln-scanner-net`, `vuln-reporter`
* `exploit-router`, `exploit-searcher`, `exploit-reporter`
* `cleanup-router`, `cleanup-local`, `cleanup-reporter`

### Capa 2 — `deepseek/deepseek-chat-v3` (11 agentes)

| Precio | Contexto | Ideal para |
|---|---|---|
| $0.23 in / $0.91 out por 1M tokens | 131K | Razonamiento sobre código, auditoría técnica, narrativa justificativa |

Agentes:
* `recon-activo`, `recon-web`, `recon-consolidador`
* `vuln-analyst-web`, `vuln-evaluator`
* `exploit-validator`, `exploit-executor`, `exploit-evidence`
* `cleanup-inventory`, `cleanup-remote`
* `master-reporter`

---

## Upgrade Opcional para Producción Real

3 agentes tienen anotado `model_upgrade_premium: anthropic/claude-sonnet-4.5` en su frontmatter. Activar esta sustitución **solo** cuando operes contra clientes reales con alto coste de error:

| Agente | Por qué upgrade | Coste adicional aprox. |
|---|---|---|
| `exploit-validator` | Auditar PoC públicos para detectar backdoors. Un falso negativo aquí = ejecutas malware del autor original contra el target. | ~$0.05 por pentest |
| `exploit-executor` | Decide abortar ante output anómalo en tiempo real. Un mal juicio = incidente operativo con el cliente. | ~$0.10 por pentest |
| `cleanup-remote` | Decide qué tocar en sistemas del cliente. Un mal juicio = borras algo que no era tuyo. | ~$0.05 por pentest |

**Coste con los 3 upgrades:** ~$0.35 por pentest. Aún ~42 pentests con $15.

---

## Optimizaciones Recomendadas

1. **Tier gratuito para testing:** durante desarrollo, sustituye `meta-llama/llama-3.3-70b-instruct` por `meta-llama/llama-3.3-70b-instruct:free` (0$ con límite 20 req/min, 200 req/día).
2. **Prompt caching:** si activas el upgrade premium, configura `cache_control` en el system prompt — Claude Sonnet reduce 90% el coste de inputs repetidos.
3. **Fallback de provider:** OpenRouter permite definir varios providers por modelo. Configura Cerebras → Groq → DeepInfra como fallback para latencia óptima sin romper la cadena.
4. **Telemetría:** activa el dashboard de OpenRouter para alertar si un agente excede 50K tokens en una sola invocación — suele ser señal de que un filtro intermedio se rompió y el modelo está leyendo output crudo.

---

## Variables de Entorno Recomendadas

```bash
export OPENROUTER_API_KEY="sk-or-v1-..."
export OPENROUTER_DEFAULT_PROVIDER="cerebras"   # para Llama 3.3, mejor latencia
export OPENROUTER_HTTP_REFERER="https://kali-agents-openclaw.local"
export OPENROUTER_X_TITLE="Kali Agents Pipeline"
```

Estas dos últimas variables aparecen en el ranking público de apps de OpenRouter; puedes dejarlas en blanco si prefieres no figurar.

---

## Cómo Lee OpenClaw el Modelo

Cada `SKILL.md` declara su modelo en el frontmatter YAML:

```yaml
---
name: exploit-validator
model: deepseek/deepseek-chat-v3
model_upgrade_premium: anthropic/claude-sonnet-4.5
description: ...
---
```

El orquestador debe:
1. Parsear el frontmatter al cargar cada skill.
2. Si la variable de entorno `PIPELINE_MODE=premium` está activa, usar `model_upgrade_premium` cuando exista.
3. Si no, usar `model` directamente.
4. Apuntar todas las llamadas a `https://openrouter.ai/api/v1/chat/completions` con el header `Authorization: Bearer $OPENROUTER_API_KEY`.
