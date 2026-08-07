---
type: synthesis
date_updated: 2026-08-07
source_count: 1
---

# Fix: placeholder de Azure AI Search tratado como configuración real

## Síntoma

Al ejecutar la compuerta de evaluación (`evaluation_gate`) de `16-python-agent-framework.ipynb`, las tres preguntas de política (devoluciones, envío, garantía) fallan o puntúan bajo. La de estado de pedido (`get_order_status`, sin red) pasa al 100%. Traza real:

```
Function failed. Error: URL has an invalid label.
Function 'search_policies' raised an exception; returning an error result to the model.
Exception: ServiceResponseError('URL has an invalid label.')
...
Evaluation pass rate: 75% (gate: 80%)
Deploy allowed: False
```

## Por qué pasa

`search_policies` decide entre el buscador real de Azure AI Search y un fallback en memoria con un interruptor booleano:

```python
USE_AZURE_SEARCH = bool(os.getenv("AZURE_SEARCH_SERVICE_ENDPOINT") and os.getenv("AZURE_SEARCH_API_KEY"))
```

El `.env` del repo trae, para la lección 05 (Agentic RAG, sin ingerir todavía), valores de **plantilla** sin rellenar:

```env
AZURE_SEARCH_SERVICE_ENDPOINT="https://..."
AZURE_SEARCH_API_KEY="..."
```

`bool("https://...")` es `True` — un string de plantilla no está vacío, así que el interruptor no distingue "configurado de verdad" de "queda un placeholder sin rellenar". `USE_AZURE_SEARCH` da `True`, `search_policies` intenta `_azure_search()` de verdad, y el SDK de Azure intenta resolver `https://...` como host: el segmento `...` no es una etiqueta de dominio válida → `ServiceResponseError('URL has an invalid label.')`.

Es el mismo patrón de fondo que otros fixes de configuración del curso ([[fix-modelos-deprecados-azure]], [[fix-az-login-cache-msal]]): **un valor no vacío no implica un valor válido**, y el código de la lección no lo comprueba.

## El fix

Fix aplicado de verdad en el `.env` del repo — solo tocó una de las dos variables:

```env
#AZURE_SEARCH_SERVICE_ENDPOINT="https://..."
AZURE_SEARCH_SERVICE_ENDPOINT=""
AZURE_SEARCH_API_KEY="..."          # sigue siendo el placeholder, sin tocar
```

Basta con vaciar **una** de las dos porque `and` en Python corta en el primer operando falsy:
`os.getenv("AZURE_SEARCH_SERVICE_ENDPOINT") and os.getenv("AZURE_SEARCH_API_KEY")` con el endpoint en `""` se evalúa como `"" and "..."` → devuelve `""` sin mirar la segunda variable → `bool("")` es `False`. `USE_AZURE_SEARCH` da `False` y `search_policies` cae al buscador en memoria (`_in_memory_search`), sin depender de red. El `AZURE_SEARCH_API_KEY` con placeholder queda inofensivo mientras el endpoint siga vacío — pero si alguien rellena el endpoint más adelante sin fijarse en la clave, el mismo bug puede reaparecer a medias.

Si se quiere probar Azure AI Search de verdad, el endpoint debe tener el formato real del servicio: `https://<nombre-del-servicio>.search.windows.net`, sin los tres puntos de plantilla, y la clave también debe ser real.

## Nota de diseño, no solo de configuración

El propio bug es una demostración en vivo del motivo de ser de la compuerta de evaluación descrita en [[16-deploying-scalable-agents]]: detectó automáticamente una herramienta rota (75% < 80% de umbral) sin que nadie tuviera que leer las respuestas del agente a ojo, y bloqueó el "deploy" simulado de la celda `release()`.

Fuente: `16-deploying-scalable-agents/code_samples/16-python-agent-framework.ipynb`, ejecución real contra Foundry en sesión de conversación (explicación celda a celda + depuración del fallo de `search_policies`).
