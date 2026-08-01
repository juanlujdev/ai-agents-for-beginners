---
type: entity
date_updated: 2026-08-01
source_count: 2
---

# MCP (Model Context Protocol)

Protocolo estándar para que un agente use **herramientas y datos externos**. Es el mismo que usa Claude. Conecta *un agente con sus recursos*, no agentes entre sí — eso es [[a2a]].

## Más que tool calling

La idea errónea habitual es que MCP solo sirve para pregunta-respuesta rápida. Cuatro capacidades lo hacen apto para tareas largas:

| Capacidad | Qué resuelve |
| --- | --- |
| **Streaming** | El servidor informa avances (`send_progress_notification()`) en vez de bloquear hasta el final |
| **Resumability** | Si cae el **cliente**, al reconectar recibe los eventos perdidos vía `EventStore` |
| **Durability** | Si cae el **servidor**, el progreso sobrevive guardado como *resource* |
| **Multi-turn** | **Elicitation** (preguntar al humano) y **sampling** (pedir ayuda a otro LLM) a mitad de tarea |

## Descubrimiento

Un host MCP **no descubre servidores solo**: hay que configurárselos. Igual que en [[a2a]], necesita una libreta de contactos.

## Dónde aparece en el curso

- `11-agentic-protocols/code_samples/mcp-agents` — servidor y cliente MCP con el SDK de Python directo, sin frameworks. Es la base conceptual.
- `github-mcp` — usa el servidor MCP oficial de GitHub (`npx @modelcontextprotocol/server-github`) dentro de una app Chainlit con 3 agentes. **Ojo: eso no es A2A**, es orquestación con `WorkflowBuilder`.
- [[microsoft-agent-framework]] interopera con MCP para herramientas externas.
- [[nlweb]] expone cada sitio web como servidor MCP.

Fuentes: [[11-agentic-protocols]]
