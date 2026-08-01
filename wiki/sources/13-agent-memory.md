---
type: source-summary
date_updated: 2026-08-01
leccion: 13-agent-memory
sesiones: [555e979a]
fechas_origen: 2026-07-27
---

# 13 — Memoria de agentes

Un agente sin memoria es **stateless**: cada conversación empieza de cero. La diferencia entre un camarero que cada día te pregunta qué quieres y uno que ya sabe que odias la cebolla.

## Los 7 tipos de memoria

| Tipo | Dura | Ejemplo |
| --- | --- | --- |
| **Working** | El paso actual del razonamiento | El bloc de notas mientras resuelve la tarea |
| **Short-term** | Una sesión | "¿Y el alojamiento *allí*?" — sabe que "allí" = París |
| **Long-term** | Entre conversaciones, siempre | Recuerda semanas después que evitas pistas avanzadas por una lesión |
| **Persona** | Permanente, define quién es | Un agente "experto en esquí" mantiene ese rol y tono |
| **Episodic** | Episodios completos, éxitos y fallos | Recuerda que una reserva falló por disponibilidad y ofrece alternativas |
| **Entity** | Entidades concretas | Extrae "París", "Torre Eiffel", "Le Chat Noir" y puede reservar de nuevo ahí |
| **Structured RAG** | — | Extrae datos exactos de un email (destino, fecha, aerolínea) en vez de buscar por similitud |

**Corte clave**: *working* y *short-term* se pierden al terminar la sesión; *long-term*, *persona*, *episodic* y *entity* sobreviven. Structured RAG es una **técnica**, no un tipo de memoria.

## Implementación

- `agent.create_session()` de [[microsoft-agent-framework]] es memoria de **corto plazo** integrada: mantiene el contexto mientras la sesión vive, y **se pierde si la app se reinicia**.
- Para sobrevivir a eso hace falta un almacén persistente → memoria de largo plazo.

Herramientas de largo plazo:

- **Mem0** — pipeline de dos fases: **extracción** (un LLM resume y saca hechos nuevos) y **actualización** (otro paso decide si la memoria se añade, modifica o elimina). Almacén híbrido: vectorial + grafo + clave-valor.
- **[[cognee]]** — convierte los datos en un **grafo de conocimiento** con embeddings. Doble almacén: similitud vectorial + relaciones de grafo, así el agente entiende no solo qué se parece a qué, sino cómo se relacionan los conceptos.
- **Azure AI Search** — backend para Structured RAG.

Notebooks: `13-agent-memory.ipynb` (Mem0 + Azure AI Search) y `13-agent-memory-cognee.ipynb` (Cognee con grafos).

## El patrón "knowledge agent"

Un segundo agente escuchando en silencio la conversación:

1. **Identifica** si algo merece guardarse.
2. **Extrae y resume** lo esencial, no la conversación entera.
3. **Guarda** en una base de conocimiento.
4. **Aumenta** las consultas futuras recuperando lo guardado y añadiéndolo al prompt del agente principal.

Es RAG, pero con memoria personal del usuario en vez de documentos genéricos. Hoy dices "odio las escalas"; tres semanas después pides un vuelo a Roma y ya filtra por directos.

Optimizaciones: un modelo barato y rápido para **decidir** si algo merece guardarse, reservando el caro para extraer en profundidad; y mover lo poco usado a *cold storage* para contener el coste.

## Cognee vs MAF — no compiten

La confusión natural, resuelta:

- **MAF** es el framework que **construye y ejecuta el agente**: conversa, decide qué tool llamar, mantiene la sesión.
- **Cognee** es un **sistema de memoria externo** que el agente consulta como una tool más.

MAF es el camarero; Cognee es la libreta de notas conectada con hilos que el camarero consulta.

```python
# CAPA MAF
coding_agent = provider.as_agent(name="CodingAssistant", instructions="...")
session = coding_agent.create_session()          # ← working memory (MAF)

# CAPA COGNEE
@tool(approval_mode="never_require")
async def search_knowledge(query):
    results = await cognee.search(query_text=query,
                                  query_type=SearchType.GRAPH_COMPLETION)
    return str(results)
```

Cognee no sabe nada de agentes ni de conversaciones: solo responde a `cognee.search(query)`. Cada uno funciona sin el otro — MAF con un simple `dict` como almacén (que es lo que hace el primer notebook), y Cognee como base de conocimiento sin agente encima. Sin MAF tendrías la enciclopedia sin el bibliotecario.

Ver también: [[12-context-engineering]], [[14-microsoft-agent-framework]]
