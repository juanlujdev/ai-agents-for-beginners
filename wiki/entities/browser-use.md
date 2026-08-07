---
type: entity
date_updated: 2026-08-07
source_count: 1
---

# Browser-Use

Framework de automatización de navegador dirigida por IA. Envuelve Playwright: en vez de programar selectores CSS, se le da al `Agent` una tarea en lenguaje natural y decide él mismo qué elementos tocar, apoyándose en visión (capturas de pantalla).

## Piezas usadas en el curso

- **`Agent(task=..., llm=..., browser=..., use_vision=True)`** — ejecuta una tarea de navegación abierta. `use_vision=True` es lo que le permite "leer" la página en vez de depender solo del DOM.
- **`Browser`** — envoltorio sobre el navegador real. Dos formas de conectarlo, vistas ambas en [[15-browser-use]]:
  - `Browser(playwright_browser=...)` — a partir de un objeto Playwright ya creado.
  - `Browser(cdp_url=..., keep_alive=True)` — conexión directa por Chrome DevTools Protocol; `keep_alive=True` evita que el navegador se cierre al terminar el `Agent`, necesario si luego se quiere seguir usando la misma página con control directo.
- **`page.extract_content(prompt=..., structured_output=ModeloPydantic, llm=...)`** — el modo "actor": lee la página actual y fuerza la salida al esquema Pydantic dado. Es la vía de [[structured-outputs]] específica de Browser-Use, aplicada a contenido visual en vez de texto.
- **`ChatAzureOpenAI`** — el propio paquete `browser_use` trae su cliente de Azure OpenAI; no es el de `langchain_openai`. En el notebook de la lección 15 se instala `langchain-openai` pero no se usa — dependencia muerta.

## Patrón de uso: Agente para explorar, Actor para extraer

La forma recomendada de usar el framework, según el curso: `Agent` para la parte impredecible del flujo (popups, navegación, búsqueda), y `page.extract_content` (control directo) para la parte estructurada (leer datos). Ver la discusión completa del porqué en [[computer-use-agents]].

## Detalle sin resolver

`page.screenshot()` parece devolver tipos distintos según cómo se conectó el `Browser` (`playwright_browser` vs `cdp_url`): en un caso el código del notebook lo trata como string base64 ya listo para mostrar, en el otro lo trata como bytes crudos y falla al reintentar codificarlo. No investigado a fondo, solo observado en la ejecución real → [[15-browser-use]].

Fuentes: [[15-browser-use]]
