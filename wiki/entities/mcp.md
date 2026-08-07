---
type: entity
date_updated: 2026-08-07
source_count: 3
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

## No solo por red: transporte `stdio` local

MCP es un protocolo de **transporte**, no un servicio que exija estar en la nube o siquiera en red. [[17-creating-local-ai-agents]] lo demuestra conectando a un servidor MCP como proceso local, hablado por entrada/salida estándar, sin abrir ningún puerto:

```python
params = StdioServerParameters(command=parts[0], args=parts[1:])
async with stdio_client(params) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()
        tools = await session.list_tools()
```

Misma advertencia de seguridad que las tools locales de esa lección: un servidor MCP local corre con los permisos del usuario que lo lanza, así que hay que acotarlo a un directorio concreto y tratar sus salidas como entrada a validar, no como verdad de confianza.

## Dónde aparece en el curso

- `11-agentic-protocols/code_samples/mcp-agents` — servidor y cliente MCP con el SDK de Python directo, sin frameworks. Es la base conceptual.
- `github-mcp` — usa el servidor MCP oficial de GitHub (`npx @modelcontextprotocol/server-github`) dentro de una app Chainlit con 3 agentes. **Ojo: eso no es A2A**, es orquestación con `WorkflowBuilder`.
- [[microsoft-agent-framework]] interopera con MCP para herramientas externas.
- [[nlweb]] expone cada sitio web como servidor MCP.
- [[17-creating-local-ai-agents]] — conexión `stdio` a un servidor MCP local, opcional y condicionada a una variable de entorno.

Fuentes: [[11-agentic-protocols]], [[17-creating-local-ai-agents]]
