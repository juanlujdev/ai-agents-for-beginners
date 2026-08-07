---
type: concept
date_updated: 2026-08-07
source_count: 6
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

**No es mala suerte, es lo esperable.** Ya son tres notebooks del curso con el mismo fallo (`04`, `14-concurrent`, `14-sequential`): todos escriben *"Return structured JSON matching the X schema"* en `instructions` y ninguno pasa `response_format`. La afirmación fuerte que se puede hacer a estas alturas: **si un notebook de este curso pide JSON solo por prompt, va a fallar tarde o temprano.** Vale la pena revisarlo antes de ejecutar.

Firma del error, para reconocerlo de un golpe:

```
N validation errors for MiModelo
mi_campo   Field required   input_value={'campo_inventado': ...}
```

El modelo **acierta el contenido y se inventa los rótulos** (`name` en vez de `attraction_name`). Lógico: nunca ve la clase Python, solo lee su nombre como texto suelto y deduce los campos.

## El prompt y el esquema tienen que estar de acuerdo

Son dos caras del mismo contrato. Cuando dicen cosas distintas, algo se rompe — y el error aparece lejos de la causa.

Caso real en `14-sequential.ipynb`: `AttractionRecommendation` tiene un solo `attraction_name: str`, así que el prompt **necesita** la palabra *single* (*"provide a single recommendation"*). Sin ella el modelo devuelve cinco opciones por instinto y no hay dónde meterlas. El fallo simétrico —ampliar el esquema a una lista y olvidar el prompt— es igual de típico.

Corolario: al tocar un esquema Pydantic, releer el prompt del agente que lo rellena.

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

## No solo desde texto: extracción estructurada desde visión

Todo lo anterior asume que la fuente es un prompt de texto. En [[15-browser-use]], [[browser-use]] aplica el mismo principio a una **captura de pantalla**: `page.extract_content(prompt=..., structured_output=MiEsquema, llm=llm)` con `use_vision=True` fuerza al modelo a leer lo que "ve" en la página y devolverlo como objeto Pydantic, no como descripción en prosa. El contrato es idéntico (esquema validado vs. sugerencia en el prompt); lo que cambia es el canal de entrada. Ver la comparación completa en [[computer-use-agents]].

Fuentes: [[03-agentic-design-patterns]], [[14-microsoft-agent-framework]], [[08-multi-agent]], [[15-browser-use]]
