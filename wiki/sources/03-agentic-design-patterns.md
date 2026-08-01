---
type: source-summary
date_updated: 2026-08-01
leccion: 03-agentic-design-patterns
sesiones: [c7ce9847]
fechas_origen: 2026-07-11
---

# 03 — Agentic design patterns

`03-python-agent-framework.ipynb` recorre tres patrones. El Pattern 2 es **structured outputs**: forzar que el agente devuelva un objeto Pydantic validado en lugar de texto libre.

## Structured outputs — [[structured-outputs]]

```python
class DestinationRecommendation(BaseModel):
    destination: str
    available: bool
    best_season: str
    highlights: list[str]
    estimated_budget_usd: int

response = await structured_agent.run(
    "Recommend 3 destinations ...",
    options={"response_format": TravelRecommendations},
)
result: TravelRecommendations = response.value
```

`response_format` no es una sugerencia al prompt: activa **Structured Outputs** de la API de OpenAI, donde el servidor impone el `json_schema`. Si el modelo no soporta esa imposición, la respuesta llega como prosa y Pydantic falla al validarla.

## El fallo y su causa raíz

`ValidationError: Invalid JSON: expected value at line 1 column 1`, con `input_value="Based on my recommendati..."`.

La causa **no** era el código de la celda ni las tools: era el modelo desplegado (`claude-haiku-4-5`). Se comprobó con una prueba mínima sin tools, que también falló — ahí Claude devolvió JSON correcto pero envuelto en un bloque markdown ` ```json `, que el parser rechaza igualmente. Las celdas anteriores funcionaban porque solo generaban texto libre.

Detalle metodológico: el diagnóstico salió de aislar la variable (reproducir sin tools) antes de tocar nada, no de probar arreglos. Ver [[claude-vs-openai-en-foundry]].

## Arreglo aplicado

Un cliente separado solo para la celda de structured outputs, dejando el resto del notebook con el modelo original:

```python
# Structured outputs (`response_format`) requiere un modelo OpenAI que soporte
# json_schema forzado por el servidor; los deployments claude-* lo ignoran.
structured_provider = FoundryChatClient(
    project_endpoint=endpoint,
    model="gpt-5-mini",
    credential=DefaultAzureCredential(),
)
structured_agent = structured_provider.as_agent(...)
```

Verificado end-to-end (tools + `response_format` + gpt-5-mini) contra el proyecto de Foundry: devuelve el objeto validado.

**No hace falta clave de API**: `FoundryChatClient` con `DefaultAzureCredential` usa la sesión de Entra ID de `az login` → [[autenticacion-azure-sin-claves]].

Alternativa descartada: quitar `response_format`, pedir el schema en el prompt y limpiar los fences con regex antes de `model_validate_json`. Es un workaround best-effort y traiciona el propósito de la lección.

Ver también: [[02-explore-agentic-frameworks]], [[microsoft-agent-framework]]
