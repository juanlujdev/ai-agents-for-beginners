---
type: synthesis
date_updated: 2026-08-01
source_count: 1
---

# Fix: notebook 04 de la lección 08 contra `agent_framework` 1.10.0

El caso más largo del curso: **nueve incompatibilidades encadenadas** entre el código del notebook y el SDK realmente instalado. Cada arreglo destapaba el siguiente.

## Causa raíz común

El notebook está escrito contra un SDK anterior — la época del paquete `azure-ai-agents` con clases `Hosted*` y `AzureAIAgentClient`. Lo instalado es `agent_framework` **1.10.0** + `azure-ai-projects` **2.3.0** (y `azure-ai-agents` **no** está instalado). El notebook 03 sí recibió la migración a Foundry; el 04 se quedó atrás.

## Tabla de arreglos

| # | Roto | Correcto en 1.10.0 |
| --- | --- | --- |
| 1 | `HostedWebSearchTool` | `chat_client.get_web_search_tool()` |
| 2 | `HostedCodeInterpreterTool` | `chat_client.get_code_interpreter_tool()` |
| 3 | `agent_framework.azure.AzureAIAgentClient` | `agent_framework.foundry.FoundryChatClient` |
| 4 | `ChatMessage` | `Message` |
| 5 | `azure.ai.agents.models` | `azure.ai.projects.models` |
| 6 | `FoundryChatClient(async_credential=...)` + `async with` | `credential=`, sin `async with` |
| 7 | `chat_client.create_agent(...)` | `chat_client.as_agent(...)` |
| 8 | `as_agent(response_format=X)` | `as_agent(default_options=ChatOptions(response_format=X))` |
| 9 | `Message(Role.USER, text=...)` | `Message(role="user", contents=[...])` |

## Los tres no obvios

**`Supports*Tool` son Protocolos, no clases.** El primer intento fue renombrar `HostedWebSearchTool` → `SupportsWebSearchTool`, que es lo que sugería Python. Falla con `TypeError: Protocols cannot be instantiated`: son interfaces de type-checking. Las herramientas reales se obtienen como **métodos factory del cliente** (`chat_client.get_web_search_tool()`), no instanciando clases sueltas.

**`Role` ya no es un enum.** En 1.10.0 es un `NewType`, así que `Role.USER` da `'NewType' object has no attribute 'USER'`. Además `text=` desapareció del constructor. Trampa adicional: `Message("user", "hola")` posicional interpreta el string como *secuencia de contents* y lo parte en caracteres sueltos — hay que usar `contents=["hola"]` explícito.

**`FoundryChatClient` no es un async context manager.** Solo `AzureCliCredential` lo es. El patrón correcto:

```python
async with AzureCliCredential(process_timeout=60) as credential:
    chat_client = FoundryChatClient(
        project_endpoint=os.environ["AZURE_AI_PROJECT_ENDPOINT"],
        model=os.environ["AZURE_AI_MODEL_DEPLOYMENT_NAME"],
        credential=credential,
    )
```

`process_timeout=60` es un ajuste para Windows: el `az` CLI puede tardar más de los 10 s por defecto en refrescar el token, y salta un `CredentialUnavailableError` a mitad del workflow.

## Bonus: un bug del propio SDK

Con todo arreglado, el flujo llegaba hasta el publisher y moría con `TypeError: Object of type AutoCodeInterpreterToolParam is not JSON serializable`. **No es del notebook:** la capa de telemetría de `agent_framework` hace `json.dumps` de las definiciones de tools para los spans de OpenTelemetry, y no sabe serializar el objeto que devuelve `get_code_interpreter_tool()`.

Workaround verificado — pasar la tool como dict plano, el formato nativo de la Responses API:

```python
tools={"type": "code_interpreter", "container": {"type": "auto"}},
```

## Sobre `BingGroundingTool`

Hubo un rodeo: `BingGroundingTool(connection_id=...)` falla en `azure-ai-projects` 2.3.0, que exige una estructura anidada de tres niveles (`BingGroundingSearchToolParameters` → `BingGroundingSearchConfiguration` → `project_connection_id`). Se aplicó ese fix, y **después** se descubrió que la forma nativa correcta es el factory del cliente:

```python
bing = chat_client.get_bing_grounding_tool(connection_id=conn_id)
```

que acepta `connection_id` plano, como el código original. Requiere moverlo a después de crear `chat_client`. En el notebook `bing` queda definido pero sin conectarse a ningún agente.

## Lección de método

Ante el cuarto símbolo roto se dejó de cazarlos uno a uno y se auditó **la celda de imports entera** contra el SDK instalado. Comprobar la existencia y la firma real de cada símbolo antes de editar evita el bucle de arreglar-ejecutar-fallar; varias sugerencias automáticas de Python (`SupportsWebSearchTool`) eran directamente erróneas.

Fuentes: [[08-multi-agent]]
