---
name: llm-wiki
description: Mantiene la wiki markdown persistente del proyecto (patrón LLM Wiki de Andrej Karpathy) en wiki/. Usar cuando el usuario pida ingerir una fuente nueva en la wiki, preguntar algo que la wiki deba responder, o revisar la salud de la wiki (lint). Trigger: "ingesta esto en la wiki", "qué sabemos sobre X", "revisa la wiki", "/llm-wiki".
---

# LLM Wiki

Mantiene `wiki/` como base de conocimiento del proyecto. Ver `CLAUDE.md` sección "LLM Wiki" para la arquitectura completa y las convenciones de frontmatter. Nunca modificar los archivos de `raw/` — son fuentes de solo lectura.

## Operación: ingest

Se invoca con una fuente: un archivo de `raw/`, una URL, o texto pegado por el usuario.

1. Leer la fuente completa.
2. Crear o actualizar la página en `wiki/sources/<slug-de-la-fuente>.md` con frontmatter `type: source-summary`, `date_updated: <hoy>`, y un resumen factual con `[[wikilinks]]` a entidades/conceptos relacionados.
3. Crear o actualizar páginas en `wiki/entities/` y `wiki/concepts/` si la fuente introduce o amplía una entidad/concepto (frontmatter `type: entity` o `type: concept`, más `source_count: <n>` = número de páginas de `sources/` que enlazan a esta página).
4. Actualizar `wiki/index.md`: añadir fila en la tabla correspondiente, quitar la fuente de "Pendientes" si estaba listada.
5. Actualizar `wiki/overview.md` si la fuente cambia el panorama general del proyecto.
6. Añadir entrada a `wiki/log.md`: `## [YYYY-MM-DD] ingest | <título de la fuente>`.

### Ingesta sin fuente escrita (aviso proactivo)

No todo lo importante llega a escribirse en un documento — hay decisiones o fixes no triviales que se resuelven en una conversación y nunca generan un archivo que dispare el `ingest` normal.

Cuando en una conversación se resuelva algo así de importante y no se vaya a documentar en ningún sitio, ofrecer proactivamente ingerirlo igualmente: "esto es importante y no va a quedar documentado, ¿lo ingiero en la wiki?". Si el usuario acepta, tratar el resumen de la propia conversación como la fuente (el "texto pegado por el usuario" de arriba) y seguir el mismo algoritmo de `ingest`.

No ofrecerlo para fixes triviales del día a día — solo cuando, de no ingerirlo, se perdería contexto que una sesión futura echaría en falta.

## Operación: query

Se invoca con una pregunta.

1. Leer `wiki/index.md` primero para localizar páginas relevantes.
2. Sintetizar la respuesta leyendo solo `wiki/` (nunca las fuentes crudas directamente — si la wiki no tiene la respuesta, ejecutar `ingest` primero sobre la fuente relevante).
3. Citar las páginas de `wiki/` usadas con `[[wikilinks]]`.
4. Si la respuesta es sustancial (no una frase suelta), archivarla en `wiki/synthesis/<slug-de-la-pregunta>.md` con frontmatter `type: synthesis`.
5. Añadir entrada a `wiki/log.md`: `## [YYYY-MM-DD] query | <pregunta>`.

## Operación: lint

Sin argumentos. Revisa:

1. **Huérfanas:** páginas en `wiki/` sin ningún `[[wikilink]]` entrante desde otra página.
2. **Wikilinks rotos:** `[[Página]]` que no corresponde a ningún archivo existente.
3. **Desactualizadas:** `date_updated` en frontmatter con más de 90 días de antigüedad respecto a la fecha actual.
4. **Conceptos sin página:** términos mencionados en 2+ páginas de `sources/` que no tienen página propia en `concepts/`.
5. **Contradicciones:** afirmaciones incompatibles entre páginas sobre el mismo tema.

Reportar hallazgos como lista; no corregir automáticamente salvo que el usuario lo pida explícitamente. Añadir entrada a `wiki/log.md`: `## [YYYY-MM-DD] lint | <n> hallazgos`.
