---
type: entity
date_updated: 2026-08-01
source_count: 1
---

# NLWeb

Convierte cualquier sitio web en algo **consultable en lenguaje natural**, tanto por humanos como por agentes. En vez de navegar menús y filtros, le preguntas al sitio: *"busco un hotel familiar con piscina en Honolulu"*.

## Cómo funciona

1. **Ingesta**: el catálogo del sitio se convierte en **embeddings** y se guarda en una base vectorial.
2. **Consulta**: la pregunta va al LLM para entenderla y, en paralelo, se buscan por similitud los ítems más parecidos.
3. **Respuesta**: en lenguaje natural pero **basada en datos reales del catálogo** — no inventa, que es la clave.
4. **Para agentes**: NLWeb también funciona como servidor [[mcp]], así que un agente externo puede llamar a `ask(...)` y recibir JSON estructurado sin que nadie navegue nada.

Un recepcionista con IA en cada sitio web, que además atiende a otros agentes.

## Componentes

NLWeb Application (el motor) · NLWeb Protocol (las reglas de pregunta/respuesta) · MCP Server (la puerta para otros agentes) · Embedding Models · Vector Database.

## Encaje con los otros protocolos

Es la tercera pata junto a [[mcp]] y [[a2a]]. Los tres se combinan: un Travel Agent (A2A) delega en un Hotel Agent, que usa NLWeb —que es también MCP— para consultar el catálogo real.

Fuentes: [[11-agentic-protocols]]
