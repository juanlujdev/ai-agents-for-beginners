---
type: source-summary
date_updated: 2026-08-01
leccion: 10-ai-agents-production
sesiones: [28dd2ff0, 702d3b0d]
fechas_origen: 2026-07-20
---

# 10 — Agentes en producción

Observabilidad, evaluación y control de coste. Analogía que la ordena: un agente es un empleado al que nunca ves trabajar, solo te entrega el resultado. Esta lección trata de ponerle cámaras.

## Observabilidad

- **Trace** = la tarea completa. **Span** = cada paso dentro (llamada al LLM, consulta a BD, uso de tool). El trace es el ticket de compra; cada span, un producto escaneado.
- Sin trazas el agente es una caja negra; con ellas, de cristal.
- El estándar es **OpenTelemetry**, ya integrado en [[microsoft-agent-framework]] (`from agent_framework.observability import get_tracer, get_meter`).
- El notebook `10-python-agent-framework.ipynb` hace la versión mínima: `time.time()` antes y después de `agent.run(...)`. Es el concepto, no la implementación productiva.

**Métricas a vigilar**: latencia, coste por ejecución, errores de request, feedback explícito (👍👎) e **implícito** (el usuario repite la pregunta o le da a reintentar — señal de fallo aunque no se queje), precisión, y evaluación automatizada.

## Evaluación: offline vs online

- **Offline** = examen con respuestas conocidas, en desarrollo/CI-CD. Repetible.
- **Online** = tráfico real en producción. Detecta lo que el examen no anticipó.

No son alternativas, se retroalimentan:

```
evaluar offline → desplegar → monitorear online →
recolectar fallos nuevos → añadirlos al dataset offline → refinar → repetir
```

El patrón [[llm-as-judge]] aparece aquí como `ResponseEvaluator`: un agente sin herramientas que puntúa la respuesta del principal en completitud, precisión, utilidad y score global. Sirve de *quality gate*, para detectar regresiones al cambiar modelo o prompt, y para monitorización continua.

## Control de coste

Modelos pequeños (SLM) para tareas simples · **router model** barato que decide a qué modelo va cada petición · cache de respuestas · presupuestos con `max_tokens` · batching. La recomendación es **por niveles**, no "el modelo más potente para todo".

## Notebook `10-expense_claim-demo.ipynb`

Workflow secuencial de dos agentes: imagen del recibo → `OCRAgent` extrae `fecha|descripción|importe|categoría` → `EmailAgent` redacta el correo a Finanzas.

```python
workflow = WorkflowBuilder(start_executor=ocr_agent).add_edge(ocr_agent, email_agent).build()
```

Ideas que ilustra:

- Un agente especializado por tarea, no un agente todopoderoso.
- **Mínimo privilegio**: cada agente solo tiene la tool que necesita.
- El formato `date|description|amount|category` es un **contrato explícito** entre el agente 1 y el 2.
- Streaming token a token con `stream=True`.

Este notebook dio dos errores reales, uno de ellos no documentado en el propio curso → [[fix-imagenes-en-foundry]] y [[fix-reasoning-item-workflow]] (este segundo quedó sin resolver aquí; la solución llegó en [[14-microsoft-agent-framework]]).

## Observación sobre el entorno

En esta lección salió a la luz que había paquetes instalados tanto en el `.venv` del proyecto como en el Python global (`Python311`/`Python312`), y que el kernel de VS Code podía estar apuntando a cualquiera de ellos. `agent-framework` con `-U` arrastra 30+ subpaquetes (Anthropic, Bedrock, Cosmos, mem0ai, qdrant, redis, ollama…), así que la primera instalación tarda varios minutos por resolución de dependencias, no porque esté rota → [[fix-pip-notebook-colgado]].

Nota de PowerShell: `pip : NativeCommandError` al final de un `pip install` **no es un fallo** — es PowerShell 5.1 envolviendo la salida normal de pip como si fuera error.

Ver también: [[09-metacognition]], [[12-context-engineering]]
