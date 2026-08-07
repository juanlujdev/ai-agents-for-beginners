---
type: concept
date_updated: 2026-08-07
source_count: 7
---

# Tool calling

El LLM **no ejecuta código**. Decide *cuándo* necesita una función y con qué argumentos; el framework la ejecuta y le devuelve el resultado.

```python
@tool(approval_mode="never_require")
def check_destination_availability(
    destination: Annotated[str, "The destination to check availability for"]
) -> str:
    """Check if a vacation destination is currently available for booking."""
    ...
```

## Lo que el modelo realmente lee

Tres cosas, y ninguna es el cuerpo de la función:

1. **El nombre** de la función.
2. **El docstring** — es la descripción de qué hace la tool, expuesta al modelo.
3. **El string dentro de `Annotated`** — **no es un comentario**: es la descripción del parámetro. El framework genera con ella el JSON schema que ve el modelo.

De ahí que la calidad de docstring y anotaciones determine si el modelo llama bien a la tool. Cuando una tool "falla", la primera sospecha es su descripción, no su código → [[10-ai-agents-production]].

## Detalles prácticos

- **Devolver texto en lenguaje natural, no JSON**: quien lee el resultado es el LLM.
- `approval_mode="never_require"` ejecuta sin confirmación humana. Apropiado en operaciones de solo lectura; lo contrario es el gancho de human-in-the-loop.
- **Mínimo privilegio**: dar a cada agente solo las tools que necesita evita que se salga del guion → [[10-ai-agents-production]].
- **Menos de 30 tools a la vez.** Con demasiadas, el modelo llama a la incorrecta — es el *Context Confusion* de [[context-engineering]]. La solución es RAG sobre las descripciones de tools (*dynamic tool selection*).

## Middleware: interceptar la llamada

```python
async def priority_check_middleware(context: FunctionInvocationContext, next) -> None:
    await next(context)                  # ejecuta la función original
    context.result = modificar(context.result)   # inspecciona o altera el resultado
```

Middleware = función `async` que recibe `(context, next)` y **decide cuándo, o si, llamar a `next`**. No llamarlo corta la ejecución. Permite añadir logging, permisos o rate-limiting sin tocar la lógica del agente → [[14-microsoft-agent-framework]].

## Fallback como metacognición

Dar al agente una tool primaria y una de respaldo, e instruirle para que detecte el fallo, cambie de estrategia y **sea transparente** sobre el cambio, es [[metacognicion]] aplicada → [[09-metacognition]].

## Fiabilidad del modelo, no solo del framework

Todo lo anterior asume que el modelo *sabe* producir una tool call bien formada. No es gratis: muchos [[slm|SLMs]] conversan bien pero generan tool calls malformadas o inconsistentes. [[qwen]] es la elección explícita de [[17-creating-local-ai-agents]] precisamente por estar entrenado para function calling fiable — sin eso, un modelo local es un chat, no un agente.

## La mecánica del bucle, sin envolver en ningún framework

[[17-creating-local-ai-agents]] implementa el bucle a mano con el SDK de OpenAI puro (sin [[microsoft-agent-framework]] de por medio), lo que deja ver la mecánica que los frameworks suelen esconder:

```python
messages.append({"role": "assistant", "content": msg.content,
                  "tool_calls": [tc.model_dump() for tc in msg.tool_calls]})
for tc in msg.tool_calls:
    result = TOOL_IMPL[tc.function.name](**json.loads(tc.function.arguments))
    messages.append({"role": "tool", "tool_call_id": tc.id, "content": str(result)})
```

Tres piezas que no son evidentes hasta que se ven explícitas: (1) el historial completo se reenvía en cada vuelta del bucle — el modelo no tiene memoria propia entre llamadas, la lleva el código; (2) `tool_call_id` enlaza cada resultado con la petición exacta que lo originó, porque el modelo puede pedir varias tools en una sola respuesta; (3) hace falta un tope de iteraciones (`max_iterations`) como red de seguridad contra un bucle que nunca converge a una respuesta final.

**Contra la alucinación**: instruir explícitamente *"prefer calling a tool over guessing"* en el system prompt no es cosmético — sin esa instrucción nada impide que el modelo se invente el contenido de un archivo en vez de pedir la tool que lo leería de verdad.

Fuentes: [[02-explore-agentic-frameworks]], [[14-microsoft-agent-framework]], [[09-metacognition]], [[10-ai-agents-production]], [[17-creating-local-ai-agents]]
