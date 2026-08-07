---
type: concept
date_updated: 2026-08-07
source_count: 4
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

Aparece en tres sitios del curso con roles distintos: en [[09-metacognition]] como auto-evaluación (el sistema juzgándose a sí mismo, [[metacognicion]]), en [[10-ai-agents-production]] como métrica operativa, y en `14-sequential.ipynb` como **nodo de un workflow** — el `concierge-agent` que puntúa la recomendación del `front-desk-agent` → [[workflows-como-grafo]].

## Por qué el juez tiene que ser otro agente

El argumento que justifica el patrón, y que el caso secuencial deja más claro que la evaluación en producción:

**Los dos agentes pueden compartir modelo, deployment y factura.** Lo único que los separa es el `instructions`. Y aun así el resultado mejora, por dos motivos independientes:

1. **Sin conflicto de intereses.** Al juez nunca se le pide ser amable con lo que evalúa: no tiene lealtad hacia una propuesta que no es suya. Un único agente al que se le pide proponer *y* criticar acaba defendiendo su propio trabajo — se pone un 10 como el estudiante que corrige su propio examen.
2. **Una instrucción corta se cumple mejor que una larga.** *"Recomienda algo entusiasta pero sé crítico, pon nota, lista pros y contras y sugiere alternativas"* sale a medias. Dos encargos pequeños salen enteros. Es reparto de atención, no de conocimiento.

Corolario práctico: **partir un prompt largo en dos agentes es una mejora real aunque no cambies de modelo.** No hace falta un modelo mejor para el juez.

## Un juez sin tools no verifica nada

Trampa de lectura en `14-sequential.ipynb`: el conserje devuelve `popularity_score: 9` y `visitor_rating: 4.6` **sin haber consultado ninguna fuente**. Son estimaciones de su entrenamiento con aspecto de dato de TripAdvisor, y el esquema Pydantic las acepta porque están bien formadas.

La separación que hay que tener presente: Pydantic garantiza **la forma** (que sea un decimal entre comillas correctas), el modelo elige **el contenido** (4.6 y no 2.1), y **nadie garantiza que sea cierto**. Para eso haría falta una tool que consultara una API real → [[tool-calling]].

## Límites

Es un LLM juzgando a otro LLM: hereda sus sesgos y no es determinista. Sirve como señal, no como verdad. El curso lo presenta junto a alternativas — feedback explícito del usuario, feedback implícito (repetir la pregunta, reintentar) y librerías como RAGAS o LLM Guard.

Para evaluación seria, la pieza que falta aquí es el **dataset offline** con respuestas conocidas, ejecutado en CI/CD → [[10-ai-agents-production]].

## Cuarto rol: compuerta de release

En [[16-deploying-scalable-agents]] el patrón se vuelve una decisión de CI/CD, no solo una métrica que se observa: `score_response(...)` puntúa cada caso de un set offline y una `evaluation_gate()` **bloquea el despliegue** si la tasa de aciertos no supera un umbral (`0.8` en el ejemplo del curso). Es el mismo loop offline/online de [[10-ai-agents-production]] hecho explícito y automático — nadie despliega a mano "porque parece que funciona".

Fuentes: [[10-ai-agents-production]], [[09-metacognition]], [[14-microsoft-agent-framework]], [[16-deploying-scalable-agents]]
