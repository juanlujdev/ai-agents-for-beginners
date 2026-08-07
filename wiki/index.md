---
type: index
date_updated: 2026-08-07
---

# Índice de la wiki

Catálogo de todas las páginas. Se actualiza en **cada** ingesta, consulta archivada y lint.
Entrada: [[overview]] · Cronología: [[log]]

## Fuentes

Una página por lección. Reconstruidas a partir del historial de conversaciones (`~/.claude/projects/…/*.jsonl`), que se cita por `sessionId` en el frontmatter de cada página.

| Página | Lección | Resumen |
| --- | --- | --- |
| [[00-course-setup]] | 00 | Venv, Foundry, `.env`, `az login`, dependencias y primera ejecución |
| [[02-explore-agentic-frameworks]] | 02 | Primer agente con tools, los dos flujos de variables de entorno |
| [[03-agentic-design-patterns]] | 03 | Structured outputs con Pydantic y el fallo que destapó |
| [[08-multi-agent]] | 08 | Workflows: fan-out concurrente y enrutamiento condicional |
| [[09-metacognition]] | 09 | Estrategia vs resultado, corrective RAG, riesgos de `exec`/SQL |
| [[10-ai-agents-production]] | 10 | Traces y spans, evaluación offline/online, coste, demo de gastos |
| [[11-agentic-protocols]] | 11 | MCP, A2A y NLWeb; `mcp-agents` y `github-mcp` |
| [[12-context-engineering]] | 12 | Los 4 fallos de contexto, scratchpad, compresión |
| [[13-agent-memory]] | 13 | 7 tipos de memoria, knowledge agent, Cognee vs MAF |
| [[14-microsoft-agent-framework]] | 14 | Sequential, concurrent, conditional, handoff, middleware, LangGraph hospedado |
| [[15-browser-use]] | 15 | Computer use agents: Browser-Use + Playwright/CDP, Agente vs Actor, extracción con visión |
| [[16-deploying-scalable-agents]] | 16 | Prototipo → producción: patrones de despliegue, evaluación como compuerta, routing/cache, observabilidad, smoke tests; notebook ejecutado celda a celda contra Foundry real |

**Sin ingerir todavía**: lecciones 01, 04, 05, 06, 07 y 18 — no aparecen en el historial de conversaciones o solo de pasada.

## Entidades

| Página | Qué es |
| --- | --- |
| [[microsoft-agent-framework]] | El framework del curso: API, trampas de la versión instalada |
| [[azure-ai-foundry]] | La plataforma: dos flujos de configuración, límites del cliente |
| [[mcp]] | Protocolo agente ↔ herramientas |
| [[a2a]] | Protocolo agente ↔ agente |
| [[nlweb]] | Protocolo agente/humano ↔ sitio web |
| [[cognee]] | Grafo de conocimiento como memoria de largo plazo |
| [[browser-use]] | Framework de automatización de navegador dirigida por IA sobre Playwright |

## Conceptos

| Página | Idea |
| --- | --- |
| [[tool-calling]] | El LLM decide *cuándo* llamar; lee docstring y `Annotated`, no el código |
| [[structured-outputs]] | Pedir JSON por prompt ≠ forzarlo con `response_format` (también desde visión) |
| [[workflows-como-grafo]] | Nodos, edges, y por qué en fan-out los agentes no se hablan |
| [[metacognicion]] | Cambiar la estrategia, no el resultado |
| [[context-engineering]] | La información correcta, no más información |
| [[llm-as-judge]] | Un agente puntuando a otro: usos y límites |
| [[autenticacion-azure-sin-claves]] | `az login` en vez de claves; caducidad y token providers |
| [[computer-use-agents]] | Agente que actúa sobre una interfaz visual; Agente vs Actor; patrón híbrido |
| [[patrones-de-despliegue]] | Client-hosted, Hosted Agent y Agent Workflow: dónde vive el bucle en producción |

## Síntesis

Hallazgos que cruzan lecciones y correcciones de código con su motivo.

| Página | Qué resuelve |
| --- | --- |
| [[claude-vs-openai-en-foundry]] | **El hallazgo transversal**: qué rompe al desplegar un modelo Claude |
| [[fix-notebook-04-sdk-desactualizado]] | Nueve incompatibilidades encadenadas con `agent_framework` 1.10.0 |
| [[fix-workflow-concurrente]] | Orden de `get_outputs()` + structured outputs de verdad |
| [[fix-reasoning-item-workflow]] | `store=True` y filtrar la mecánica de tools entre agentes |
| [[fix-imagenes-en-foundry]] | Foundry descarta las imágenes que devuelven las tools |
| [[fix-modelos-deprecados-azure]] | `gpt-4o` deprecado: cómo elegir sustituto |
| [[fix-pip-notebook-colgado]] | `!pip` vs `%pip`, y cuándo está lento pero vivo |
| [[fix-az-login-cache-msal]] | `az account clear` cuando el login entra en bucle |
| [[fix-git-push-divergente]] | Rebase tras sincronizar con el upstream del fork |
| [[fix-hotel-booking-sample-imports]] | `ChatMessage`/`ai_function`/`Role.USER` rotos en un script `.py` de la 14, no solo en notebooks |
| [[fix-browser-use-windows-jupyter]] | Chrome no encontrado, `.env` mal nombrado, y `NotImplementedError` de asyncio en Jupyter/Windows |
| [[fix-azure-search-placeholder-url]] | Placeholder `"https://..."` en `.env` pasa el `bool()`, `search_policies` intenta Azure real y rompe la compuerta de evaluación |

## Pendientes

| Pendiente | Tipo | Detectado |
| --- | --- | --- |
| Lecciones 01, 04, 05, 06, 07, 18 sin página de fuente | hueco | 2026-08-01 |
| `fix-reasoning-item-workflow` no está verificado en el notebook de la lección 10, solo en el de la 14 | verificación | 2026-08-01 |
| Confusión de entornos: paquetes instalados en `.venv`, `Python311` y `Python312` globales; el kernel de VS Code puede apuntar a cualquiera | entorno | 2026-08-01 |
| Bug del `SyntaxError` en `14-handoff.ipynb` (`> ` sin valor): no consta si llegó a corregirse en el archivo | seguimiento | 2026-08-01 |
| `github-mcp` nunca se ejecutó (falta Azure AI Search); las páginas sobre él son solo lectura de código | hueco | 2026-08-01 |
| Sin página propia: Semantic Kernel, AutoGen, Mem0, Chainlit, Azure AI Search | concepto sin página | 2026-08-01 |
| `output_executors` deprecado en favor de `output_from`: sin comprobar si la semántica es idéntica ni si afecta a los otros notebooks de la 14 | API | 2026-08-02 |
| Los notebooks que aún piden JSON solo por prompt (sin `response_format`) no están inventariados; ya han fallado tres | seguimiento | 2026-08-02 |
| Los demás scripts `.py` de `code-samples/` (fuera de notebooks) no se han auditado contra el SDK instalado; `hotel_booking_workflow_sample.py` tenía 3 imports rotos sin que nadie lo hubiera ejecutado | seguimiento | 2026-08-07 |
| `take_screenshot()` en `15-browser-user.ipynb` falla (`a bytes-like object is required, not 'str'`) por inconsistencia entre las dos implementaciones de `AirbnbSearchAgent` sobre qué devuelve `page.screenshot()` de Browser-Use (`playwright_browser` vs `cdp_url`) — sin corregir | bug sin fix | 2026-08-07 |
| `langchain-openai` se instala en `15-browser-user.ipynb` pero no se usa (`ChatAzureOpenAI` viene de `browser_use`) — dependencia muerta | limpieza | 2026-08-07 |
| Project Opal cita las lecciones 04, 06, 07 y 18 como referencia (tools reutilizables, human-in-the-loop, planificación, agentes seguros); esas lecciones siguen sin página propia, así que la comparación en [[computer-use-agents]] queda con enlaces pendientes | hueco | 2026-08-07 |
| `.env` del repo tiene ahora `AZURE_OPENAI_DEPLOYMENT` y `AZURE_OPENAI_CHAT_DEPLOYMENT_NAME` con el mismo valor duplicado, por las dos convenciones de nombre que conviven (Entra ID vs `browser_use.ChatAzureOpenAI`) — ver [[fix-browser-use-windows-jupyter]]; sin limpiar | limpieza | 2026-08-07 |
| El fix de `NotImplementedError` en [[fix-browser-use-windows-jupyter]] es específico de `ipykernel` 7.3.0 en Windows; no comprobado si sigue haciendo falta en versiones más nuevas de `ipykernel` que puedan dejar de forzar `WindowsSelectorEventLoopPolicy` | verificación | 2026-08-07 |
El pipeline de smoke tests de la lección 16 (`tests/lesson-16-smoke-tests.json`, `.github/workflows/smoke-test.yml`) sigue sin ejecutarse — el notebook (`16-python-agent-framework.ipynb`) ya se ejecutó y verificó, ver [[fix-azure-search-placeholder-url]] | verificación | 2026-08-07 |
| `.env` tiene placeholders (`AZURE_SEARCH_SERVICE_ENDPOINT="https://..."`, `AZURE_SEARCH_API_KEY="..."`) para la lección 05 (sin ingerir) que un `bool()` no distingue de configuración real; puede volver a romper cualquier notebook que use `search_policies`/RAG hasta que la lección 05 se ingiera y rellene con valores reales o se documente el patrón como regla general | limpieza | 2026-08-07 |
