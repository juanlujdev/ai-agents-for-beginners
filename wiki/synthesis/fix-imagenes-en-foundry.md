---
type: synthesis
date_updated: 2026-08-01
source_count: 1
---

# Fix: pasar imágenes a un agente con FoundryChatClient

Un fallo con **tres capas**, cada una descubierta al arreglar la anterior. El notebook original avisaba solo de la primera.

## Capa 1 — base64 como texto no es una imagen

```python
@tool(approval_mode="never_require")
def load_receipt_image(image_path: str = "receipt.jpg") -> str:
    image_data = base64.b64encode(open(image_path,"rb").read()).decode("utf-8")
    return f"data:image/jpeg;base64,{image_data}"
```

Síntoma: `No expenses detected`. El modelo recibe un bloque de texto gigantesco, no una imagen. Necesita un mensaje de tipo `image_url` / `input_image`. Esto **sí** lo advierte el propio notebook, que además señala el riesgo de desbordar la ventana de contexto.

## Capa 2 — el framework convierte los `str` en texto plano

Cualquier `str` devuelto por una tool lo envuelve el framework en `Content.from_text(...)`. Primer intento de fix:

```python
return Content.from_data(image_bytes, media_type="image/jpeg")   # type='data' → input_image
```

Necesario, pero **insuficiente**.

## Capa 3 — Foundry descarta las salidas ricas de herramientas

Síntoma nuevo: el agente dice que no tiene acceso al fichero. La causa está en el código fuente del paquete instalado:

```python
# agent_framework_foundry/_chat_client.py:153
SUPPORTS_RICH_FUNCTION_OUTPUT: ClassVar[bool] = False
```

Cuando una tool devuelve contenido rico (una imagen), **`FoundryChatClient` lo descarta en silencio** y solo reenvía la parte de texto del resultado — vacía. El agente no recibe un error, recibe un resultado vacío, y responde razonablemente que no puede acceder al fichero.

No es un límite del modelo: `gpt-5-mini` sí ve imágenes. Es una limitación del cliente de Foundry con salidas de herramientas.

## Fix definitivo

Dejar de pedir la imagen por tool y **adjuntarla al mensaje de entrada**, que en Foundry no tiene esa restricción:

```python
# load_receipt_image deja de ser @tool: es una función normal
receipt_image = load_receipt_image("receipt.jpg")     # devuelve Content.from_data(...)
message = Message("user", [prompt_text, receipt_image])
events = workflow.run(message, stream=True)
```

El `ocr_agent` pierde `tools=[load_receipt_image]` y sus instrucciones pasan a decir que la imagen viene en el mensaje. Verificado localmente: `Message("user", [texto, Content.from_data(...)])` construye dos contenidos, uno `text` y uno `data`, que es lo que la API necesita como `input_text` + `input_image`.

**Reiniciar el kernel** antes de re-ejecutar: cambian imports y la firma de la función, y la conversación de Foundry puede haber quedado en estado inconsistente por los intentos previos.

## La lección

Leer el código fuente del paquete instalado resolvió en un paso lo que dos iteraciones de prueba y error no habían resuelto. `SUPPORTS_RICH_FUNCTION_OUTPUT = False` es una constante de clase que no aparece en ninguna documentación ni mensaje de error — solo en el fuente.

Fuentes: [[10-ai-agents-production]]
