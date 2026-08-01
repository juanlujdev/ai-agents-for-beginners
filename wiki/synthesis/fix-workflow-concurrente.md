---
type: synthesis
date_updated: 2026-08-01
source_count: 1
---

# Fix: los dos bugs del workflow concurrente

`14-concurrent.ipynb` fallaba con errores de validación Pydantic que parecían uno solo. Eran **dos bugs independientes**, y el primer arreglo tapaba parcialmente al segundo.

## Bug 1 — `get_outputs()` no respeta el orden declarado

**Síntoma**: `7 validation errors for AttractionsRecommendation` — el contenido de historia se validaba contra el esquema de atracciones.

**Causa raíz**, leída en el fuente del paquete instalado:

- `WorkflowRunResult.get_outputs()` (`_workflow.py:127`) hace `[event.data for event in self if event.type == "output"]` — devuelve los eventos **en orden de llegada**, no en el orden de `output_executors`.
- Cada `AgentExecutor` (`_agent_executor.py:455`) hace `ctx.yield_output(response)` de forma independiente, en cuanto ese agente termina, **sin sincronización** entre ellos.

Con tres agentes en paralelo, el más rápido cae en `outputs[0]`. El notebook asumía `outputs[i] ↔ sections[i]` por posición, y ese supuesto es **falso en modo concurrente**.

**Fix** — emparejar por nombre, no por posición:

```python
outputs_by_agent = {out.messages[0].author_name: out for out in events.get_outputs()}
```

`author_name` lo rellena el framework con el `name` del agente (`_agents.py:1065`), así que el orden real de finalización deja de importar.

## Bug 2 — pedir JSON por prompt no es structured output

**Síntoma tras el fix anterior**: seguían los errores, ahora con campos extra (`summary`, `historical_context`), campos anidados donde se esperaba texto plano (`activities` como lista de objetos `{category, ...}` en vez de lista de strings) y campos obligatorios ausentes.

**Causa**: las instrucciones de cada agente solo **pedían por texto** *"Return structured JSON matching the X schema"*. Eso es una sugerencia en el prompt, no una restricción. El LLM la interpretaba libremente.

**Fix** — activar structured outputs de verdad:

```python
attractions_agent = provider.as_agent(
    name="attractions-agent",
    instructions="...",                                    # sin la petición redundante de JSON
    default_options={"response_format": AttractionsRecommendation},
)
```

`default_options={"response_format": ModeloPydantic}` convierte el modelo en JSON Schema y lo manda como **restricción dura** (`text_format` / `json_schema` en la request), no como sugerencia. Ver [[structured-outputs]].

## Verificación

Se probó end-to-end contra Foundry real, con los tres agentes en el workflow concurrente completo:

```
Orden real de finalización (author_name): ['history-agent', 'attractions-agent', 'dining-agent']
OK: attractions-agent valida contra AttractionsRecommendation
OK: dining-agent valida contra DiningRecommendation
OK: history-agent valida contra HistoryRecommendation
```

El log confirma de paso que el orden de finalización **no** coincidió con el declarado, justificando por qué el fix 1 era necesario además del 2.

## Trampa de método: el kernel cacheado

Tras aplicar el primer fix el error seguía apareciendo, porque el kernel de Jupyter tenía cargada en memoria la versión vieja de la función. **Jupyter no recarga funciones automáticamente**: hay que re-ejecutar la celda que las define, o reiniciar el kernel. Si un fix "no funciona" y el archivo sí está corregido, esta es la primera hipótesis.

Relacionado: Claude Code lee el `.ipynb` guardado en disco, no el estado vivo del kernel. Si VS Code no ha guardado tras la ejecución, la salida nueva no se ve desde fuera.

Fuentes: [[14-microsoft-agent-framework]]
