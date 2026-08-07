---
type: entity
date_updated: 2026-08-07
source_count: 1
---

# Qwen (modelos de function calling)

Familia de [[slm|SLMs]] entrenados específicamente para producir **tool calls bien formadas de forma consistente** — a diferencia de muchos SLMs que conversan bien pero generan llamadas a función malformadas o inconsistentes. Es lo que convierte un modelo de chat local en un **agente** local de verdad: sin function calling fiable, [[tool-calling]] no funciona y el agente solo puede hablar, no actuar.

Servido en el curso a través de [[foundry-local]] (`foundry model run qwen2.5-7b-instruct`), consumido con el cliente OpenAI estándar sin cambios de código respecto a un modelo de nube.

Fuentes: [[17-creating-local-ai-agents]]
