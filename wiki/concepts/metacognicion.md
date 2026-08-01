---
type: concept
date_updated: 2026-08-01
source_count: 2
---

# Metacognición

Que el agente **observe cómo decide** y ajuste el método de decisión, no solo el resultado.

## La línea divisoria

| | Corrección de errores | Metacognición |
| --- | --- | --- |
| Qué cambia | El dato / el resultado | **La estrategia que produjo el resultado** |
| Ejemplo | "Prueba otro hotel" | "Prueba otra *forma de elegir* hoteles" |
| En código | `preferences["avoid"] = ...` | `strategy = 'highest_quality'` |

El estudiante que tacha la respuesta mala y pone la correcta corrige errores. El que se pregunta *"¿por qué me equivoqué? Siempre fallo cuando asumo X"* y cambia su método de estudio hace metacognición.

## El bucle mínimo

```
DECIDIR ──▶ REGISTRAR ──▶ REFLEXIONAR
   ▲                            │
   └───── nueva estrategia ◀────┘
```

**Registrar es la pieza imprescindible.** Sin acumulador de decisiones pasadas (`previous_choices`, `experience_data`) no hay nada sobre lo que reflexionar: sería reacción, no metacognición.

## Cómo aparece en la práctica

- **Fallback de herramientas**: tool primaria que falla → el agente lo detecta, cambia al backup y **es transparente** sobre el cambio, en vez de romperse o inventar datos → [[tool-calling]].
- **Auto-evaluación**: un segundo agente puntúa la respuesta del primero → [[llm-as-judge]].
- **Corrective RAG**: RAG + ciclo de autocorrección. El feedback se convierte en restricción explícita para la siguiente búsqueda (`preferences["avoid"] = feedback["disliked"]`).

## Límites de los ejemplos del curso

El `HotelRecommendationAgent` de [[09-metacognition]] solo alterna entre dos estrategias fijas y **solo mira la última decisión** (`previous_choices[-1]`). Un agente metacognitivo robusto miraría patrones en todo el historial ("cheapest falla el 80% de las veces"), no reaccionaría a un solo dato.

Fuentes: [[09-metacognition]]
