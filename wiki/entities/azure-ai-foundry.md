---
type: entity
date_updated: 2026-08-07
source_count: 8
---

# Azure AI Foundry

Plataforma de Azure (ai.azure.com) donde se despliegan los modelos que usa el curso. Antes "Azure AI Studio"; el servicio de agentes se llama **Foundry Agent Service**.

## Foundry Agent Service como Hosted Agent

En producción ([[16-deploying-scalable-agents]]), el agente se **registra como recurso** en Foundry en vez de vivir dentro del proceso de la app: Foundry aloja el bucle de razonamiento, persiste los threads, aplica RBAC y seguridad de contenido, y lo hace visible en el portal. La app cliente pasa a ser un cliente delgado que crea threads y lee respuestas. Es uno de los tres [[patrones-de-despliegue]], con **Model Router** nativo para enrutar por complejidad y un thread store que es lo que permite que el agente sea sin estado en el proceso y escale horizontalmente.

## Los dos flujos, que se confunden

El curso usa **dos rutas distintas** hacia el modelo, con variables de entorno diferentes:

| Flujo | Cliente | Variables |
| --- | --- | --- |
| **Proyecto Foundry** | `FoundryChatClient` | `AZURE_AI_PROJECT_ENDPOINT` (con `/api/projects/<proyecto>`), `AZURE_AI_MODEL_DEPLOYMENT_NAME` |
| **Azure OpenAI directo** | `OpenAIChatClient` | `AZURE_OPENAI_ENDPOINT` (URL base, **sin** `/openai/v1`), `AZURE_OPENAI_DEPLOYMENT` |

Tener uno configurado no sirve para el otro. El endpoint del proyecto se copia de la pestaña **Overview**, no de Implementaciones (ahí sale truncado).

## Lo que hay que saber al desplegar

- El catálogo **retira modelos**: `gpt-4o` y `gpt-4o-mini` aparecen deprecados → [[fix-modelos-deprecados-azure]].
- Se copia el **nombre del deployment**, que no siempre coincide con el ID del modelo.
- **No todos los modelos vigentes valen**: los `claude-*` del catálogo fallan en structured outputs y en tool-calling con sesión → [[claude-vs-openai-en-foundry]].
- Se pueden tener varios deployments a la vez y crear un cliente distinto por celda. Es lo que se hizo en [[03-agentic-design-patterns]].

## Autenticación

Sin claves de API: `AzureCliCredential` / `DefaultAzureCredential` usan la sesión de `az login` → [[autenticacion-azure-sin-claves]].

## Limitaciones propias del cliente de Foundry

- Descarta las salidas ricas (imágenes) de las herramientas → [[fix-imagenes-en-foundry]].
- No guarda estado en servidor por defecto, lo que rompe los modelos de razonamiento → [[fix-reasoning-item-workflow]].

Otros servicios de Azure que aparecen en el curso: **Azure AI Search** (índice para RAG y Structured RAG, imprescindible en `github-mcp`), **Bing Grounding** (búsqueda web, `BING_CONNECTION_ID`) y **Azure AI Vision** (OCR real, recomendado en la lección 10).

Fuentes: [[00-course-setup]], [[02-explore-agentic-frameworks]], [[10-ai-agents-production]], [[14-microsoft-agent-framework]], [[16-deploying-scalable-agents]]
