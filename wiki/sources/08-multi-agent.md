---
type: source-summary
date_updated: 2026-08-01
leccion: 08-multi-agent
sesiones: [f470acdc]
fechas_origen: 2026-07-17 .. 2026-07-18
---

# 08 — Multi-agent (workflows)

Dos notebooks en `code_samples/workflows-agent-framework/python/`. Introducen los [[workflows-como-grafo]] del [[microsoft-agent-framework]]: agentes conectados como nodos de un grafo, no como una cadena de llamadas.

## Notebook 03 — fan-out concurrente

```python
class InputDispatcher(Executor):
    @handler
    async def forward(self, text: str, ctx: WorkflowContext[str]) -> None:
        await ctx.send_message(text)

workflow = (
    WorkflowBuilder(start_executor=dispatcher, output_executors=agents)
    .add_fan_out_edges(dispatcher, agents)
    .build()
)
```

- `InputDispatcher` es un executor *passthrough*: no transforma nada, solo reenvía el input. Su papel es ser el punto de entrada conectable a varios destinos.
- `add_fan_out_edges` reparte el **mismo mensaje** a todos los agentes **a la vez**.
- `output_executors` está deprecado a favor de `output_from` (sale un `DeprecationWarning`).

**Malentendido corregido:** en fan-out los agentes **no se comunican entre sí**. Cada uno recibe el input original y responde aislado, sin ver la respuesta del otro. No hay agente agregador ni puesta en común: el workflow solo recolecta las N salidas por separado.

**Bug conceptual del propio ejemplo:** las instrucciones de `plan_agent` dicen *"based on the researcher's findings"*, pero con fan-out puro nunca recibe esos hallazgos. Para eso haría falta un patrón secuencial o fan-in, no `add_fan_out_edges`.

## Notebook 04 — workflow condicional (branching)

Escenario: `evangelist_agent` escribe un borrador → `reviewer_agent` lo evalúa (`"Yes"`/`"No"`) → si pasa, `publisher_agent` lo guarda como `.md`.

```
evangelist_agent → reviewer_agent → to_reviewer_result (parsea JSON → ReviewResult)
   → [DECISIÓN] select_targets
      "Yes" → save_draft → publisher_agent
      "No"  → handle_review → yield_output(...) → FIN
```

Piezas clave:

- `@executor(id=...)` convierte una función en nodo del grafo — equivalente en decorador a heredar de `Executor`.
- `to_reviewer_result` es el **traductor**: texto JSON crudo del LLM → objeto Python validado (`model_validate_json`) → dataclass que viaja entre nodos.
- `select_targets` es la decisión, y es **Python puro, no un LLM**. Recibe `target_ids` en el mismo orden que la lista pasada al builder.
- `add_multi_selection_edge_group(origen, [destinos], selection_func=...)` es lo que hace condicional al workflow: la ruta depende del *contenido* de la respuesta, no de la topología fija.
- `draft_content` es el **payload que atraviesa todo**: nace en el evangelist, el reviewer lo copia intacto junto a su veredicto, y termina en el publisher. Si la revisión falla, se pierde (no hay reintento automático pese a lo que pide el prompt).

**Código muerto detectado en el ejemplo:** el `else` de `handle_review` nunca se ejecuta (solo se llega ahí con `"No"`); el comentario `# Order: [handle_review, submit_to_email_assistant, ...]` es un resto copiado de un ejemplo de triaje de emails; y `class DatabaseEvent(WorkflowEvent)` no lo emite nadie, así que su `isinstance` nunca se dispara.

## Estado del código

El notebook 04 estaba escrito contra una versión anterior del SDK y **no ejecutaba**. Nueve incompatibilidades encadenadas hasta hacerlo funcionar → [[fix-notebook-04-sdk-desactualizado]].

Ver también: [[03-agentic-design-patterns]], [[14-microsoft-agent-framework]]
