---
type: source-summary
date_updated: 2026-08-02
leccion: 14-microsoft-agent-framework
sesiones: [5f237daa, 771cdf26, e8c216f7, 1a4c626f, dd752e87, ea1ac533, 62cab66d, 7d9a8e0d, 78c7fe19]
fechas_origen: 2026-07-28 .. 2026-08-02
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

## `14-sequential.ipynb` — producir y luego juzgar

El más simple de los cinco y el mejor para entender qué es un agente. Dos agentes en fila: `front-desk-agent` recomienda **una** atracción de una ciudad, `concierge-agent` la revisa y la puntúa.

```python
workflow = (WorkflowBuilder(start_executor=front_desk_agent,
                            output_executors=[front_desk_agent, concierge_agent])
    .add_edge(front_desk_agent, concierge_agent)
    .build())

events  = await workflow.run(f"I want to visit an attraction in {city}")
outputs = events.get_outputs()       # [respuesta_recepción, respuesta_conserje]
```

Las cuatro decisiones del `WorkflowBuilder`, que es lo que conviene memorizar: `start_executor` (quién recibe la entrada humana) · `add_edge` (quién pasa el trabajo a quién) · `output_executors` (qué salidas se conservan) · `.build()` (cerrar y validar el plano).

**Los dos agentes comparten el mismo `provider`, el mismo modelo y la misma factura.** Lo único que los diferencia es el `instructions`. Es el ejemplo más limpio de *agente = modelo + instrucciones*: el prompt no es decoración, es la personalidad y el criterio del trabajador.

Tres frases de los prompts hacen todo el trabajo:

- **`"provide a single, well-researched recommendation"`** — sin *single*, el modelo devuelve cinco opciones por instinto, y `AttractionRecommendation` solo tiene sitio para un `attraction_name`. **Prompt y esquema tienen que estar de acuerdo** → [[structured-outputs]].
- **`"You will receive an attraction recommendation"`** — la bisagra del patrón secuencial: avisa al conserje de que su entrada viene de **otro agente**, no de un humano. Sin ella recibe un bloque de JSON y se extraña.
- Al conserje **nunca** se le pide ser amable con la propuesta de recepción. Hereda el dato pero no el orgullo → [[llm-as-judge]].

Ninguno tiene tools ni memoria. Consecuencia: cuando el conserje devuelve `visitor_rating: 4.6` **no ha consultado nada** — es una estimación de su entrenamiento con aspecto de dato de TripAdvisor. El ejemplo enseña orquestación, no obtención de datos verídicos.

### El bug: tercera aparición del mismo fallo

Ejecutado contra Foundry real (2026-08-02), falla con:

```
4 validation errors for AttractionRecommendation
attraction_name  Field required   input_value={'name': 'Vasa Museum (Va...
description / category / why_recommended   Field required
```

El modelo **acertó la atracción** (Museo Vasa) y **se inventó los rótulos** de las casillas: `name` en vez de `attraction_name`. Causa: las instrucciones solo *pedían* `"Return structured JSON matching the AttractionRecommendation schema"`. El modelo nunca ve la clase Python — solo lee ese nombre como texto y adivina los campos.

Mismo Bug 2 que [[fix-workflow-concurrente]]. Fix aplicado:

```python
front_desk_agent = provider.as_agent(
    name="front-desk-agent",
    instructions="...",                                              # sin la petición de JSON, ya redundante
    default_options={"response_format": AttractionRecommendation},   # restricción dura
)
```

**Verificado end-to-end** contra Foundry real (2026-08-02), notebook completo sin una sola salida de tipo `error`:

```
celda 10 (Stockholm):  Front Desk -> Vasa Museum (Vasamuseet) | category "Maritime museum / History"
                       Concierge  -> popularity 9/10, visitor_rating 4.5/5.0
celda 12 (Barcelona):  Step 1 user -> Step 2 front-desk (JSON) -> Step 3 concierge (JSON)
                       Total Steps: 3
```

Dos detalles que confirman el diagnóstico: los campos ahora llegan con **el nombre exacto del esquema** (`"attraction_name":"Sagrada Família..."`, antes `name`), y la salida del recepcionista que se ve en el Step 2 es **JSON estricto sin envolver** — la restricción dura funcionando, no una sugerencia obedecida por casualidad.

Al re-ejecutar hay que acordarse de la trampa del kernel cacheado: la celda de los agentes **y** la del `workflow`, porque el flujo guarda referencias a los agentes viejos.

### Detalles honestos del notebook

- **`output_executors` está deprecado** en la versión instalada: `DeprecationWarning: use output_from instead`. Funciona, solo avisa.
- **Las salidas se leen por posición** (`outputs[0]`, `outputs[1]`). Es la fragilidad de [[fix-workflow-concurrente]] Bug 1, pero **dormida**: al ser secuencial, el orden de llegada coincide con el declarado. Rompería al convertir el flujo en concurrente.
- La numeración de los pasos salta del 4 al 8 — restos de una versión más larga recortada sin renumerar.
- `analyze_sequential_flow()` **no analiza** la ejecución anterior: **re-ejecuta el workflow completo** con otra ciudad (Barcelona) y reconstruye a mano una narración de tres pasos. Cuesta dos llamadas más. Un análisis real usaría el contenido de `events`, que ya trae la crónica interna sin volver a pagar.
- El resumen final ("Agents Involved: 2", "Flow Pattern: Linear sequential") está **escrito a mano en el HTML**; solo `len(steps)` se calcula. Miente en cuanto se añada un tercer agente.
- `# 1-10 scale` junto a `popularity_score: int` es un comentario, no una validación. Un 47 pasa el `model_validate_json` y rompe la barra de emojis (`"🟩" * 47`). Se arreglaría con `Field(ge=1, le=10)`.
- Imports huérfanos: `asyncio`, `json`, `Any`, `cast`, `Message` no se usan.

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
