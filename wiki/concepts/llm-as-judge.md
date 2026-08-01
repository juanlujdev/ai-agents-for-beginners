---
type: concept
date_updated: 2026-08-01
source_count: 2
---

# LLM-as-judge

Un segundo agente, **sin herramientas**, cuyo único trabajo es puntuar la respuesta del primero.

```python
evaluator = client.as_agent(
    name="ResponseEvaluator",
    instructions="""You evaluate travel agent responses on these criteria:
1. Completeness (1-5)   2. Accuracy (1-5)
3. Helpfulness (1-5)    4. Overall Score (1-5)
Provide scores and a brief explanation for each.""",
)
evaluation = await evaluator.run(f"Evaluate this response:\n\n{response}")
```

## Para qué sirve

- **Quality gate**: bloquear respuestas malas antes de mostrarlas al usuario.
- **Detectar regresiones** al cambiar de modelo o de prompt.
- **Monitorización continua** de la calidad en producción.

Aparece en dos sitios del curso con roles distintos: en [[09-metacognition]] como forma de auto-evaluación (el sistema juzgándose a sí mismo, [[metacognicion]]) y en [[10-ai-agents-production]] como métrica operativa.

## Límites

Es un LLM juzgando a otro LLM: hereda sus sesgos y no es determinista. Sirve como señal, no como verdad. El curso lo presenta junto a alternativas — feedback explícito del usuario, feedback implícito (repetir la pregunta, reintentar) y librerías como RAGAS o LLM Guard.

Para evaluación seria, la pieza que falta aquí es el **dataset offline** con respuestas conocidas, ejecutado en CI/CD → [[10-ai-agents-production]].

Fuentes: [[10-ai-agents-production]], [[09-metacognition]]
