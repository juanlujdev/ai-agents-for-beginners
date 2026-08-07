---
type: synthesis
date_updated: 2026-08-07
source_count: 1
---

# Fix: imports desactualizados en `hotel_booking_workflow_sample.py`

Cuarta aparición documentada de la misma familia de bugs que [[fix-notebook-04-sdk-desactualizado]]: código de la lección 14 escrito contra una versión anterior de `agent_framework`, que no importa con el SDK realmente instalado.

Archivo: `14-microsoft-agent-framework/code-samples/hotel_booking_workflow_sample.py` — el script standalone que implementa el mismo patrón de [[workflows-como-grafo]] condicional que `14-conditional-workflow.ipynb` (3 agentes + 1 executor sin IA, dos condiciones mutuamente excluyentes sobre `has_availability`).

## Tabla de arreglos

| # | Roto | Correcto en el SDK instalado | Verificado |
| --- | --- | --- | --- |
| 1 | `ChatMessage` | `Message` | `ImportError` al importar `ChatMessage` |
| 2 | `ai_function` | `tool` | `ImportError` al importar `ai_function` |
| 3 | `Role.USER` (con `text=...`) | `role="user"` (con `contents=[...]`) | `AttributeError: 'NewType' object has no attribute 'USER'` |

Los tres se probaron contra el `agent_framework` real instalado en el entorno del usuario (no contra documentación), igual que exige la entidad [[microsoft-agent-framework]].

## La corrección aplicada

```python
# imports
from agent_framework import (
    AgentExecutor, AgentExecutorRequest, AgentExecutorResponse,
    Message, WorkflowBuilder, WorkflowContext, executor, tool,
)

# decorador de la tool
@tool(description="Check hotel room availability for a destination city")
def hotel_booking(...): ...

# construcción del mensaje de usuario
Message(role="user", contents=["I want to book a hotel in Paris"])
```

## Contradice una entrada previa — documentado, no sobrescrito

[[14-microsoft-agent-framework]] (sección `14-conditional-workflow.ipynb`) registraba esto como *"discrepancia menor: el markdown menciona `@ai_function`, pero el código usa `@tool`. Es desfase de redacción, mismo comportamiento"*. Esa nota es correcta **para el notebook** (donde el código sí usa `@tool` y solo el texto markdown menciona el nombre viejo).

Pero en `hotel_booking_workflow_sample.py` — un archivo `.py` distinto, no el notebook — **el código en sí** importa `ai_function`, no solo lo menciona en prosa, y eso revienta con `ImportError` real. No es "mismo comportamiento": es el mismo bug de nombres que ya atrapó a `ChatMessage`/`Role.USER` en el notebook 04 de la lección 08, esta vez también arrastrando `ai_function`.

## Por qué importa como patrón, no solo como fix puntual

Van ya dos archivos de dos lecciones distintas (08 y 14) con el mismo trío de síntomas (`ChatMessage`, `Role.USER`, y ahora también `ai_function`/`tool`). Sugiere que el material del curso se generó o copió en un momento en que la API tenía esos nombres, y ninguno de los scripts standalone de `code-samples/` se ha vuelto a ejecutar contra el SDK actual desde entonces — a diferencia de los notebooks, que sí se han ido probando y corrigiendo uno a uno (ver [[fix-notebook-04-sdk-desactualizado]], [[fix-workflow-concurrente]], [[fix-reasoning-item-workflow]]).

Fuentes: [[14-microsoft-agent-framework]]
