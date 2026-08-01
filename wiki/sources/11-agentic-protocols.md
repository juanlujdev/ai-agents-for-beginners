---
type: source-summary
date_updated: 2026-08-01
leccion: 11-agentic-protocols
sesiones: [13cdd450]
fechas_origen: 2026-07-21 .. 2026-07-26
---

# 11 — Protocolos agénticos

Tres protocolos que resuelven problemas distintos: [[mcp]], [[a2a]] y [[nlweb]].

| Protocolo | Conecta |
| --- | --- |
| **MCP** | agente ↔ herramientas/datos |
| **A2A** | agente ↔ agente (autónomos, de empresas distintas) |
| **NLWeb** | humano/agente ↔ sitio web |

Pueden combinarse: un Travel Agent (A2A) delega en un Hotel Agent, que usa NLWeb (que además es servidor MCP) para consultar el catálogo real.

## MCP más allá del tool calling

El README de `code_samples/mcp-agents` argumenta que MCP ya no sirve solo para pregunta-respuesta rápida. Cuatro capacidades lo hacen apto para agentes de verdad:

1. **Streaming** — `send_progress_notification()` en el servidor, `message_handler` en el cliente. "10% analizando dependencias…"
2. **Resumability** — si el **cliente** se desconecta, al reconectar el servidor le reenvía los eventos perdidos gracias a un `EventStore` que guarda cada evento con su ID.
3. **Durability** — si el **servidor** se cae, el progreso sobrevive guardado como *resource* persistente.
4. **Multi-turn** — dos mecanismos:
   - **Elicitation**: el agente pregunta al *humano* (`ctx.session.elicit(message=..., requestedSchema=...)`, respuesta con `action == "accept"`).
   - **Sampling**: el agente pide ayuda a *otro LLM* a mitad de ejecución.

## A2A — las tres dudas que surgieron

**¿Cómo sé que Hertz tiene un agente y cómo me conecto? ¿No es vía MCP?**
No, es mecanismo propio de A2A. Cada agente publica su **Agent Card** en una URL predecible: `https://hertz.com/.well-known/agent.json`. Mismo patrón que `robots.txt`. Tu agente hace un GET, lee el `endpoint` que trae dentro y ya sabe a dónde mandar tareas.

**¿Mi agente necesita saber de antemano qué agentes existen?**
Sí, alguien tiene que decírselo. Dos formas: configuración manual (lista de URLs de Agent Cards) o un **Agent Registry** (catálogo donde las empresas registran sus agentes). Igual que tú: o conoces la web de Hertz, o la buscas en Google. Es lo mismo que ocurre con MCP — un host tampoco descubre servidores solo.

**¿Qué es un Artifact?**
El resultado final del trabajo del agente remoto, empaquetado: el resultado en sí + una descripción de lo completado + el texto de contexto. Detalle importante: **cuando llega el Artifact, la conexión se cierra**. Es "pido tarea → recibo artifact → listo", no una sesión continua.

Piezas de A2A: **Agent Card** (currículum) · **Agent Executor** (pasa el contexto al agente remoto) · **Artifact** (entrega) · **Event Queue** (evita que se corte la conexión en tareas largas).

Por qué no basta MCP: cada agente puede ser de una empresa distinta y usar un LLM distinto. A2A estandariza *cómo se hablan entre agentes*; MCP es para que *un* agente use herramientas.

## NLWeb

Convierte un sitio web en algo consultable en lenguaje natural. El catálogo se convierte en **embeddings** en una base vectorial; la pregunta va al LLM y en paralelo se busca por similitud; la respuesta es en lenguaje natural pero **basada en datos reales del catálogo** (evita alucinaciones). Como NLWeb también funciona como servidor MCP, un agente externo puede llamar a `ask(...)` directamente y recibir JSON.

Componentes: NLWeb Application (motor) · NLWeb Protocol (reglas) · MCP Server (puerta para otros agentes) · Embedding Models · Vector Database.

## Orden de lectura de las subcarpetas

**1º `mcp-agents`**, **2º `github-mcp`**. Al revés, `github-mcp` parece magia (Chainlit, Azure AI Search, Router Agent) sin entender qué pasa debajo.

Para probar `mcp-agents`:
```bash
python -m server.server --port 8006
python -m client.client --url http://127.0.0.1:8006/mcp
```
Se puede simular una caída cortando el cliente con `Ctrl+C` a mitad de una tarea larga: al reiniciar, retoma donde iba.

## `github-mcp` — qué es cada fichero

| Fichero | Qué es |
| --- | --- |
| `README.md` | Guía principal. Si solo lees uno, este. |
| `MCP_SETUP.md` | Troubleshooting de la conexión MCP↔Chainlit. Solo si algo falla. |
| `chainlit.md` | Plantilla de bienvenida por defecto de Chainlit. No es documentación de la demo. |
| `event-descriptions.md` | **Datos**, no docs: ~13 eventos de Microsoft Reactor separados por `---`, que `app.py` indexa en Azure AI Search. Es el catálogo contra el que se hace RAG. |

Inconsistencia detectada: `MCP_SETUP.md` menciona `GITHUB_TOKEN` en `.env`, pero el README y `app.py` usan `GITHUB_PERSONAL_ACCESS_TOKEN` pasado directo al lanzar el servidor con `npx`. **El método correcto es el del README.**

`app.py` monta 3 agentes (Github → Hackathon → Events) con [[microsoft-agent-framework]] sobre Foundry, más un router por expresiones regulares que decide qué agentes activar según palabras clave.

**Dato importante: `github-mcp` NO usa A2A.** Tiene 3 agentes colaborando, pero no hay Agent Cards ni Artifacts — es orquestación con `WorkflowBuilder` dentro de un mismo framework. Conviene no confundir "varios agentes trabajando juntos" con "el protocolo A2A".

Ejecutarlo requiere Node.js + npm, un PAT de GitHub, un proyecto de Foundry con `az login`, y un servicio de **Azure AI Search**. Sin Azure AI Search falla al arrancar, antes incluso de levantar la interfaz.

Ver también: [[08-multi-agent]], [[13-agent-memory]]
