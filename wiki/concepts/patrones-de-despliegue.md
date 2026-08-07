---
type: concept
date_updated: 2026-08-07
source_count: 1
---

# Patrones de despliegue de agentes

Tres formas de decidir **dónde vive el bucle de razonamiento** de un agente en producción, con un trade-off constante entre control y superficie operativa que hay que mantener uno mismo.

| Patrón | Dónde corre el loop | Cuándo usarlo | Trade-off |
| --- | --- | --- | --- |
| **Client-hosted** | Dentro de tu propio proceso, llamando al proveedor de modelo directamente | Control total del loop, middleware a medida, agente embebido en un backend existente | Tú te encargas de escalado, estado y resiliencia |
| **Hosted Agent** ([[azure-ai-foundry]] Agent Service) | El agente se registra como recurso; Foundry aloja el loop, guarda threads, aplica RBAC y seguridad de contenido | Durabilidad, observabilidad y gobernanza ya resueltas | Menos control de bajo nivel a cambio de un runtime gestionado |
| **Agent Workflow** | Varios agentes/tools compuestos en un grafo con control de flujo explícito | Una tarea cruza varios agentes especializados, o necesita un paso de aprobación en medio | Más piezas móviles; necesita observabilidad a nivel de orquestación |

El tercer patrón no es nuevo técnicamente: es [[workflows-como-grafo]] (`WorkflowBuilder`, nodos, edges) aplicado **a escala de despliegue**, con la variante de que uno de los nodos puede ser un `Human Approval Node` que pausa el grafo hasta que alguien aprueba una acción de negocio (reembolso, borrado de cuenta) — mismo primitivo `RequestInfoEvent` que ya pausaba workflows por falta de datos, ahora pausando por falta de permiso.

## La progresión no es "elegir uno"

Los tres patrones se combinan: un despliegue típico empieza client-hosted en desarrollo, se registra como Hosted Agent para producción, y usa Agent Workflows cuando una sola llamada no basta (triage → resolución → aprobación → acción). El README de [[16-deploying-scalable-agents]] ilustra esto con tres diagramas — desarrollo, deployment (pipeline CI con compuerta de evaluación), runtime (agente hospedado con router, RAG, memoria, tools MCP, tracing y aprobación humana todos conectados) — como tres fotos del mismo agente en distintas etapas de su vida, no tres sistemas distintos.

Fuentes: [[16-deploying-scalable-agents]]
