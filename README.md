# 🛡️ OpenClaw Security Agents Framework

Un entorno modular y avanzado de agentes de Inteligencia Artificial especializados en auditoría técnica y ciberseguridad, diseñado para ejecutarse sobre **Kali Linux** utilizando el framework **OpenClaw**. 

Este proyecto implementa una arquitectura multi-agente que divide las fases de un test de intrusión (Reconocimiento, Análisis de Vulnerabilidades, Explotación, Evasión y Reporte) en micro-agentes altamente especializados ("Skills"), optimizando la velocidad y el consumo de tokens.

---

## 🧠 Arquitectura de Modelos Híbrida

El framework está diseñado para balancear la carga de trabajo entre dos potentes motores de IA para lograr la máxima eficiencia y el menor costo operativo:

* **Cerebras (Llama 3):** Encargado de las operaciones de latencia ultra-baja. Actúa como enrutador (*router*), procesador pasivo de herramientas OSINT, y generador veloz de informes Markdown.
* **Claude (Anthropic):** El cerebro analítico. Se encarga de las tareas de alto razonamiento, como diseñar estrategias de escaneo quirúrgico en red, perfilar ataques web (OWASP Top 10), calcular métricas de impacto (CVSSv3) y correlacionar vulnerabilidades complejas.

---

## 📂 Estructura del Proyecto

El sistema está dividido en módulos lógicos que representan las fases de una auditoría de seguridad:

### 1. 🔍 Reconocimiento (`/Reconomiento`)
* `recon-router`: Orquestador del flujo de descubrimiento.
* `recon-pasivo`: Recolector OSINT no intrusivo (subdominios, resolución DNS).
* `recon-activo`: Escaneador inteligente de puertos y mapeo de red (Nmap).
* `recon-web`: Perfilador de tecnologías web y directorios.
* `recon-consolidador`: Cerebro lógico que correlaciona activos para definir la superficie de ataque real.
* `recon-reporter`: Redactor del informe técnico de activos.

### 2. ☢️ Vulnerabilidades (`/Vulnerabilidades`)
* `vuln-router`: Orquestador del análisis de seguridad.
* `vuln-scanner-net`: Escáner automatizado dirigido para puertos y servicios (Nuclei/Scripts).
* `vuln-analyst-web`: Analista avanzado para lógica web y fallos estructurales.
* `vuln-evaluator`: Calculador de severidad, descarte de falsos positivos y diseño de remediación.
* `vuln-reporter`: Documentador de la sección técnica de fallos.

### 3. 💥 Exploits (`/Exploits`)
* Entorno reservado para el desarrollo, búsqueda, validación y ejecución de *Proof of Concepts* y exploits estructurados.

### 4. 📝 Informe (`/Informe`)
* `master-reporter`: Agente maestro encargado de leer todo el flujo residual (Recon + Vuln + Exploits) y emitir un Informe Final Ejecutivo, detallando niveles de riesgo e impacto en el negocio con una estética impecable.

---

## ⚙️ Optimización en Kali Linux

El framework implementa estrategias críticas de mitigación de tokens:
1. **Sin Salidas Crudas (*Raw Output*):** Los agentes tienen prohibido leer la salida estándar de comandos masivos.
2. **Piping y Parsing Local:** Las herramientas (como Nmap o Nuclei) exportan sus resultados a archivos (XML, JSONL). Scripts locales o comandos Bash filtran y extraen exclusivamente los datos útiles, entregando a la IA JSONs extremadamente compactos y estructurados.

---

## ⚠️ Disclaimer Legal y Académico

**Este proyecto, sus scripts y arquitecturas de agentes (SKILLS) han sido desarrollados con fines netamente académicos y de investigación en el ámbito de la ciberseguridad defensiva y las pruebas de penetración autorizadas.**

El autor no se hace responsable del mal uso de estas herramientas. Este framework está diseñado para ser ejecutado única y exclusivamente en entornos propios, laboratorios controlados, o sobre infraestructuras donde se cuente con un **consentimiento explícito y por escrito** por parte de los propietarios para realizar auditorías de seguridad.

**El uso de herramientas de explotación y reconocimiento activo contra objetivos sin autorización es un delito. Actúa de forma ética y legal.**
