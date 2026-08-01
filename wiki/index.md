---
type: index
date_updated: 2026-08-01
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
| [[14-microsoft-agent-framework]] | 14 | Concurrent, conditional, handoff, middleware, LangGraph hospedado |

**Sin ingerir todavía**: lecciones 01, 04, 05, 06, 07, 15 y 18 — no aparecen en el historial de conversaciones o solo de pasada.

## Entidades

| Página | Qué es |
| --- | --- |
| [[microsoft-agent-framework]] | El framework del curso: API, trampas de la versión instalada |
| [[azure-ai-foundry]] | La plataforma: dos flujos de configuración, límites del cliente |
| [[mcp]] | Protocolo agente ↔ herramientas |
| [[a2a]] | Protocolo agente ↔ agente |
| [[nlweb]] | Protocolo agente/humano ↔ sitio web |
| [[cognee]] | Grafo de conocimiento como memoria de largo plazo |

## Conceptos

| Página | Idea |
| --- | --- |
| [[tool-calling]] | El LLM decide *cuándo* llamar; lee docstring y `Annotated`, no el código |
| [[structured-outputs]] | Pedir JSON por prompt ≠ forzarlo con `response_format` |
| [[workflows-como-grafo]] | Nodos, edges, y por qué en fan-out los agentes no se hablan |
| [[metacognicion]] | Cambiar la estrategia, no el resultado |
| [[context-engineering]] | La información correcta, no más información |
| [[llm-as-judge]] | Un agente puntuando a otro: usos y límites |
| [[autenticacion-azure-sin-claves]] | `az login` en vez de claves; caducidad y token providers |

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

## Pendientes

| Pendiente | Tipo | Detectado |
| --- | --- | --- |
| Lecciones 01, 04, 05, 06, 07, 15, 18 sin página de fuente | hueco | 2026-08-01 |
| `fix-reasoning-item-workflow` no está verificado en el notebook de la lección 10, solo en el de la 14 | verificación | 2026-08-01 |
| Confusión de entornos: paquetes instalados en `.venv`, `Python311` y `Python312` globales; el kernel de VS Code puede apuntar a cualquiera | entorno | 2026-08-01 |
| Bug del `SyntaxError` en `14-handoff.ipynb` (`> ` sin valor): no consta si llegó a corregirse en el archivo | seguimiento | 2026-08-01 |
| `github-mcp` nunca se ejecutó (falta Azure AI Search); las páginas sobre él son solo lectura de código | hueco | 2026-08-01 |
| Sin página propia: Semantic Kernel, AutoGen, Mem0, Chainlit, Azure AI Search | concepto sin página | 2026-08-01 |
