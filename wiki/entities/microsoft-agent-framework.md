---
type: entity
date_updated: 2026-08-01
source_count: 8
---

# Microsoft Agent Framework (MAF)

Framework paraguas de Microsoft para agentes. Fusiona **Semantic Kernel** y **AutoGen** en una API única. Paquete: `agent-framework`. Versión usada en este repo: **1.10.0 → 1.11.0**.

## API que hay que conocer

| Pieza | Para qué |
| --- | --- |
| `FoundryChatClient` / `OpenAIChatClient` | Cliente hacia el LLM. Se separa del agente para poder cambiar de proveedor. |
| `client.as_agent(...)` | Crea el agente. **No** existe `create_agent` en este cliente. |
| `@tool(approval_mode=...)` | Convierte una función Python en herramienta → [[tool-calling]] |
| `agent.create_session()` / `get_new_thread()` | Memoria de corto plazo |
| `WorkflowBuilder` | Orquestación como grafo → [[workflows-como-grafo]] |
| `HandoffBuilder` | Derivación entre agentes tipo call center |
| `AgentExecutor` | Envuelve un agente como nodo de workflow |
| `@executor(id=...)` | Convierte una función en nodo del workflow |
| `FunctionInvocationContext` | Contexto que recibe el middleware |
| `default_options=ChatOptions(...)` | Opciones del modelo: `response_format`, `store` |
| `agent_framework.observability` | `get_tracer()`, `get_meter()` — OpenTelemetry integrado |

## Trampas conocidas de la versión instalada

Todas verificadas contra el paquete real, no contra documentación:

- **`Role` no es un enum**, es un `NewType`. `Role.USER` falla. Se usa `Message(role="user", contents=[...])`.
- **`Supports*Tool` son Protocolos**, no clases instanciables. Las tools se piden al cliente: `chat_client.get_web_search_tool()`.
- **`FoundryChatClient` no es async context manager.** Solo `AzureCliCredential` lo es.
- **`STORES_BY_DEFAULT = False`** — sin `store=True` se pierden los reasoning items → [[fix-reasoning-item-workflow]].
- **`SUPPORTS_RICH_FUNCTION_OUTPUT = False`** en el cliente de Foundry — descarta imágenes devueltas por tools → [[fix-imagenes-en-foundry]].
- **`get_outputs()` devuelve por orden de llegada**, no por el orden declarado → [[fix-workflow-concurrente]].
- La telemetría hace `json.dumps` de las definiciones de tools y **no sabe serializar** `AutoCodeInterpreterToolParam` → [[fix-notebook-04-sdk-desactualizado]].

Varios notebooks del curso están escritos contra una API anterior (`Hosted*Tool`, `AzureAIAgentClient`, `ChatMessage`) y no ejecutan sin arreglos.

## Relación con otras piezas

MAF es el **cerebro conversacional** que orquesta. No es un almacén de conocimiento: [[cognee]] o Mem0 cubren la memoria de largo plazo, y MAF los consulta como una tool más → [[13-agent-memory]].

Interopera con [[mcp]] (herramientas externas) y A2A (agentes remotos), y puede alojar agentes de LangChain/LangGraph en [[azure-ai-foundry]].

Fuentes: [[14-microsoft-agent-framework]], [[08-multi-agent]], [[10-ai-agents-production]], [[02-explore-agentic-frameworks]]
