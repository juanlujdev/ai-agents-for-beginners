---
type: log
date_updated: 2026-08-01
---

# Log

Append-only. Formato de cabecera fijo para poder filtrar:
`grep "^## \[" wiki/log.md | tail -5`

## [2026-08-07] ingest | Fix de imports desactualizados en hotel_booking_workflow_sample.py

`ChatMessage`, `ai_function` y `Role.USER` no existen en el `agent_framework` instalado (verificado con imports reales, no con documentación): son `Message`, `tool` y `role="user"` con `contents=[...]`. Corregido en el archivo. Cuarta aparición del mismo patrón de bug ya visto en [[fix-notebook-04-sdk-desactualizado]] — nueva página [[fix-hotel-booking-sample-imports]]. Corrige además una conclusión previa en [[14-microsoft-agent-framework]] que había calificado la mención a `ai_function` como "desfase de redacción" sin bug real: en este script sí lo es.

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

## [2026-08-02] ingest | 14-sequential.ipynb (explicación celda a celda + fix de structured outputs)

**Fuente**: `14-microsoft-agent-framework/code-samples/14-sequential.ipynb`, recorrido celda a celda en sesión didáctica (`78c7fe19`), más su ejecución real contra Foundry.

El único de los cinco notebooks de la lección 14 que no tenía sección propia en la wiki.

**Fix aplicado al notebook** (queda en el repo, no solo en la wiki): los dos agentes pasan ahora `default_options={"response_format": ...}` y se elimina de `instructions` la petición redundante `"Return structured JSON matching the X schema"`. Sin ello, el modelo devolvía `{'name': 'Vasa Museum'...}` y Pydantic daba `4 validation errors ... Field required`.

**Tercera aparición del mismo bug** (`04`, `14-concurrent`, `14-sequential`). Se pasó de "pasó dos veces" a una afirmación general en [[structured-outputs]]: *si un notebook de este curso pide JSON solo por prompt, va a fallar*. Añadida al índice como pendiente el inventario de notebooks aún sin `response_format`.

**Conocimiento nuevo propagado**:
- [[structured-outputs]] — firma reconocible del error (acierta el contenido, se inventa los rótulos) y sección nueva **"el prompt y el esquema tienen que estar de acuerdo"**: `AttractionRecommendation` tiene un solo `attraction_name`, luego el prompt necesita la palabra *single*. Corolario: al tocar un esquema, releer el prompt.
- [[workflows-como-grafo]] — tabla de las cuatro decisiones de `WorkflowBuilder`; sección del patrón secuencial; y el matiz de que la fragilidad de leer `outputs[i]` por posición está **dormida** en secuencial (el orden de llegada coincide con el declarado), no ausente.
- [[llm-as-judge]] — tercer rol del patrón en el curso: juez como **nodo de un workflow**. Argumento nuevo de por qué el juez debe ser otro agente aunque comparta modelo (sin conflicto de intereses + una instrucción corta se cumple mejor). Y la advertencia de que un juez sin tools no verifica nada: `visitor_rating: 4.6` es estimación con aspecto de dato.

**Debilidades del notebook documentadas** sin corregir: `output_executors` deprecado, `analyze_sequential_flow()` re-ejecuta el workflow en vez de leer `events` (dos llamadas de más), resumen final escrito a mano en el HTML, `# 1-10 scale` como comentario en vez de `Field(ge=1, le=10)`, numeración de pasos que salta del 4 al 8, cinco imports huérfanos.

## [2026-08-02] ingest | Verificación del fix de 14-sequential.ipynb

El fix de `response_format` de la entrada anterior queda **verificado end-to-end** contra Foundry real. Notebook completo re-ejecutado, cero salidas de tipo `error`:

```
Stockholm: front-desk -> Vasa Museum (Vasamuseet) | concierge -> 9/10, 4.5/5.0
Barcelona: user -> front-desk (JSON) -> concierge (JSON) | Total Steps: 3
```

Los campos llegan con **el nombre exacto del esquema** (`attraction_name`, antes `name`) y el JSON del recepcionista viaja sin envolver. Confirma que la restricción dura actúa, y no que el modelo obedeciera el prompt por casualidad.

Retirado el aviso "sin verificar" de [[14-microsoft-agent-framework]]. El pendiente del inventario de notebooks sin `response_format` sigue abierto.
