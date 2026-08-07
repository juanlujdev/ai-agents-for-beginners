---
type: concept
date_updated: 2026-08-07
source_count: 1
---

# Small Language Models (SLMs)

Modelos de pocos miles de millones de parámetros, dimensionados para correr en una máquina de desarrollo (CPU, GPU o NPU) en vez de un datacenter. Contrapartida "de bolsillo" de un modelo de frontera en la nube.

## Fuertes en

- Tareas acotadas: clasificación, extracción, resumen de un documento conocido.
- **Tool calling**: decidir qué función llamar y con qué argumentos.
- Iteración rápida, barata y privada sobre datos propios.

## Débiles en

- Razonamiento abierto multi-paso sobre contexto amplio.
- Conocimiento general del mundo (menos visto en entrenamiento, se les "olvida" más).

## La regla de diseño que se deriva

**El SLM orquesta, las tools cargan con el peso.** No hace falta que el modelo "sepa" el codebase — hace falta que sepa cuándo llamar a `read_file` o `search_docs`. Es la misma división de responsabilidades que [[tool-calling]] ya establece en general (el LLM decide, el framework ejecuta), pero aquí es la única forma de que un modelo pequeño rinda: compensar su debilidad (conocimiento, razonamiento largo) apoyándose en su fortaleza (decisiones acotadas).

Los motivadores prácticos de elegir un SLM local sobre un modelo de nube son **privacidad** (los datos no salen de la máquina), **coste** (sin facturación por token) y **funcionamiento offline** — casos frecuentes en trabajo de ingeniería real: código sensible, cumplimiento normativo, o trabajo sin red.

Fuentes: [[17-creating-local-ai-agents]]
