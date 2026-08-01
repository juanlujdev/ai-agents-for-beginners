# CLAUDE.md

Fork del curso *AI Agents for Beginners* (Microsoft). Ver [AGENTS.md](AGENTS.md) para las convenciones del repo original.

## LLM Wiki

Base de conocimiento persistente sobre AI agents que **tú mantienes**, no el usuario. No es RAG: el conocimiento se compila una vez al ingerir y se mantiene al día, en vez de re-derivarse en cada pregunta.

### Arquitectura de 3 capas

1. **Fuentes (inmutables)** — las lecciones del curso (`01-…` … `18-…`: README y notebooks) y lo que haya en [raw/](raw/). Se leen, **nunca** se modifican. Son la verdad.
2. **Wiki** — [wiki/](wiki/), markdown generado por ti. Tú la escribes entera:
   - [wiki/index.md](wiki/index.md) — catálogo (Fuentes / Entidades / Conceptos / Síntesis / Pendientes)
   - [wiki/log.md](wiki/log.md) — cronológico, append-only
   - [wiki/overview.md](wiki/overview.md) — entrada y tesis actual
   - `wiki/sources/` resúmenes · `wiki/entities/` frameworks, SDKs, servicios · `wiki/concepts/` patrones e ideas · `wiki/synthesis/` comparativas y respuestas archivadas
3. **Esquema** — esta sección. Se co-evoluciona con el usuario cuando el flujo cambie.

Alcance: conceptos de AI agents **y** referencia de implementación del código del repo. Todo (`wiki/` y `raw/`) se versiona en git.

### Convenciones

Frontmatter YAML en toda página:

```yaml
---
type: source-summary | entity | concept | synthesis | overview | index | log
date_updated: YYYY-MM-DD
source_count: 3   # opcional, solo si agrega varias fuentes
---
```

- Enlaces wiki de Obsidian: `[[nombre-de-pagina]]`. Enlaza generosamente; un enlace a una página que aún no existe es válido — marca un pendiente.
- Nombres de archivo en kebab-case.
- Citar siempre la fuente: ruta del archivo o `[[pagina-fuente]]`.

### Reglas fundamentales

1. **Nunca modificar fuentes.** Ni las lecciones ni `raw/`. Solo lectura.
2. **Toda ingesta actualiza `index.md` y `log.md`.** Sin excepción. Entrada de log con prefijo `## [YYYY-MM-DD] ingest|query|lint | Título`.
3. **Las contradicciones se documentan, no se resuelven en silencio.** Si una fuente nueva choca con una página existente, deja ambas versiones con su cita y una nota explícita del conflicto; añádelo a Pendientes. No sobrescribas la afirmación vieja como si nunca hubiera existido.

### Operaciones

- **Ingest** — leer la fuente, comentar lo clave con el usuario, escribir/actualizar su página en `sources/`, propagar a las entidades y conceptos afectados, actualizar `index.md` y `log.md`. Una fuente puede tocar 10-15 páginas. Por defecto, de una en una.
- **Query** — leer `index.md`, drill-down a las páginas relevantes, responder con citas. Si la respuesta tiene valor duradero, archivarla en `synthesis/` y catalogarla.
- **Lint** — contradicciones, afirmaciones obsoletas, páginas huérfanas, conceptos citados sin página, cross-references faltantes. Reportar en Pendientes; proponer qué preguntar o buscar a continuación.
