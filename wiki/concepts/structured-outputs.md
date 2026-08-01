---
type: concept
date_updated: 2026-08-01
source_count: 5
---

# Structured outputs

Forzar que el agente devuelva un objeto **validado** en lugar de texto libre. El concepto que más errores ha causado en este curso.

## Pedir JSON ≠ forzar JSON

Esta es la distinción central:

| | Qué es | Fiabilidad |
| --- | --- | --- |
| *"Return structured JSON matching the X schema"* en las instrucciones | Una **sugerencia** en el prompt | El modelo la interpreta libremente: añade campos, anida donde esperabas texto, omite obligatorios |
| `response_format=ModeloPydantic` | El modelo Pydantic se convierte en JSON Schema y viaja como **restricción dura** (`text_format`/`json_schema` en la request) | El servidor impone el esquema |

Los errores de `14-concurrent.ipynb` venían justo de confundir ambas → [[fix-workflow-concurrente]].

## Cómo se pasa, según el sitio

```python
# En as_agent de MAF
agent = provider.as_agent(..., default_options={"response_format": MiModelo})
agent = client.as_agent(..., default_options=ChatOptions(response_format=MiModelo))

# En una ejecución puntual
response = await agent.run("...", options={"response_format": MiModelo})
result = response.value        # ← el objeto validado, no .text
```

`as_agent(response_format=X)` **directo no existe**: es un campo de `ChatOptions` → [[fix-notebook-04-sdk-desactualizado]].

## Requiere un modelo que lo soporte

Structured outputs es una función de generación restringida **de OpenAI**. Los deployments `claude-*` de Foundry la ignoran: responden prosa, o JSON envuelto en ` ```json `, y Pydantic falla con `Invalid JSON: expected value at line 1 column 1` → [[claude-vs-openai-en-foundry]].

## Restricción del schema en Azure/OpenAI

**Todos los campos deben ir en el array `required`.** No se admiten opcionales con valor por defecto:

```python
priority_override: bool          # ✅
priority_override: bool = False  # ❌ no válido en structured outputs
```

## Para qué sirve realmente

Convierte la salida del LLM en un dato con el que se puede programar. En los workflows es lo que permite que una función de condición decida la ruta (`result.has_availability`) o que un executor traduzca texto crudo a un objeto tipado antes de pasarlo al siguiente nodo → [[workflows-como-grafo]].

Fuentes: [[03-agentic-design-patterns]], [[14-microsoft-agent-framework]], [[08-multi-agent]]
