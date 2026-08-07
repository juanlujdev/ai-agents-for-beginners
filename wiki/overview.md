---
type: overview
date_updated: 2026-08-07
source_count: 13
---

# Overview

Wiki del curso *AI Agents for Beginners* (Microsoft), construida sobre las lecciones del repo y el historial de trabajo con ellas.

**Estado**: 13 lecciones ingeridas de 18. Cubre setup, frameworks, patrones de diseño, multi-agente, metacognición, producción, protocolos, context engineering, memoria, Microsoft Agent Framework, computer use agents (navegador), despliegue a escala y agentes locales. Ver [[index]] para el catálogo y [[log]] para la cronología.

## Tesis actual

Tres cosas que se repiten en todas las lecciones:

**1. La mayoría de los errores no están en el código de la lección, sino en el desajuste entre el código y lo que hay instalado (o configurado).** El curso se escribió contra una versión anterior del SDK y contra `gpt-4o`, y ambos se movieron. Nueve símbolos rotos en un solo notebook ([[fix-notebook-04-sdk-desactualizado]]), métodos renombrados, clases que pasaron a ser Protocolos, enums que pasaron a `NewType`. El método que funciona es verificar la existencia y la firma real de cada símbolo contra el paquete instalado **antes** de editar, en vez de cazarlos de uno en uno. El mismo patrón aparece en `.env`: un valor placeholder no vacío (`"https://..."`) pasa cualquier comprobación con `bool()` y activa silenciosamente el camino de código equivocado — [[fix-azure-search-placeholder-url]].

**2. La elección del modelo rompe cosas que parecen bugs de código.** Es el hallazgo transversal ([[claude-vs-openai-en-foundry]]): un deployment `claude-*` funciona para texto libre y falla en [[structured-outputs]] y en tool-calling con sesión; un modelo de razonamiento como `gpt-5-mini` rompe el traspaso de historial entre agentes ([[fix-reasoning-item-workflow]]). Ante un fallo raro, la familia del modelo es la primera sospechosa, no la última.

**3. Los límites reales están en constantes de clase que nadie documenta.** `SUPPORTS_RICH_FUNCTION_OUTPUT = False`, `STORES_BY_DEFAULT = False`, `get_outputs()` devolviendo por orden de llegada. Ninguno aparece en un mensaje de error ni en la documentación: solo en el fuente del paquete. Leer el código instalado ha resuelto en un paso lo que varias iteraciones de prueba y error no resolvían.

## Hilo conceptual

El curso avanza de "un agente con tools" a "varios agentes coordinados en producción". Las piezas encajan así:

- [[tool-calling]] es la unidad básica; [[structured-outputs]] convierte la salida del LLM en dato programable.
- [[workflows-como-grafo]] coordina varios agentes — y la decisión de ruta es **código Python**, no el LLM.
- [[context-engineering]] gestiona qué entra en la ventana **dentro** de una sesión; la memoria ([[cognee]], Mem0) es lo que persiste **entre** sesiones.
- [[metacognicion]] y [[llm-as-judge]] son las capas de auto-observación.
- [[mcp]], [[a2a]] y [[nlweb]] son la interoperabilidad hacia fuera.
- [[computer-use-agents]] es la variante donde el agente actúa sobre una interfaz visual (navegador) en vez de una API — mismo principio de [[structured-outputs]], aplicado a lo que el modelo "ve" en pantalla.
- [[patrones-de-despliegue]] cierra el arco de producción: lleva ese mismo agente de notebook a producción, convirtiendo la evaluación offline/online de [[10-ai-agents-production]] en una **compuerta de release** ([[llm-as-judge]]) y el `RequestInfoEvent` de [[workflows-como-grafo]] en un nodo de aprobación humana para acciones de negocio reales.
- [[17-creating-local-ai-agents|El agente local]] es la contrapartida de todo lo anterior: en vez de escalar hacia la nube, el mismo bucle tool-calling corre entero en la máquina con un [[slm]] servido por [[foundry-local]]. La pieza que lo hace posible es [[qwen]] (function calling fiable) y el mismo patrón de model routing de la 16 se extiende con un tercer eje — local vs. nube por sensibilidad y disponibilidad, no solo por complejidad.

Todo se implementa con [[microsoft-agent-framework]] sobre [[azure-ai-foundry]], salvo la lección 15 ([[browser-use]]), que añade Playwright/CDP como capa de control del navegador, y la lección 17, que sustituye la nube por [[foundry-local]].
