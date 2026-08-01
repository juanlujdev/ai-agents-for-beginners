---
type: entity
date_updated: 2026-08-01
source_count: 1
---

# Cognee

Motor de memoria semántica / grafo de conocimiento. Convierte datos en un **grafo** respaldado por embeddings, con arquitectura de doble almacén: búsqueda por similitud vectorial **+** relaciones de grafo. El agente entiende no solo qué se parece a qué, sino cómo se relacionan los conceptos.

## API

```python
await cognee.add(datos)        # ingerir
await cognee.cognify()         # construir el grafo
await cognee.search(query_text=..., query_type=SearchType.GRAPH_COMPLETION)
```

## No compite con MAF

La confusión natural, resuelta en [[13-agent-memory]]: **[[microsoft-agent-framework]] es el camarero, Cognee es la libreta de notas conectada con hilos que el camarero consulta.**

- MAF construye y ejecuta el agente, mantiene la sesión (working memory).
- Cognee almacena la memoria de largo plazo, y el agente la consulta **como una tool más**.

Cognee no sabe nada de agentes ni de conversaciones: solo responde a `search(query)`. Cada uno funciona sin el otro — MAF puede usar un simple `dict` como almacén; Cognee sirve como base de conocimiento sin agente encima (la enciclopedia sin el bibliotecario).

## Alternativa

**Mem0** cubre el mismo hueco con otro enfoque: pipeline de dos fases (extracción con LLM → decisión de añadir/modificar/eliminar) sobre un almacén híbrido vectorial + grafo + clave-valor. MAF lo integra vía `context_providers=Mem0Provider(...)`.

Fuentes: [[13-agent-memory]]
