---
type: overview
date_updated: 2026-08-01
source_count: 10
---

# Overview

Wiki del curso *AI Agents for Beginners* (Microsoft), construida sobre las lecciones del repo y el historial de trabajo con ellas.

**Estado**: 10 lecciones ingeridas de 18. Cubre setup, frameworks, patrones de diseño, multi-agente, metacognición, producción, protocolos, context engineering, memoria y Microsoft Agent Framework. Ver [[index]] para el catálogo y [[log]] para la cronología.

## Tesis actual

Tres cosas que se repiten en todas las lecciones:

**1. La mayoría de los errores no están en el código de la lección, sino en el desajuste entre el código y lo que hay instalado.** El curso se escribió contra una versión anterior del SDK y contra `gpt-4o`, y ambos se movieron. Nueve símbolos rotos en un solo notebook ([[fix-notebook-04-sdk-desactualizado]]), métodos renombrados, clases que pasaron a ser Protocolos, enums que pasaron a `NewType`. El método que funciona es verificar la existencia y la firma real de cada símbolo contra el paquete instalado **antes** de editar, en vez de cazarlos de uno en uno.

**2. La elección del modelo rompe cosas que parecen bugs de código.** Es el hallazgo transversal ([[claude-vs-openai-en-foundry]]): un deployment `claude-*` funciona para texto libre y falla en [[structured-outputs]] y en tool-calling con sesión; un modelo de razonamiento como `gpt-5-mini` rompe el traspaso de historial entre agentes ([[fix-reasoning-item-workflow]]). Ante un fallo raro, la familia del modelo es la primera sospechosa, no la última.

**3. Los límites reales están en constantes de clase que nadie documenta.** `SUPPORTS_RICH_FUNCTION_OUTPUT = False`, `STORES_BY_DEFAULT = False`, `get_outputs()` devolviendo por orden de llegada. Ninguno aparece en un mensaje de error ni en la documentación: solo en el fuente del paquete. Leer el código instalado ha resuelto en un paso lo que varias iteraciones de prueba y error no resolvían.

## Hilo conceptual

El curso avanza de "un agente con tools" a "varios agentes coordinados en producción". Las piezas encajan así:

- [[tool-calling]] es la unidad básica; [[structured-outputs]] convierte la salida del LLM en dato programable.
- [[workflows-como-grafo]] coordina varios agentes — y la decisión de ruta es **código Python**, no el LLM.
- [[context-engineering]] gestiona qué entra en la ventana **dentro** de una sesión; la memoria ([[cognee]], Mem0) es lo que persiste **entre** sesiones.
- [[metacognicion]] y [[llm-as-judge]] son las capas de auto-observación.
- [[mcp]], [[a2a]] y [[nlweb]] son la interoperabilidad hacia fuera.

Todo se implementa con [[microsoft-agent-framework]] sobre [[azure-ai-foundry]].
