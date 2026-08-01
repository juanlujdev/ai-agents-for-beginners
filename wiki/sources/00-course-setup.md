---
type: source-summary
date_updated: 2026-08-01
leccion: 00-course-setup
sesiones: [84b83fd8, e236fce2]
fechas_origen: 2026-07-03 .. 2026-07-05
---

# 00 — Course setup

Puesta en marcha del entorno: venv, Azure AI Foundry, `.env`, dependencias y primera ejecución del notebook de la lección 01.

## Secuencia que funcionó

1. Clonar repo + venv con **Python 3.12** (el README exige 3.12+).
2. `cp .env.example .env`.
3. En [[azure-ai-foundry]] (ai.azure.com): crear hub → proyecto → desplegar un modelo (**Models + Endpoints → Deploy model**).
4. Rellenar `.env`:
   ```env
   AZURE_AI_PROJECT_ENDPOINT=https://<TU_RECURSO>.services.ai.azure.com/api/projects/<TU_PROYECTO>
   AZURE_AI_MODEL_DEPLOYMENT_NAME=<nombre-del-deployment>
   ```
   El endpoint se copia de la pestaña **Overview** del proyecto, no de Implementaciones (ahí sale truncado).
5. `az login` → [[autenticacion-azure-sin-claves]].
6. `pip install -r requirements.txt` (~100 paquetes, 3-10 min es normal en Windows).

## Puntos que costaron

- **El primer deployment fue `claude-haiku-4-5`**, no un modelo OpenAI. Funciona para texto libre, pero rompe en cuanto la lección usa structured outputs o tool-calling multi-turno → [[claude-vs-openai-en-foundry]].
- **`gpt-4o` aparecía como deprecado** en el catálogo. El curso no depende de ese nombre concreto: sirve cualquier modelo de chat vigente → [[fix-modelos-deprecados-azure]].
- **`pip install` parecía colgado.** No lo estaba: el proceso acumulaba CPU. Se verifica mirando el PID, no el parpadeo del cursor.
- **`%pip install` de la lección 01 tardó ~10 min** porque el resolvedor de pip recalculaba dependencias ya instaladas por `requirements.txt`. Terminó bien; hay que reiniciar el kernel después.
- **La sesión de `az login` caduca.** Al día siguiente el notebook falló con `AzureCliCredential`, y el reintento dio `does not exist in MSAL token cache` → [[fix-az-login-cache-msal]].

## Dudas resueltas

- El `.env` es un archivo en disco: apagar el PC no lo borra.
- No hace falta `deactivate` antes de apagar; al volver, `.venv\Scripts\activate`.
- Las variables `AZURE_SEARCH_*`, `GITHUB_TOKEN`, `BING_CONNECTION_ID`, `MINIMAX_*` solo hacen falta en las lecciones 5, 6 y 8.
- El aviso de VS Code sobre `ipykernel` es normal: se instala con **Install** y no indica problema de versión.

Ver también: [[microsoft-agent-framework]], [[02-explore-agentic-frameworks]]
