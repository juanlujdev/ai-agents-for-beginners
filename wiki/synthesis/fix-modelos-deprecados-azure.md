---
type: synthesis
date_updated: 2026-08-01
source_count: 2
---

# Fix: modelos deprecados en el catálogo de Foundry

## Síntoma

El README del curso pide desplegar `gpt-4o`. En el catálogo de [[azure-ai-foundry]] aparece marcado como **en desuso / deprecated**, igual que `gpt-4o-mini`.

## Por qué pasa

Microsoft retira modelos del catálogo progresivamente. El curso se escribió cuando `gpt-4o` era el modelo por defecto.

## El fix

El curso **no depende del nombre concreto del modelo**. Solo necesita un modelo de chat vigente que soporte lo que pide la lección. Basta con:

1. **Models + Endpoints → Deploy model**, filtrar por proveedor **OpenAI**.
2. Comprobar en "Hechos rápidos" que el Ciclo de vida **no** diga Deprecated.
3. Desplegar y copiar el **nombre del deployment** (no el ID del modelo — no siempre coinciden).
4. Poner ese nombre en `.env`:
   ```env
   AZURE_AI_MODEL_DEPLOYMENT_NAME=<nombre-del-deployment>   # flujo Foundry
   AZURE_OPENAI_DEPLOYMENT=<nombre-del-deployment>          # flujo Azure OpenAI directo
   ```
5. **Reiniciar el kernel** del notebook. `load_dotenv()` no relee el `.env` si el proceso ya lo cargó en memoria: sin reinicio se sigue usando el valor viejo y parece que el cambio no surtió efecto.

En este proyecto la elección final fue **`gpt-5-mini`**.

## Cuidado al elegir

Que un modelo esté vigente no basta: los deployments `claude-*` del catálogo son vigentes y aun así fallan en structured outputs y tool-calling con sesión → [[claude-vs-openai-en-foundry]].

Fuentes: [[00-course-setup]], [[02-explore-agentic-frameworks]]
