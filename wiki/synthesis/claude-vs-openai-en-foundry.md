---
type: synthesis
date_updated: 2026-08-01
source_count: 3
---

# Modelos Claude vs OpenAI en Azure AI Foundry

**El hallazgo transversal del curso.** Desplegar un modelo `claude-*` en [[azure-ai-foundry]] funciona para texto libre, pero rompe en tres escenarios concretos. Explica varios errores que a primera vista parecían bugs del código de las lecciones.

## Qué falla y por qué

| Escenario | Síntoma | Causa |
| --- | --- | --- |
| [[structured-outputs]] (`response_format`) | `ValidationError: Invalid JSON: expected value at line 1 column 1` | Structured Outputs impone el `json_schema` **desde el servidor**. La capa de compatibilidad de Foundry no lo aplica a modelos Claude: responden prosa, o JSON envuelto en ` ```json `, y Pydantic lo rechaza. |
| [[tool-calling]] multi-turno con sesión | `400 — No tool output found for function call call_toolu_…` | La Responses API con estado (`store`/`previous_response_id`) reenvía el historial en el turno 2, pero el output de la tool call del turno 1 no queda bien adjunto del lado servidor. |
| Texto libre sin tools | Funciona | — |

Pista diagnóstica: el prefijo `call_toolu_` en el id de la function call delata que quien responde es un modelo Anthropic, aunque se esté hablando por un cliente OpenAI.

## Regla práctica

- **Texto libre, sin `response_format` ni sesión multi-turno** → cualquier modelo del catálogo, incluido Claude.
- **`response_format`, o tool-calling con `AgentSession`** → hace falta un deployment OpenAI vigente (`gpt-5-mini` es el que se usó aquí).

No hace falta elegir uno solo: se pueden tener ambos desplegados y crear un cliente distinto por celda, que es lo que se hizo en [[03-agentic-design-patterns]].

## Cómo se diagnosticó

Aislando la variable antes de tocar el código: reproducir el fallo **sin tools**. Al fallar igual, quedó descartado que el problema fuera la celda o las tools, y señaló al modelo. Sin ese paso, lo natural habría sido reescribir el schema o el prompt — arreglando un síntoma que no era la causa.

Fuentes: [[02-explore-agentic-frameworks]], [[03-agentic-design-patterns]], [[00-course-setup]]
