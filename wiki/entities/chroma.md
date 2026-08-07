---
type: entity
date_updated: 2026-08-07
source_count: 1
---

# Chroma

Base de datos vectorial **embebida**: corre en el mismo proceso, sin servidor aparte que levantar ni gestionar — comparable a SQLite, pero para vectores. Usada en el curso para RAG local: documentos → embeddings locales → Chroma en disco → retrieval local → [[slm|SLM]] local, sin que ningún componente del pipeline toque la nube.

Mismo patrón de Agentic RAG que la Lección 5 (sin ingerir todavía) — la única diferencia respecto a esa lección es que aquí cada pieza del pipeline corre en local.

## Uso mínimo (verificado en el notebook de la 17)

```python
chroma_client = chromadb.Client()
collection = chroma_client.get_or_create_collection("project_docs")
collection.upsert(ids=list(DOCS.keys()), documents=list(DOCS.values()))

results = collection.query(query_texts=[query], n_results=2)
```

`upsert` calcula el embedding de cada documento **automáticamente** con un modelo de embeddings local que Chroma trae de fábrica — no hace falta llamar a ningún servicio externo ni elegir un modelo de embeddings aparte para que la vectorización siga siendo local de punta a punta. `query` busca por similitud de significado, no por coincidencia literal de palabras: una consulta sin ninguna palabra en común con el documento puede seguir siendo el resultado más relevante.

Fuentes: [[17-creating-local-ai-agents]]
