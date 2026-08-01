---
type: entity
date_updated: 2026-08-01
source_count: 1
---

# A2A (Agent-to-Agent)

Protocolo para que **un agente delegue en otro agente autónomo**, posiblemente de otra empresa y con otro LLM por dentro. Hermano de [[mcp]], no sustituto.

> MCP es usar una calculadora. A2A es llamar por teléfono a un contador para que te resuelva algo.

## Piezas

| Componente | Qué es |
| --- | --- |
| **Agent Card** | El currículum del agente: nombre, capacidades, URL. Publicado en `https://empresa.com/.well-known/agent.json`, misma convención que `robots.txt` |
| **Agent Executor** | Pasa el contexto de la conversación al agente remoto |
| **Artifact** | La entrega: el resultado + una descripción de lo completado + el texto de contexto |
| **Event Queue** | Cola que evita que se corte la conexión en tareas largas |

## Cómo funciona el descubrimiento

Tu agente hace un GET a la URL `.well-known`, se descarga el Agent Card, lee el `endpoint` y ya sabe a dónde mandar tareas. Pero **alguien tiene que decirle qué agentes existen**: o configuración manual (lista de URLs) o un **Agent Registry** (catálogo donde las empresas registran sus agentes). Igual que tú: o conoces la web de Hertz, o la buscas en Google.

## Detalle importante

Cuando llega el Artifact, **la conexión se cierra**. Es "pido tarea → recibo artifact → listo", no una sesión continua.

## No confundir

Varios agentes colaborando **no es A2A** por sí solo. `github-mcp` tiene tres agentes en cadena y no usa A2A: es orquestación con `WorkflowBuilder` dentro de un mismo framework, sin Agent Cards ni Artifacts → [[workflows-como-grafo]].

[[microsoft-agent-framework]] sí interopera con A2A real, vía `A2AAgent(name=..., url=...)`.

Fuentes: [[11-agentic-protocols]]
