---
type: source-summary
date_updated: 2026-08-01
leccion: 14-microsoft-agent-framework
sesiones: [5f237daa, 771cdf26, e8c216f7, 1a4c626f, dd752e87, ea1ac533, 62cab66d, 7d9a8e0d]
fechas_origen: 2026-07-28 .. 2026-07-31
---

# 14 — Microsoft Agent Framework

La lección más trabajada del historial: ocho sesiones y cinco notebooks. Ver la entidad en [[microsoft-agent-framework]].

MAF es el framework paraguas que fusiona **Semantic Kernel** y **AutoGen** en una API única. Resuelve tres preguntas: cómo hablo con el LLM, cómo le doy herramientas y memoria, y cómo coordino varios agentes en producción con seguridad y trazabilidad.

## Conceptos del README

**Crear agente** — se separa el *cliente* (quién provee el LLM) del *agente* (cómo se comporta), así se cambia de proveedor sin tocar la lógica. Cambiar el motor sin rediseñar la carrocería.

```python
agent = AzureOpenAIChatClient(credential=AzureCliCredential()).create_agent(
    instructions="...", name="TripRecommender")
```

**Tools** — función Python normal; `Annotated[..., Field(description=...)]` genera el JSON schema que lee el modelo. El LLM no ejecuta código: decide *cuándo* llamar y con qué argumentos. Se pueden pasar también por ejecución puntual: `agent.run("...", tools=[get_attractions])`.

**Threads** — `agent.get_new_thread()` guarda el historial. Sin thread, cada `.run()` es una conversación nueva. Se serializan (`await thread.serialize()`) para persistir entre sesiones.

**Memoria en tres sabores**: in-memory (durante el thread) · persistent messages (`ChatMessageStore`) · dynamic memory (Mem0 vía `context_providers`, que inyecta memoria relevante *antes* de responder) → [[13-agent-memory]].

**Orquestación**: secuencial · concurrente · group chat · handoff · magnetic (un manager crea la lista de tareas y coordina subagentes).

**Tipos de edge en workflows**: direct · conditional · switch-case · fan-out · fan-in → [[workflows-como-grafo]].

## `14-concurrent.ipynb` — fan-out con tres especialistas

Tres agentes (`attractions`, `dining`, `history`) reciben el mismo destino en paralelo y devuelven JSON validado con Pydantic. Dio **dos bugs encadenados** → [[fix-workflow-concurrente]].

## `14-conditional-workflow.ipynb` — enrutamiento por condición

La bifurcación vive en dos piezas separadas, y esto es lo importante de entender:

```python
def has_availability_condition(message: Any) -> bool:
    result = BookingCheckResult.model_validate_json(message.agent_response.text)
    return result.has_availability          # solo inspecciona, no llama a nada

workflow = (WorkflowBuilder(start_executor=availability_agent, output_executors=[display_result])
    .add_edge(availability_agent, alternative_agent, condition=no_availability_condition)
    .add_edge(availability_agent, booking_agent,     condition=has_availability_condition)
    .build())
```

Las funciones de condición **no llaman a nadie**: devuelven un booleano. El **motor del workflow** evalúa cada arista que sale del agente e invoca el destino de aquellas cuya condición dio `True`. Como las dos son mutuamente excluyentes, solo una se activa.

Este notebook destapó el problema de los reasoning items y su solución → [[fix-reasoning-item-workflow]].

Discrepancia menor: el markdown del notebook menciona `@ai_function`, pero el código usa `@tool`. Es desfase de redacción, mismo comportamiento.

## `14-handoff.ipynb` — derivación tipo call center

Cuatro agentes: `customer_support_agent` (triaje) que deriva a `booking`, `disputes` o `trip_check`. `HandoffBuilder` genera automáticamente las tools `handoff_to_<agente>` que el coordinador menciona en su prompt.

```python
workflow = (HandoffBuilder(name="travel_support_handoff", participants=[...])
    .set_coordinator(customer_support_agent)
    .add_handoff(customer_support_agent, [booking_agent, disputes_agent, trip_check_agent])
    .with_termination_condition(lambda conv: sum(1 for m in conv if m.role.value == "user") > 3)
    .build())
```

**Bug del notebook original**: la condición de terminación venía como `... == "user") > ` — sin ningún valor tras el `>`. Es un **`SyntaxError`**: el notebook no pasa de esa celda. El comentario dice *"Stop after 3 user messages"*, así que falta el `3`.

El patrón human-in-the-loop aparece aquí: `RequestInfoEvent` con un `HandoffUserInputRequest` significa que el workflow **se pausa** esperando respuesta humana; se continúa con `workflow.send_responses_streaming({req.request_id: respuesta})`.

## `14-middleware.ipynb` — interceptar sin tocar el flujo

Evolución del conditional-workflow: mismo workflow, mismo tool, misma lógica de enrutamiento, **comportamiento distinto según quién llama**.

```python
async def priority_check_middleware(context: FunctionInvocationContext, next) -> None:
    await next(context)                     # ejecuta la función original
    if context.result and context.function.name == "hotel_booking":
        result_data = json.loads(context.result)
        if current_user_id in PRIORITY_MEMBERS and not result_data["has_availability"]:
            result_data["has_availability"] = True
            result_data["priority_override"] = True
            context.result = json.dumps(result_data)
```

El patrón a memorizar: middleware = función `async` que recibe `(context, next)` y **decide cuándo (o si) llamar a `next(context)`**. No llamarlo corta la ejecución. Mismo principio que Express, Django o FastAPI.

Dos detalles no obvios:

- **`priority_override` existe como campo del modelo** porque en structured outputs de Azure/OpenAI **todos los campos deben ir en el array `required`** del JSON schema — no se admiten opcionales con valor por defecto. Por eso no se escribió `priority_override: bool = False`.
- La variable global `current_user_id` es simplificación didáctica; el propio código lo advierte ("in real app, use proper session management").

Que "paris" no esté en la lista de ciudades con habitaciones es deliberado: permite demostrar cómo el middleware revierte el resultado para usuarios prioritarios.

## `14-langchain-hosted-agent.py` — LangGraph dentro de Foundry

No es un notebook, es un script. Expone un agente de LangChain/LangGraph como *hosted agent* de Foundry: tú programas la lógica con LangGraph, Foundry gestiona runtime, sesiones, escalado, identidad y endpoints.

```python
def build_chat_model() -> ChatOpenAI:
    project = AIProjectClient(endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"].rstrip("/"),
                              credential=DefaultAzureCredential())
    return ChatOpenAI(model=deployment,
                      base_url=str(project.get_openai_client().base_url),
                      api_key=get_bearer_token_provider(credential, "https://ai.azure.com/.default"))

graph = create_agent(build_chat_model(), tools=[])
ResponsesHostServer(graph).run(port=int(os.environ.get("PORT", "8088")))
```

Detalle elegante: `api_key` no recibe una clave sino un **`token_provider`**, una función que genera tokens frescos bajo demanda. Los tokens de Azure caducan cada hora y así se renuevan solos. Como pedirle una contraseña temporal nueva al guardia cada vez, en vez de tener una fija.

Dos protocolos de exposición: **Responses** (`/responses`, recomendado — compatible con OpenAI, con streaming e historial) e **Invocations** (`/invocations`, para JSON custom o webhooks).

Ver también: [[08-multi-agent]], [[10-ai-agents-production]], [[13-agent-memory]]
