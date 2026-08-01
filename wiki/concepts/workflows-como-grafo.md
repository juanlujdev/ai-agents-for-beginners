---
type: concept
date_updated: 2026-08-01
source_count: 4
---

# Workflows como grafo

Orquestar agentes como un **diagrama de flujo**: los nodos son *executors* (agentes o código Python normal) y las aristas (*edges*) definen cómo fluyen los mensajes. No es una cadena de llamadas: es un grafo que se compila con `WorkflowBuilder` y luego se ejecuta.

## Tipos de edge

| Edge | Forma | Dónde se ve |
| --- | --- | --- |
| **Direct** | A → B siempre | `add_edge(a, b)` |
| **Conditional** | A → B solo si se cumple una condición | `add_edge(a, b, condition=fn)` |
| **Switch-case** | A → (B, C o D) según el caso | `add_multi_selection_edge_group(..., selection_func=fn)` |
| **Fan-out** | A → [B, C, D] en paralelo | `add_fan_out_edges(a, agentes)` |
| **Fan-in** | [B, C, D] → A que los combina | — |

## Fan-out: los agentes NO se comunican

El malentendido más común. Con `add_fan_out_edges`, cada agente recibe el input **original** y responde aislado, sin ver la respuesta de los demás. No hay puesta en común ni agente agregador: el workflow solo recolecta N salidas separadas.

Si quieres que uno use lo que produjo otro, necesitas un patrón **secuencial o fan-in**. El propio ejemplo del curso tiene ese bug conceptual: las instrucciones del `plan_agent` dicen *"based on the researcher's findings"*, pero con fan-out nunca los recibe → [[08-multi-agent]].

## El "cerebro" de la decisión es Python, no el LLM

En un workflow condicional, la ruta la decide **código normal** que inspecciona la salida estructurada del agente anterior:

```python
def has_availability_condition(message) -> bool:
    result = BookingCheckResult.model_validate_json(message.agent_response.text)
    return result.has_availability      # solo devuelve un booleano
```

La función de condición **no llama a nadie**. El motor del workflow evalúa cada arista saliente e invoca los destinos cuya condición dio `True`. Esto hace que [[structured-outputs]] sea un prerrequisito: sin salida validada no hay campo que inspeccionar.

## Ejecución concurrente: el orden no es el declarado

`get_outputs()` devuelve los resultados **en orden de llegada**, no en el de `output_executors`. Emparejar por posición es un bug latente; hay que emparejar por `author_name` → [[fix-workflow-concurrente]].

## Traspaso de contexto entre nodos

`AgentExecutor` reenvía por defecto (`context_mode="full"`) toda la conversación previa al nodo siguiente, incluida la mecánica interna de las tool calls. Eso rompe con modelos de razonamiento; se filtra con `context_mode="custom"` y un `context_filter` → [[fix-reasoning-item-workflow]].

## Patrones de orquestación

Secuencial · concurrente · group chat · **handoff** (un agente deriva a un especialista, tipo call center; `HandoffBuilder` genera automáticamente las tools `handoff_to_*`) · **magnetic** (un manager crea la lista de tareas y coordina subagentes).

Human-in-the-loop encaja aquí: un `RequestInfoEvent` **pausa** el workflow esperando respuesta humana, que se envía con `send_responses_streaming(...)`.

Fuentes: [[08-multi-agent]], [[14-microsoft-agent-framework]], [[10-ai-agents-production]]
