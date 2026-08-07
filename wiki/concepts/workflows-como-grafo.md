---
type: concept
date_updated: 2026-08-07
source_count: 5
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

## Las cuatro decisiones de `WorkflowBuilder`

Todo grafo se declara respondiendo a cuatro preguntas, y conviene verlas separadas porque se confunden:

| Pieza | Pregunta | Nota |
| --- | --- | --- |
| `start_executor` | ¿Quién recibe la entrada humana? | Un nodo puede ser esto **y además** salida; son roles distintos |
| `add_edge(a, b)` | ¿Quién pasa el trabajo a quién? | Lo que viaja es el **texto** de `a`, no un objeto |
| `output_executors` | ¿Qué salidas se conservan? | **Deprecado**: usar `output_from`. Sin él solo verías el último nodo |
| `.build()` | ¿Está el plano cerrado? | Además **valida** el grafo: nodos desconectados, start inexistente |

## Secuencial: producir y luego juzgar

La topología mínima (`A → B`) y la que más valor conceptual da. Su interés no es técnico sino de diseño: **separar quién produce de quién juzga elimina el conflicto de intereses**. Un solo agente al que se le pide proponer *y* criticar acaba defendiendo su propia propuesta; el segundo agente hereda el dato pero no el orgullo → [[llm-as-judge]].

Lo que hace posible el traspaso es una frase en el prompt del nodo receptor: *"You will receive an attraction recommendation"*. Sin ella el agente recibe un bloque de JSON sin saber que es su material de trabajo. **El contexto del traspaso viaja además dentro del propio esquema** — repetir `city` y `attraction_name` en los dos modelos Pydantic no es redundancia, es lo que permite que la evaluación se explique sola → [[14-microsoft-agent-framework]].

Motivo para no hacerlo a mano: con dos nodos, `add_edge` solo ahorra tres líneas de fontanería. La ganancia aparece con condiciones, reintentos y trazas.

## Ejecución concurrente: el orden no es el declarado

`get_outputs()` devuelve los resultados **en orden de llegada**, no en el de `output_executors`. Emparejar por posición es un bug latente; hay que emparejar por `author_name` → [[fix-workflow-concurrente]].

**En secuencial la fragilidad está dormida**, no ausente: el orden de llegada coincide con el declarado porque `B` no puede terminar antes que `A`. `14-sequential.ipynb` lee `outputs[0]`/`outputs[1]` por posición y funciona. Rompería en silencio —datos cruzados, ningún error— al convertir ese flujo en concurrente o al reordenar la lista de salidas.

## Traspaso de contexto entre nodos

`AgentExecutor` reenvía por defecto (`context_mode="full"`) toda la conversación previa al nodo siguiente, incluida la mecánica interna de las tool calls. Eso rompe con modelos de razonamiento; se filtra con `context_mode="custom"` y un `context_filter` → [[fix-reasoning-item-workflow]].

## Patrones de orquestación

Secuencial · concurrente · group chat · **handoff** (un agente deriva a un especialista, tipo call center; `HandoffBuilder` genera automáticamente las tools `handoff_to_*`) · **magnetic** (un manager crea la lista de tareas y coordina subagentes).

Human-in-the-loop encaja aquí: un `RequestInfoEvent` **pausa** el workflow esperando respuesta humana, que se envía con `send_responses_streaming(...)`.

## El mismo primitivo, aplicado a permiso en vez de a datos

En [[16-deploying-scalable-agents]], el patrón de despliegue **Agent Workflow** ([[patrones-de-despliegue]]) usa esta misma pausa para un `Human Approval Node`: el grafo no espera un dato que falta, espera que un humano apruebe o rechace una acción de negocio (reembolso, borrado de cuenta) antes de que el nodo siguiente se ejecute. Mismo mecanismo (`RequestInfoEvent`), distinto motivo para pausar.

Fuentes: [[08-multi-agent]], [[14-microsoft-agent-framework]], [[10-ai-agents-production]], [[16-deploying-scalable-agents]]
