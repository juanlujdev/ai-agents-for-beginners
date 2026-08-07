---
type: log
date_updated: 2026-08-07
---

# Log

Append-only. Formato de cabecera fijo para poder filtrar:
`grep "^## \[" wiki/log.md | tail -5`

## [2026-08-07] ingest | 16-python-agent-framework.ipynb (recorrido celda a celda + fix de RAG)

**Fuente**: `16-deploying-scalable-agents/code_samples/16-python-agent-framework.ipynb`, recorrido celda a celda en sesión didáctica (20 celdas, explicación en castellano de cada bloque de código y cada prompt) y ejecución real contra Foundry. Sin fuente escrita propia previa a esta sesión — el notebook ya estaba citado desde el README pero no se había abierto ni ejecutado (pendiente registrado el mismo día en la ingesta anterior).

Confirma en código real las ocho piezas descritas en el README: `@tool(approval_mode=...)` (`never_require` vs `always_require` en `issue_refund`), `search_policies` con conmutador `USE_AZURE_SEARCH`, `memory_context()` inyectada como prefijo de prompt, `is_simple()` + `agent_for()` con caché de agentes por modelo, `response_cache` con `normalize()`, `evaluation_gate()` (umbral 80% global / 50% por pregunta — dos umbrales distintos, no uno solo) y `_NoopTracer` como patrón Null Object cuando el paquete de observabilidad no está instalado.

**Bug real encontrado y corregido**: la compuerta de evaluación dio 75% (bloqueó el "deploy") porque `search_policies` fallaba con `ServiceResponseError('URL has an invalid label.')`. Causa: `.env` trae `AZURE_SEARCH_SERVICE_ENDPOINT="https://..."` y `AZURE_SEARCH_API_KEY="..."` como placeholders de la lección 05 (sin ingerir); `bool("https://...")` es `True`, así que `USE_AZURE_SEARCH` activa el buscador real de Azure contra un host inválido en vez de caer al fallback en memoria. Fix aplicado: vaciar solo `AZURE_SEARCH_SERVICE_ENDPOINT` en `.env` (el `and` corta en el primer falsy, así que basta una de las dos variables); `AZURE_SEARCH_API_KEY` se dejó con el placeholder, inofensivo mientras el endpoint siga vacío. Detalle completo en [[fix-azure-search-placeholder-url]].

**Páginas creadas** (1): `synthesis/fix-azure-search-placeholder-url.md`
**Páginas actualizadas** (4): `sources/16-deploying-scalable-agents.md` (frontmatter `fuente` + sección Laboratorio con resultado de ejecución real), `overview.md` (tesis 1 extendida al caso `.env`), `index.md` (fila de fuente + fila de síntesis + pendiente resuelto/reemplazado por 2 pendientes más precisos)

## [2026-08-07] ingest | 16-deploying-scalable-agents (README, prototipo → producción)

**Fuente**: `16-deploying-scalable-agents/README.md`, recorrido y explicación didáctica completa en sesión de conversación (tabla prototipo/producción, tres patrones de despliegue, ciclo de vida, escalado, observabilidad, coste, controles de empresa, smoke tests, laboratorio y knowledge check). El notebook (`16-python-agent-framework.ipynb`) y el pipeline de smoke tests **no se ejecutaron** — ingesta solo de lectura sobre el README.

Lección bisagra del curso: cierra el arco que va de "agente en un notebook" a "agente en producción". Introduce un concepto nuevo con página propia — [[patrones-de-despliegue]] (client-hosted / Hosted Agent / Agent Workflow) — y extiende tres piezas ya existentes en vez de duplicarlas:

- **Evaluación como compuerta de release** — el loop offline/online de [[10-ai-agents-production]] se vuelve código explícito (`evaluation_gate()` bloquea el deploy bajo un umbral); nuevo rol para [[llm-as-judge]] (`source_count` 3→4).
- **`RequestInfoEvent` aplicado a permiso, no a datos** — el `Human Approval Node` del patrón Agent Workflow es el mismo primitivo de pausa de [[workflows-como-grafo]] (`source_count` 4→5), ahora deteniendo el grafo para una aprobación humana de negocio (reembolso, borrado de cuenta).
- **Foundry Agent Service como Hosted Agent** — nueva sección en [[azure-ai-foundry]] (`source_count` 7→8): el agente como recurso registrado, no como proceso propio.
- **`WorkflowBuilder` y `agent_framework.observability` a escala de despliegue** — nueva sección en [[microsoft-agent-framework]] (`source_count` 8→9): atributos de span de negocio, `@tool(approval_mode=...)` como gate de aprobación.

**Páginas creadas** (2): `sources/16-deploying-scalable-agents.md`, `concepts/patrones-de-despliegue.md`
**Páginas actualizadas** (6): `entities/azure-ai-foundry.md`, `entities/microsoft-agent-framework.md`, `concepts/workflows-como-grafo.md`, `concepts/llm-as-judge.md`, `index.md` (fuente + concepto + 1 pendiente nuevo), `overview.md` (source_count 11→12, hilo conceptual)

**Pendiente nuevo**: el notebook y el pipeline de smoke tests de la lección 16 no se han ejecutado contra Foundry real — todo lo ingerido es diseño documentado en el README, no comportamiento verificado.

## [2026-08-07] ingest | Fix: 15-browser-user.ipynb en Windows (Chrome, .env, event loop)

**Fuente**: sesión de depuración real en esta conversación, ejecutando `15-browser-user.ipynb` en Windows tras el recorrido didáctico. Sin fuente escrita propia — se ingiere el resumen de la conversación (caso "ingesta sin fuente escrita").

Tres fallos encadenados, cada uno reproducido de forma aislada y con el fix verificado antes de tocar el notebook:
1. `start_chrome_with_cdp` no encontraba Chrome en Windows (lista de rutas Mac/Linux/PATH) → se cambió a `p.chromium.executable_path` de Playwright.
2. `.env` con `AZURE_OPENAI_DEPLOYMENT` pero no `AZURE_OPENAI_CHAT_DEPLOYMENT_NAME`, y sin `AZURE_OPENAI_API_KEY` → añadidas las 3 variables que lee `browser_use.ChatAzureOpenAI`, sin tocar la variable que usan otras 35 referencias del curso.
3. `NotImplementedError` al arrancar Playwright, por `ipykernel` forzando `WindowsSelectorEventLoopPolicy` (incompatible con subprocesos) → `main()` se ejecuta en un hilo aparte con `WindowsProactorEventLoopPolicy` propia, solo en Windows.

**Páginas creadas** (1): `synthesis/fix-browser-use-windows-jupyter.md`
**Páginas actualizadas** (2): `sources/15-browser-use.md` (nueva sección con enlace), `index.md` (síntesis + 2 pendientes nuevos)

## [2026-08-07] ingest | 15-browser-use (README + notebook, computer use agents)

**Fuente**: `15-browser-use/README.md` y `15-browser-use/15-browser-user.ipynb`, recorrido completo en sesión didáctica (explicación celda a celda + arquitectura), con el log de una ejecución real del notebook contra Airbnb.

Primera lección del curso sobre **computer use agents**: el agente actúa sobre una interfaz visual (navegador) en vez de una API. Introduce dos entidades nuevas ([[browser-use]] como framework, Playwright/CDP como capa de control) y el patrón conceptual central de la lección: **Agente vs Actor**, con su combinación híbrida (Agente para navegación abierta, control directo para extracción estructurada).

**Páginas creadas** (3):
- `sources/15-browser-use.md`
- `entities/browser-use.md`
- `concepts/computer-use-agents.md`

**Páginas actualizadas** (2):
- [[structured-outputs]] — nueva sección: el mismo contrato (esquema forzado vs. sugerencia en prompt) aplicado a **visión** en vez de texto, vía `page.extract_content(structured_output=..., ...)`. `source_count` 5→6.
- [[metacognicion]] — nueva referencia de producción: Project Opal planifica y se autosupervisa, mismo principio que la metacognición del curso pero a escala empresarial. `source_count` 2→3.

**Hallazgos nuevos, sin fix aplicado** (quedan en Pendientes):
- Bug real en la ejecución: `take_screenshot()` falla (`a bytes-like object is required, not 'str'`) porque las dos versiones de `AirbnbSearchAgent` en el mismo notebook asumen tipos de retorno distintos para `page.screenshot()` de Browser-Use, según si la conexión es vía `playwright_browser` o `cdp_url`.
- `langchain-openai` se instala pero no se usa (el notebook cambió a `ChatAzureOpenAI` de `browser_use`, con comentario explícito en el código, sin limpiar el `pip install`).

**Conocimiento propagado**: Project Opal (referencia de Microsoft citada en el README) conecta esta lección con conceptos ya vistos — human-in-the-loop, agentes seguros, [[metacognicion|metacognición]], [[tool-calling|tools reutilizables]] — pero de lecciones (04, 06, 07, 18) que siguen sin ingerir. Los enlaces quedan marcados como pendientes en vez de inventar contenido para esas lecciones.

## [2026-08-07] ingest | Fix de imports desactualizados en hotel_booking_workflow_sample.py

`ChatMessage`, `ai_function` y `Role.USER` no existen en el `agent_framework` instalado (verificado con imports reales, no con documentación): son `Message`, `tool` y `role="user"` con `contents=[...]`. Corregido en el archivo. Cuarta aparición del mismo patrón de bug ya visto en [[fix-notebook-04-sdk-desactualizado]] — nueva página [[fix-hotel-booking-sample-imports]]. Corrige además una conclusión previa en [[14-microsoft-agent-framework]] que había calificado la mención a `ai_function` como "desfase de redacción" sin bug real: en este script sí lo es.

## [2026-08-01] setup | Esqueleto de la wiki creado

Estructura inicial: `index.md`, `overview.md`, `sources/`, `entities/`, `concepts/`, `synthesis/`, `raw/`.
Fuentes definidas: lecciones del curso (`01-…/18-…`, README + notebooks) más lo que se deje en `raw/`.
Alcance: conceptos de AI agents + referencia de código del repo.

## [2026-08-01] ingest | Historial de conversaciones con Claude Code (25 sesiones)

**Fuente**: transcripciones de `~/.claude/projects/c--Users-lujan-Proyectos-Cursos-ai-agents-for-beginners/*.jsonl`, del 2026-07-03 al 2026-07-31. 25 sesiones útiles, ~755 K caracteres de diálogo tras descartar tool calls y resultados.

**Decisiones de ingesta**:
- Granularidad: una página de `sources/` **por lección** (no por sesión), más los fixes en `synthesis/`.
- Datos del entorno **anonimizados** (`<TU_RECURSO>`, `<TU_PROYECTO>`); las rutas locales absolutas se sustituyen por rutas del repo.
- Las transcripciones **no se copian a `raw/`** (contienen `.env`, endpoints y rutas locales). Se citan por `sessionId` en el frontmatter de cada página; los `.jsonl` son inmutables de facto en `~/.claude/`.

**Páginas creadas** (33):
- `sources/` (10): lecciones 00, 02, 03, 08, 09, 10, 11, 12, 13, 14.
- `entities/` (6): microsoft-agent-framework, azure-ai-foundry, mcp, a2a, nlweb, cognee.
- `concepts/` (7): tool-calling, structured-outputs, workflows-como-grafo, metacognicion, context-engineering, llm-as-judge, autenticacion-azure-sin-claves.
- `synthesis/` (9): claude-vs-openai-en-foundry, fix-notebook-04-sdk-desactualizado, fix-workflow-concurrente, fix-reasoning-item-workflow, fix-imagenes-en-foundry, fix-modelos-deprecados-azure, fix-pip-notebook-colgado, fix-az-login-cache-msal, fix-git-push-divergente.
- `index.md` y `overview.md` reescritos; tesis inicial formulada.

**Conocimiento propagado entre lecciones**: el error `function_call` sin `reasoning` item quedó **sin resolver** en la lección 10 y se resolvió en la 14 (`store=True` + `context_mode="custom"` con `context_filter`). Ambas páginas se enlazaron y `fix-reasoning-item-workflow` pasó de "problema abierto" a fix verificado.

**Lecciones sin ingerir**: 01, 04, 05, 06, 07, 15, 18 — no aparecen en el historial o solo de pasada. Anotado en Pendientes.

## [2026-08-01] lint | 0 wikilinks rotos, 0 huérfanas

Comprobados los 33 ficheros: todos los enlaces wiki resuelven y todas las páginas tienen al menos un enlace entrante (salvo `overview.md`, que es la entrada por diseño y ahora se enlaza desde `index.md`).
6 pendientes registrados en el índice, entre ellos la confusión de entornos Python (`.venv` vs `Python311`/`Python312` globales) y el `SyntaxError` de `14-handoff.ipynb`, cuyo estado en el archivo está sin confirmar.

## [2026-08-02] ingest | 14-sequential.ipynb (explicación celda a celda + fix de structured outputs)

**Fuente**: `14-microsoft-agent-framework/code-samples/14-sequential.ipynb`, recorrido celda a celda en sesión didáctica (`78c7fe19`), más su ejecución real contra Foundry.

El único de los cinco notebooks de la lección 14 que no tenía sección propia en la wiki.

**Fix aplicado al notebook** (queda en el repo, no solo en la wiki): los dos agentes pasan ahora `default_options={"response_format": ...}` y se elimina de `instructions` la petición redundante `"Return structured JSON matching the X schema"`. Sin ello, el modelo devolvía `{'name': 'Vasa Museum'...}` y Pydantic daba `4 validation errors ... Field required`.

**Tercera aparición del mismo bug** (`04`, `14-concurrent`, `14-sequential`). Se pasó de "pasó dos veces" a una afirmación general en [[structured-outputs]]: *si un notebook de este curso pide JSON solo por prompt, va a fallar*. Añadida al índice como pendiente el inventario de notebooks aún sin `response_format`.

**Conocimiento nuevo propagado**:
- [[structured-outputs]] — firma reconocible del error (acierta el contenido, se inventa los rótulos) y sección nueva **"el prompt y el esquema tienen que estar de acuerdo"**: `AttractionRecommendation` tiene un solo `attraction_name`, luego el prompt necesita la palabra *single*. Corolario: al tocar un esquema, releer el prompt.
- [[workflows-como-grafo]] — tabla de las cuatro decisiones de `WorkflowBuilder`; sección del patrón secuencial; y el matiz de que la fragilidad de leer `outputs[i]` por posición está **dormida** en secuencial (el orden de llegada coincide con el declarado), no ausente.
- [[llm-as-judge]] — tercer rol del patrón en el curso: juez como **nodo de un workflow**. Argumento nuevo de por qué el juez debe ser otro agente aunque comparta modelo (sin conflicto de intereses + una instrucción corta se cumple mejor). Y la advertencia de que un juez sin tools no verifica nada: `visitor_rating: 4.6` es estimación con aspecto de dato.

**Debilidades del notebook documentadas** sin corregir: `output_executors` deprecado, `analyze_sequential_flow()` re-ejecuta el workflow en vez de leer `events` (dos llamadas de más), resumen final escrito a mano en el HTML, `# 1-10 scale` como comentario en vez de `Field(ge=1, le=10)`, numeración de pasos que salta del 4 al 8, cinco imports huérfanos.

## [2026-08-02] ingest | Verificación del fix de 14-sequential.ipynb

El fix de `response_format` de la entrada anterior queda **verificado end-to-end** contra Foundry real. Notebook completo re-ejecutado, cero salidas de tipo `error`:

```
Stockholm: front-desk -> Vasa Museum (Vasamuseet) | concierge -> 9/10, 4.5/5.0
Barcelona: user -> front-desk (JSON) -> concierge (JSON) | Total Steps: 3
```

Los campos llegan con **el nombre exacto del esquema** (`attraction_name`, antes `name`) y el JSON del recepcionista viaja sin envolver. Confirma que la restricción dura actúa, y no que el modelo obedeciera el prompt por casualidad.

Retirado el aviso "sin verificar" de [[14-microsoft-agent-framework]]. El pendiente del inventario de notebooks sin `response_format` sigue abierto.
