---
type: synthesis
date_updated: 2026-08-01
source_count: 2
---

# Fix: `function_call` sin su `reasoning` item entre agentes

## Síntoma

```
Error code: 400 - {'error': {'message':
"Item 'fc_00bb...' of type 'function_call' was provided without its
required 'reasoning' item: 'rs_00bb...'.", 'type': 'invalid_request_error'}}
```

Aparece al pasar de un agente a otro en un workflow, cuando el primero ha llamado a una tool. Se encontró primero en [[10-ai-agents-production]] (workflow OCR → Email) y se resolvió después en [[14-microsoft-agent-framework]] (`14-conditional-workflow.ipynb`).

## Causa raíz — son dos capas

Los modelos de razonamiento (familia o1/o3/gpt-5, aquí `gpt-5-mini`) emiten un item `reasoning` junto a cada `function_call`. La Responses API **exige el par completo** al reenviar el historial. Se pierde en dos sitios distintos:

**Capa 1 — el cliente no guarda estado en el servidor.**

```python
# agent_framework/_clients.py:276
STORES_BY_DEFAULT = False
```

`FoundryChatClient` no usa almacenamiento del lado del servidor por defecto. Sin `store=True`, la librería **descarta el ítem `reasoning`** al reenviar el historial en la segunda vuelta (tras la tool call), dejando el `function_call` huérfano.

**Capa 2 — `AgentExecutor` reenvía la mecánica interna al agente siguiente.**

Con `context_mode="full"` (el valor por defecto), el executor pasa **toda** la conversación previa —incluidos el `function_call` y el `reasoning` internos— al agente siguiente. Pero ese agente tiene su **propia sesión nueva**, sin continuidad de servidor con esa respuesta concreta, así que la API vuelve a rechazar el par. Por eso el error persistía **idéntico** aunque `store=True` ya estuviera puesto.

## El fix completo

```python
# Capa 1: que el servidor retenga el reasoning
agent = provider.as_agent(..., default_options={"response_format": Modelo, "store": True})

# Capa 2: no propagar la mecánica interna de la tool al agente siguiente
def strip_tool_internals(messages):
    # filtra los content types: function_call, function_result, text_reasoning
    ...

alternative_agent = AgentExecutor(..., context_mode="custom", context_filter=strip_tool_internals)
booking_agent     = AgentExecutor(..., context_mode="custom", context_filter=strip_tool_internals)
```

Así entre agentes solo viaja el texto relevante, no la mecánica interna de la llamada.

## Verificación

Reproducido el notebook completo fuera de Jupyter (mismos agentes, mismo grafo) contra Foundry real, con los dos caminos del condicional:

- **Paris** → `alternative_agent` → `{"alternative_destination":"Versailles, France", ...}` ✅
- **Stockholm** → `booking_agent` → `{"destination":"Stockholm","action":"book_now", ...}` ✅

Sin errores de `reasoning`/`function_call` en ninguna rama.

## El callejón sin salida: cambiar a Claude

Un intento intermedio fue cambiar el deployment a `claude-haiku-4-5` para esquivar los reasoning items. Eso sustituye un error por otro: Claude ignora `response_format` y responde texto libre, así que Pydantic falla con `Invalid JSON` → [[claude-vs-openai-en-foundry]]. La salida correcta es volver a `gpt-5-mini` y arreglar la causa real.

**Reiniciar el kernel** tras aplicar los cambios: las nuevas `default_options` / `context_mode` requieren recrear los agentes.

## Nota sobre errores transitorios

Durante las pruebas apareció un `Could not fetch workspace resource` un par de veces: es un fallo transitorio del lado de Azure, desaparece al reintentar. No confundirlo con un error de configuración.

## Regla derivada

Un modelo de razonamiento no es un reemplazo transparente en un pipeline multi-agente. Si un workflow empieza a dar 400 al saltar de un agente a otro, revisar en este orden: `store=True`, y luego qué está reenviando `context_mode`.

Fuentes: [[14-microsoft-agent-framework]], [[10-ai-agents-production]]
