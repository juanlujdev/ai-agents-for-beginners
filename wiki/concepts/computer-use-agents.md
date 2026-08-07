---
type: concept
date_updated: 2026-08-07
source_count: 1
---

# Computer Use Agents (CUA)

Un agente que actúa sobre una **interfaz visual** — abre un navegador, mira la pantalla, decide el siguiente clic — en vez de llamar a una API. Existe porque muchos sitios (Airbnb, en el ejemplo del curso) no exponen una API pública: la única forma de conseguir el dato es actuar "como lo haría una persona".

## Agente vs Actor

La decisión de diseño central de cualquier CUA. No son alternativas excluyentes — son dos modos con costes distintos:

| | Agente (lenguaje natural) | Actor (control directo, selectores) |
| --- | --- | --- |
| Layout dinámico | Se adapta | Selectores frágiles, se rompe |
| Estructura conocida y estable | Más lento de lo necesario | Rápido y preciso |
| Encontrar un elemento por descripción | Funciona bien | Necesita selector exacto |
| Control fino de tiempos/reintentos | Menos predecible | Control total |
| Flujo con sorpresas (popups, banners) | Maneja estados inesperados | Requiere una rama de código por cada caso previsto |

**Regla práctica**: empezar con Agente para explorar un sitio nuevo o impredecible; cambiar a control directo en cuanto la interacción se vuelva predecible, porque un Agente que decide con un LLM en cada paso es más lento y más caro que un selector fijo.

## El patrón híbrido

En producción no se elige uno u otro: se combinan. En [[15-browser-use]], el Agente resuelve la parte abierta (navegar, cerrar popups, buscar), y luego el flujo cambia a control directo (`page.extract_content`) para la parte estructurada (leer precios). El primero es flexible pero imprevisible en tiempo/coste; el segundo es rápido pero frágil si el sitio cambia. Juntos cubren lo que ninguno cubre solo.

```python
# Agente: resuelve lo impredecible
await Agent(task="...", llm=llm, browser=browser, use_vision=True).run()

# Actor: extrae con esquema fijo, sin margen de interpretación
result = await page.extract_content(prompt="...", structured_output=MiEsquema, llm=llm)
```

## Extracción estructurada desde visión

La parte de "Actor" casi siempre termina en lo mismo que ya se vio en [[structured-outputs]]: forzar al LLM a devolver un objeto Pydantic validado en vez de texto libre. La diferencia en un CUA es **de dónde** viene la información — no de un prompt de texto, sino de lo que el modelo "ve" en una captura de pantalla (`use_vision=True` + `page.extract_content(structured_output=...)`). El principio es el mismo: sin el esquema forzado, el LLM devolvería prosa ("encontré varios apartamentos, el más barato ronda...") en vez de un `float` utilizable en código.

Una vez extraído el dato estructurado, la lógica que lo usa (comparar precios, ordenar, elegir el más barato) es **código Python normal, sin LLM** — el mismo principio de [[workflows-como-grafo]]: usar el modelo solo donde aporta (ver, decidir, interpretar) y código determinista para todo lo demás.

## Buenas prácticas observadas

- Esperas estratégicas (`asyncio.sleep(...)`) tras acciones que disparan cambios visibles de UI — extraer antes de que la página termine de renderizar da datos vacíos o parciales.
- Screenshots durante el desarrollo para depurar visualmente lo que "ve" el modelo.
- Diseñar para que el sitio cambie: popups y cambios de layout se dan por hecho, no por excepción.

## Project Opal: la misma idea en producción

Microsoft aplica el mismo concepto a escala empresarial con **Project Opal (Frontier)**, dentro de M365 Copilot: un CUA que opera sobre un Windows 365 Cloud PC, de forma asíncrona, con la identidad del usuario. Sirve como referencia de qué le falta a un CUA "de juguete" para ser confiable en producción:

- **Human-in-the-loop real**: se detiene ante credenciales o acciones ambiguas y pide confirmación explícita, con *Take Control*/*Return Control* — no solo "recuperarse de un timeout de LLM" como en el ejemplo del curso.
- **Aislamiento y auditoría**: Cloud PC aislado, solo navegador (bloqueado vía Intune), identidad del usuario como límite de permisos, log de cada acción.
- **Planifica antes de actuar y se autosupervisa** — el mismo principio que [[metacognicion|metacognición]]: observar la propia estrategia, no solo el resultado, y frenar si algo se ve sospechoso.
- **Skills reutilizables** entre conversaciones — la misma idea que las tools de [[tool-calling]], pero como instrucciones reutilizables en vez de funciones Python.

Detalle completo de la tabla de correspondencias en [[15-browser-use]]. Disponible solo en el programa Frontier early access con M365 Copilot; feature experimental.

## Aplicaciones reales del patrón

Monitoreo de precios de viajes, comparación de precios/disponibilidad en e-commerce, extracción estructurada de sitios dinámicos sin API, testing de UI asistido por visión, monitoreo de cambios en sitios web, relleno inteligente de formularios multi-paso.

Fuentes: [[15-browser-use]]
