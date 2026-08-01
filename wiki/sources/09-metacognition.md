---
type: source-summary
date_updated: 2026-08-01
leccion: 09-metacognition
sesiones: [04f4b72d, 515207c2, 897c171a, f496adff]
fechas_origen: 2026-07-18
---

# 09 — Metacognición

La lección con la discusión conceptual más profunda de todo el historial. Ver el concepto en [[metacognicion]].

## La distinción que ordena toda la lección

Metacognición **no** es corregir el resultado, es corregir **la estrategia que produjo el resultado**.

Analogía: un estudiante que tacha la respuesta mala y pone la correcta hace corrección de errores. El que se pregunta *"¿por qué me equivoqué? Siempre fallo cuando asumo X sin comprobarlo"* y cambia su método de estudio hace metacognición.

El README lo dice así: *"Cada vez que el usuario dice 'demasiado concurrido', no solo debo quitar esas atracciones, sino reconocer que mi método de rankear por popularidad está mal."*

## Qué NO es metacognición (aunque lo parezca)

El primer ejemplo del README, `Travel_Agent`, **no es metacognición todavía**:

```python
def adjust_based_on_feedback(self, feedback):
    self.experience_data.append(feedback)
    self.user_preferences = adjust_preferences(self.user_preferences, feedback)
```

Ajusta **datos de entrada** (`preferences`) con el feedback. Es aprendizaje por refuerzo básico. Sigue siendo valioso: `experience_data` es lo que le da memoria — sin acumulador no hay nada sobre lo que reflexionar, sería un chatbot sin estado.

## `HotelRecommendationAgent` — el ejemplo que sí lo es

Tres hoteles con **precio y calidad en tensión** (ninguno gana en ambas), lo que obliga a tener una estrategia.

```python
def reflect_on_choice(self):
    last_choice_strategy, last_choice = self.previous_choices[-1]
    user_feedback = self.get_user_feedback(last_choice)
    if user_feedback == "bad":
        new_strategy = 'highest_quality' if last_choice_strategy == 'cheapest' else 'cheapest'
        self.corrected_choices.append((new_strategy, last_choice))
```

Lo que se modifica es **el algoritmo de selección** (`strategy`), no los datos. No dice "prueba otro hotel", dice "prueba otra *forma de elegir* hoteles".

El bucle mínimo de cualquier sistema metacognitivo:

```
DECIDIR ──▶ REGISTRAR ──▶ REFLEXIONAR
(recommend)  (previous_choices)  (reflect_on_choice)
   ▲                                  │
   └──── nueva estrategia ◀── ¿feedback malo?
```

`self.previous_choices.append(...)` dentro de `recommend_hotel` es la línea que hace posible todo lo demás.

## Tres malentendidos naturales sobre este código

Corregidos al contrastar el modelo mental con el código real:

| Lo que uno asume | Lo que hace el código |
| --- | --- |
| El usuario da feedback y se guarda | `get_user_feedback` **lo inventa** con una regla fija (`price < 100 or quality < 7`). No existe ninguna variable con la opinión real del usuario. |
| La reflexión ocurre "días después", al pedir otra recomendación | Se llama a mano justo después de la primera recomendación, en el mismo flujo. |
| El agente revisa su historial de elecciones | Solo mira `previous_choices[-1]`. Si pides 3 recomendaciones antes de reflexionar, las dos primeras nunca se revisan. |

Detalle sutil: `reflect_on_choice()` **solo devuelve un string** con la nueva estrategia. Es el código llamador quien vuelve a invocar `recommend_hotel` a mano. En un agente real ese paso estaría automatizado.

## Corrective RAG y contexto

- **RAG vs Pre-emptive Context Load**: uno busca en el momento de la consulta (datos frescos), el otro precarga en memoria (rápido, sin llamada externa). No son lo mismo.
- **Corrective RAG** = RAG + ciclo de autocorrección: generar query → evaluar relevancia de lo recuperado → refinar con el feedback y volver a buscar.
  ```python
  if "disliked" in feedback:
      preferences["avoid"] = feedback["disliked"]
  ```
  Convierte feedback humano en restricción explícita para la siguiente búsqueda.
- **RAG como prompt vs como tool**: prompt = control manual de cada query; tool = integrado en la arquitectura, automático y escalable.
- **Scoring heurístico vs con LLM**: `relevance_score` con `if`s es rápido y determinista pero rígido; el LLM re-rankea con flexibilidad semántica ("cultura diversa" no se captura con `if`s) a cambio de coste y predictibilidad.

## Riesgos de seguridad que el README simplifica

Dos patrones que se presentan sin advertencia:

- `exec()` sobre código generado por un LLM → ejecución de código arbitrario. En producción, sandbox aislado.
- `generate_sql_query` concatena valores con f-strings → **inyección SQL**. Lo correcto son queries parametrizadas.
- `identify_intent` clasifica intención con substring matching (`if "book" in query`), no NLP. Ejemplo pedagógico, no producción.

## El notebook `09-python-agent-framework.ipynb`

Aplica la metacognición con [[microsoft-agent-framework]] en dos patrones:

1. **Fallback de herramientas**: `get_flight_times` (primario, lanza 404 si no cubre la ciudad) + `get_flight_times_backup`. Las instrucciones del agente le dicen que detecte el fallo, cambie al backup y **sea transparente** sobre el cambio. El agente detecta su propio fallo en vez de romperse o inventar datos.
2. **Auto-evaluación**: un segundo agente `ResponseEvaluator` puntúa la respuesta del primero (completitud, precisión, utilidad, 1-5) más una sugerencia. Es [[llm-as-judge]] aplicado al propio sistema.

Ver también: [[10-ai-agents-production]]
