---
type: log
date_updated: 2026-08-01
---

# Log

Append-only. Formato de cabecera fijo para poder filtrar:
`grep "^## \[" wiki/log.md | tail -5`

## [2026-08-01] setup | Esqueleto de la wiki creado

Estructura inicial: `index.md`, `overview.md`, `sources/`, `entities/`, `concepts/`, `synthesis/`, `raw/`.
Fuentes definidas: lecciones del curso (`01-…/18-…`, README + notebooks) más lo que se deje en `raw/`.
Alcance: conceptos de AI agents + referencia de código del repo.

## [2026-08-01] ingest | Historial de conversaciones con Claude Code (25 sesiones)

**Fuente**: transcripciones de `~/.claude/projects/c--Users-lujan-Proyectos-Cursos-ai-agents-for-beginners/*.jsonl`, del 2026-07-03 al 2026-07-31. 25 sesiones útiles, ~755 K caracteres de diálogo tras descartar tool calls y resultados.

**Decisiones de ingesta**:
- Granularidad: una página de `sources/` **por lección** (no por sesión), más los fixes en `synthesis/`.
- Datos del entorno **anonimizados** (`<TU_RECURSO>`, `<TU_PROYECTO>`); las rutas locales absolutas se sustituyen por rutas del repo.
- Las transcripciones **no se copian a `raw/`** (contienen `.env`, endpoints y rutas locales). Se citan por `sessionId` en el frontmatter de cada página; los `.jsonl` son inmutables de facto en `~/.claude/`.

**Páginas creadas** (33):
- `sources/` (10): lecciones 00, 02, 03, 08, 09, 10, 11, 12, 13, 14.
- `entities/` (6): microsoft-agent-framework, azure-ai-foundry, mcp, a2a, nlweb, cognee.
- `concepts/` (7): tool-calling, structured-outputs, workflows-como-grafo, metacognicion, context-engineering, llm-as-judge, autenticacion-azure-sin-claves.
- `synthesis/` (9): claude-vs-openai-en-foundry, fix-notebook-04-sdk-desactualizado, fix-workflow-concurrente, fix-reasoning-item-workflow, fix-imagenes-en-foundry, fix-modelos-deprecados-azure, fix-pip-notebook-colgado, fix-az-login-cache-msal, fix-git-push-divergente.
- `index.md` y `overview.md` reescritos; tesis inicial formulada.

**Conocimiento propagado entre lecciones**: el error `function_call` sin `reasoning` item quedó **sin resolver** en la lección 10 y se resolvió en la 14 (`store=True` + `context_mode="custom"` con `context_filter`). Ambas páginas se enlazaron y `fix-reasoning-item-workflow` pasó de "problema abierto" a fix verificado.

**Lecciones sin ingerir**: 01, 04, 05, 06, 07, 15, 18 — no aparecen en el historial o solo de pasada. Anotado en Pendientes.

## [2026-08-01] lint | 0 wikilinks rotos, 0 huérfanas

Comprobados los 33 ficheros: todos los enlaces wiki resuelven y todas las páginas tienen al menos un enlace entrante (salvo `overview.md`, que es la entrada por diseño y ahora se enlaza desde `index.md`).
6 pendientes registrados en el índice, entre ellos la confusión de entornos Python (`.venv` vs `Python311`/`Python312` globales) y el `SyntaxError` de `14-handoff.ipynb`, cuyo estado en el archivo está sin confirmar.
