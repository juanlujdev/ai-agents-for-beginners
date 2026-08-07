---
type: source-summary
date_updated: 2026-08-07
leccion: 16-deploying-scalable-agents
fuente: [16-deploying-scalable-agents/README.md, 16-deploying-scalable-agents/code_samples/16-python-agent-framework.ipynb]
---

# 16 — Deploying Scalable Agents (Microsoft Foundry)

Lección bisagra: hasta la 15 el agente corre en un notebook con `az login`; aquí se explica todo lo que hay que añadir para que ese mismo bucle (razonar → llamar tools → responder) sobreviva a producción real. Ingerida primero desde el README (diseño documentado); el notebook (`16-python-agent-framework.ipynb`) se recorrió después celda a celda en una segunda sesión y se ejecutó contra Foundry real, incluida la compuerta de evaluación — ver [[fix-azure-search-placeholder-url]] para el bug real que salió de esa ejecución.

## La idea que ordena la lección: el modelo es ~20% del sistema

Prototipo y producción comparten el bucle central; lo que cambia es el 80% que lo rodea:

| Aspecto | Prototipo | Producción |
| --- | --- | --- |
| Hosting | notebook | servicio versionado, con rollout |
| Identidad | `az login` personal | identidad gestionada, RBAC con mínimo privilegio |
| Estado | en memoria del proceso, se pierde | externalizado (thread store / memory service) |
| Fallos | se ve el traceback | reintentos, fallback, dead-letter, alertas |
| Coste | "son centavos" | medido por request, enrutado, cacheado, presupuestado |
| Calidad | se revisa a ojo | evaluación automática antes de cada release |
| Confianza | se aprueba cada acción a mano | política + humano en el loop |

## Patrones de despliegue

Tres formas de alojar el bucle del agente, con trade-off de control vs. superficie operativa. Detalle y diagrama en [[patrones-de-despliegue]]:

1. **Client-hosted** — el bucle corre en tu propio proceso (lo que hace el curso hasta la lección 15).
2. **Hosted Agents (Foundry Agent Service)** — el agente se registra como recurso en [[azure-ai-foundry]]; Foundry aloja el loop, guarda threads, aplica RBAC y seguridad de contenido.
3. **Agent Workflows** — varios agentes compuestos en grafo con nodos de aprobación humana y checkpoints durables; es [[workflows-como-grafo]] aplicado a escala de despliegue.

## Ciclo de vida: la evaluación es una compuerta, no un paso posterior

```
Crear → Versionar → Evaluar (offline) → [pasa el umbral] → Desplegar → Observar (online) → recoger fallos → Crear
                  ↳ [no pasa el umbral] → vuelve a Crear
```

Esto es el mismo loop offline/online de [[10-ai-agents-production]] (`evaluar offline → desplegar → monitorear online → recolectar fallos → refinar → repetir`), pero aquí se vuelve explícito: **una versión nueva no se despliega si no supera el umbral**. Se implementa como código, no como buena intención:

```python
async def evaluation_gate(agent, test_cases, threshold: float = 0.8) -> bool:
    passed = 0
    for case in test_cases:
        result = await agent.run(case["input"])
        if score_response(result.text, case["expected"]) >= 0.8:
            passed += 1
    return (passed / len(test_cases)) >= threshold  # solo despliega si pasa el umbral
```

El evaluador de puntuación (`score_response`) es el mismo rol que [[llm-as-judge]] — aquí conectado a una decisión de CI/CD (desplegar o no), no solo a una métrica que se observa.

## Estrategias de escalado

Cuatro técnicas, porque cada request puede disparar varias llamadas caras a modelo/tools (no es como escalar una API REST sin estado):

- **Sin estado en el proceso** — el estado por usuario vive en el thread store de Foundry o un memory service, nunca en memoria del proceso. Es lo que permite escalar horizontalmente sin sticky sessions.
- **Model routing** — un clasificador de complejidad manda las preguntas simples a un modelo pequeño y barato, y reserva el grande para razonamiento real. Foundry tiene un **Model Router** nativo; el notebook construye la versión DIY.
- **Cache de respuestas** — preguntas casi-duplicadas se sirven sin tocar el modelo. La llamada más barata es la que no se hace.
- **Concurrencia acotada** — reintentos con backoff exponencial, fallo elegante (cola) en vez de 500, por los rate limits del proveedor.

```python
async def handle_support_request(query: str, customer_id: str) -> str:
    cached = response_cache.get(normalize(query))
    if cached:
        return cached

    model = "gpt-5-nano" if is_simple(query) else "gpt-5-mini"  # routing por complejidad

    with tracer.start_as_current_span("support_request") as span:
        span.set_attribute("routed.model", model)
        span.set_attribute("customer.id", customer_id)
        response = await support_agent.run(query, model=model)

    response_cache.set(normalize(query), response.text)
    return response.text
```

Esta misma disciplina de coste (SLM para tareas simples, router barato, cache, presupuesto) ya aparecía en [[10-ai-agents-production]] como recomendación general; aquí se convierte en código concreto con dos modelos nombrados (`gpt-5-nano` / `gpt-5-mini`) y una regla explícita de `is_simple()`.

[[17-creating-local-ai-agents]] extiende este mismo routing con un tercer eje: en vez de enrutar solo entre dos modelos de nube, una de las opciones puede ser la propia máquina (un [[slm]] servido por [[foundry-local]]), enrutando por sensibilidad/offline además de por complejidad.

## Observabilidad

Reafirma OpenTelemetry de [[10-ai-agents-production]] (`agent_framework.observability`, `get_tracer()`), con el matiz de producción: los **atributos del span** (`customer.tier`, `routed.model`) son lo que convierte una traza en pregunta de negocio respondible — "¿los clientes enterprise se están enrutando de más al modelo pequeño?" — no solo en un log de auditoría.

## Coste: orden de impacto

1. **Modelo del tamaño correcto** — el más pequeño que aún pasa la compuerta de evaluación; evaluar es lo que *prueba* que basta, no una intuición.
2. **Enrutar por complejidad.**
3. **Cachear agresivamente.**

Coste y calidad son la misma disciplina vista desde dos ángulos: la evaluación fija el **suelo de calidad**, routing+cache acercan el **coste** a ese suelo sin bajarlo.

## Controles de empresa

- **Gobernanza**: cada agente con identidad gestionada de mínimo privilegio (RBAC, auditoría) heredada de [[azure-ai-foundry]].
- **Humano en el loop**: tools con `approval_mode` que pausan la ejecución hasta que un humano aprueba o rechaza (reembolsos por encima de un umbral, borrar cuentas). Es el mismo `RequestInfoEvent` de [[workflows-como-grafo]], aplicado a una acción de negocio real en vez de a un dato que falta. El README cita la lección 06 (sin ingerir todavía) como origen del primitivo.
- **MCP como límite no confiable**: cada servidor [[mcp]] en producción se trata como una dependencia de terceros — versión fijada, identidad acotada, salidas validadas, sin secretos expuestos.

## Smoke tests: una capa distinta de la evaluación offline

La compuerta de evaluación prueba el **objeto agente** en local. Una vez desplegado como Hosted Agent hace falta una comprobación más barata y distinta: **¿el endpoint responde de verdad?** Un despliegue "exitoso" solo prueba que el control plane aceptó la definición, no que el agente conteste — puede faltar una dependencia o tener mal el routing y seguir en verde.

El repo trae un pipeline listo con la GitHub Action `ai-smoketest`, un catálogo de prompts en `tests/lesson-16-smoke-tests.json` y el workflow `.github/workflows/smoke-test.yml`. Pirámide de tres capas: **smoke test** (¿vivo?) en cada deploy → **evaluación offline** (¿suficientemente bueno?) antes de promover → **evaluación online** (¿cómo va en el mundo real?) de forma continua.

## Laboratorio: agente de soporte Contoso

El notebook integra en un solo agente las ocho piezas de la lección: tools con `@tool(approval_mode=...)` (`get_order_status`, `open_ticket`, `issue_refund` con `always_require`), RAG (`search_policies`, con `_azure_search` real y fallback `_in_memory_search` conmutados por `USE_AZURE_SEARCH`), memoria de cliente (`memory_context()` inyectada como prefijo del prompt, no como historial de turnos), model routing (`is_simple()` por palabras de alarma + longitud, con caché de agentes por modelo en `agent_for()`), cache de respuestas (`normalize()` + diccionario), compuerta de evaluación (`evaluation_gate`, umbral 80% global / 50% por pregunta) y trazas OpenTelemetry con fallback `_NoopTracer` (patrón Null Object) si el paquete de observabilidad no está instalado. La tarea (assignment) pide adaptarlo a un agente de facturación SaaS, con `issue_credit` sujeto a aprobación humana por encima de $50 — mismo patrón de umbral que el ejemplo de reembolsos.

**Ejecución real verificada**: la compuerta de evaluación corrió contra Foundry real y dio **75% (< 80%, deploy bloqueado)** por un bug de configuración, no del notebook en sí — ver [[fix-azure-search-placeholder-url]]. `get_order_status` (sin red) puntuó 100%; las tres preguntas de política fallaron porque `search_policies` intentaba Azure AI Search real contra un endpoint de plantilla sin rellenar en `.env`.

Fuente: `16-deploying-scalable-agents/README.md` y `16-deploying-scalable-agents/code_samples/16-python-agent-framework.ipynb`, recorrido celda a celda + ejecución real en sesión de conversación.
