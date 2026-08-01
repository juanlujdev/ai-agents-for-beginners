---
type: source-summary
date_updated: 2026-08-01
leccion: 02-explore-agentic-frameworks
sesiones: [2a8f0bc2, b31fd85b]
fechas_origen: 2026-07-06 .. 2026-07-07
---

# 02 — Explore agentic frameworks

Dos notebooks: `02-python-agent-framework.ipynb` (Foundry) y `02-python-agent-framework-azure-openai.ipynb` (Azure OpenAI directo).

## Qué enseña el notebook de Azure OpenAI, celda a celda

1. Instala `agent-framework`, `agent-framework-openai`, `azure-identity`.
2. Imports: `tool` y `OpenAIChatClient` de `agent_framework`, `AzureCliCredential`, `load_dotenv`.
3. Define la tool `get_random_destination`: lista fija de 10 destinos y una global `_last_destination` para no repetir el anterior.
4. Cliente: `OpenAIChatClient(model=deployment, azure_endpoint=endpoint, credential=AzureCliCredential())`.
5. Agente: `chat_client.as_agent(name=..., instructions=..., tools=[...])`.
6. Ejecución: `agent.create_session()` para memoria entre turnos + `agent.run(..., stream=True)` con salida token a token renderizada en HTML.

Nota de migración del propio notebook: antes usaba Semantic Kernel + GitHub Models (que se retira en julio de 2026); ahora `OpenAIChatClient` contra el endpoint estable `/openai/v1/`.

## El decorador `@tool` — [[tool-calling]]

```python
@tool(approval_mode="never_require")
def check_destination_availability(
    destination: Annotated[str, "The destination to check availability for"]
) -> str:
    """Check if a vacation destination is currently available for booking."""
```

- `approval_mode="never_require"`: se ejecuta sin confirmación humana (apropiado para operaciones de solo lectura).
- El string dentro de `Annotated` **no es un comentario**: es la descripción que lee el modelo para construir la llamada.
- El docstring también se expone al modelo, como descripción de la tool.
- Devuelve texto en lenguaje natural, no JSON, porque quien lo lee es el LLM.

## Variables de entorno: dos flujos distintos

Este notebook necesita variables que el setup inicial **no** crea:

```env
AZURE_OPENAI_ENDPOINT=https://<TU_RECURSO>.openai.azure.com   # URL base, SIN /openai/v1
AZURE_OPENAI_DEPLOYMENT=<nombre-del-deployment>
```

El sufijo `/openai/v1` lo añade `OpenAIChatClient` solo; copiarlo del portal duplica el path y rompe la conexión. Están documentadas en `00-course-setup/README.md`, sección *"Additional Setup for Lessons that Call Azure OpenAI Directly (Lessons 6 and 8)"* — aunque hagan falta ya en la lección 2.

`AZURE_AI_PROJECT_ENDPOINT` (Foundry) y `GITHUB_TOKEN` son de otros flujos y no sirven aquí.

## Errores encontrados

- Celda colgada 8 h por usar `!pip` en vez de `%pip` → [[fix-pip-notebook-colgado]].
- `No tool output found for function call` entre turno 1 y 2 con `claude-haiku-4-5` → [[claude-vs-openai-en-foundry]].
- Push rechazado por historias divergentes → [[fix-git-push-divergente]].

Ver también: [[00-course-setup]], [[microsoft-agent-framework]]
