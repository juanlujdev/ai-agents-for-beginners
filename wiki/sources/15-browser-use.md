---
type: source-summary
date_updated: 2026-08-07
leccion: 15-browser-use
fuente: [15-browser-use/README.md, 15-browser-use/15-browser-user.ipynb]
---

# 15 — Browser Use (Computer Use Agents)

Primera lección del curso donde el agente actúa sobre una **interfaz visual** en vez de una API: abre un navegador, mira la pantalla y decide el siguiente clic. Ver el concepto general en [[computer-use-agents]].

## El caso de uso: por qué un navegador y no una API

Airbnb no ofrece una API pública para buscar y leer precios. La única vía es "como lo haría una persona": abrir el sitio, escribir el destino, esperar los resultados y leer lo que aparece en pantalla. El notebook (`15-browser-user.ipynb`) construye exactamente eso: busca alojamientos en Estocolmo y devuelve el más barato como dato estructurado.

## Las cuatro piezas y su función

| Pieza | Rol |
| --- | --- |
| [[browser-use]] | El "cerebro": Agente que decide qué hacer viendo capturas de pantalla |
| Playwright + CDP | Las "manos": ciclo de vida del navegador, conexión estable |
| Azure OpenAI con visión | Lee las capturas y razona sobre ellas |
| Pydantic | Convierte lo leído en datos Python tipados y validados |

**CDP** (Chrome DevTools Protocol) es el detalle técnico que hace posible compartir sesión: Playwright y Browser-Use se conectan al **mismo** Chrome (`--remote-debugging-port=9222`), en vez de cada uno abrir su propio navegador aislado.

## Agente vs Actor — el patrón central de la lección

Distinción que organiza toda la lección (README, tabla "When to Use Agent vs Actor"):

| Situación | Agente (lenguaje natural, decide solo) | Actor (control directo, selectores) |
| --- | --- | --- |
| Layout dinámico | Se adapta | Selectores frágiles se rompen |
| Estructura conocida | Más lento que control directo | Rápido y preciso |
| Buscar un elemento | Funciona bien en lenguaje natural | Requiere selector exacto |
| Control de tiempos | Menos predecible | Control total de esperas/reintentos |
| Flujo con sorpresas (popups) | Maneja estados inesperados | Necesita ramas explícitas para cada caso |

El notebook aplica el patrón **híbrido**, que es el punto pedagógico real: Agente para la parte impredecible (navegar, cerrar popups, buscar) y control directo (`page.extract_content`) para la parte estructurada (leer precios). Detalle completo en [[computer-use-agents]].

```python
# Paso 1 — Agente: navegación abierta
search_agent = Agent(
    task="Navigate to https://www.airbnb.com. Close any pop-ups... "
         "Search for 'Stockholm, Sweden' in the search box.",
    llm=self.llm, browser=self.browser, use_vision=True
)
await search_agent.run()

# Paso 2 — control directo: extracción estructurada
search_results = await page.extract_content(
    prompt=extraction_prompt, structured_output=SearchResult, llm=self.llm
)
```

## Extracción estructurada con visión

`SearchResult`/`AirbnbListing` son modelos Pydantic (`title`, `price_per_night: float`, `currency`, `rating: Optional[float]`, `url: Optional[str]`). `page.extract_content(structured_output=SearchResult, ...)` obliga al modelo a rellenar exactamente esos campos a partir de lo que "ve" en la página, en vez de devolver prosa libre. Es la misma idea que [[structured-outputs]], pero la fuente que se restringe es una **captura de pantalla**, no texto — ver sección nueva en esa página.

Tras extraer, la comparación de precios (`sorted(result.listings, key=lambda x: x.price_per_night)`) es **código Python puro**, sin IA: el LLM se usa solo donde aporta (ver/decidir), y código determinista para el resto.

## Arquitectura de 4 pasos (README, "Architecture Overview")

1. Chrome arranca con CDP habilitado, compartido por Playwright y Browser-Use.
2. El Agente maneja navegación abierta (abrir Airbnb, cerrar popups, buscar Stockholm).
3. `page.extract_content` con el esquema Pydantic lee título, precio, rating y URL de cada listing.
4. Python compara y destaca el más barato — sin LLM.

## Ejecución real verificada

El notebook se ejecutó de punta a punta contra Airbnb real (no solo el diseño en el README). El log de la corrida muestra el Agente navegando, escribiendo "Stockholm, Sweden" en el buscador y confirmando la carga de resultados en 4 pasos, con un timeout de LLM de 60s en el primer intento que se recuperó solo en el segundo. Esto confirma que el patrón híbrido funciona en un sitio real y dinámico, no solo en teoría.

**Bug observado en la propia ejecución**: `take_screenshot()` en la clase `AirbnbSearchAgent` (versión CDP) falla con `a bytes-like object is required, not 'str'` al intentar `base64.b64encode(screenshot_bytes)`. La causa aparente: la primera versión de la clase (conexión vía `playwright_browser`) ya trataba `page.screenshot(format='png')` como si devolviera directamente un string base64 (`screenshot_base64 = await page.screenshot(...)`), mientras que la segunda versión (conexión vía `cdp_url`) asume bytes crudos y los vuelve a codificar — inconsistencia entre las dos implementaciones del mismo notebook sobre qué devuelve `page.screenshot()` en Browser-Use. No bloquea el resto del flujo (la extracción de precios sigue funcionando), pero el screenshot de depuración se pierde. Sin corregir — anotado en Pendientes.

**Dependencia declarada pero no usada**: la celda de instalación pide `pip install browser_use langchain-openai playwright`, pero el código importa `ChatAzureOpenAI` desde `browser_use` (no desde `langchain_openai`), con un comentario explícito en el código: *"Changed from langchain_openai"*. `langchain-openai` queda instalado sin usarse.

## Sesión de depuración en Windows — 3 fallos reales, resueltos

En una ejecución posterior en Windows (fuera del entorno donde se grabó el log de arriba), el notebook falló en tres puntos encadenados antes de completar la búsqueda: ruta de Chrome no encontrada, variables de `.env` con nombres equivocados/faltantes, y un `NotImplementedError` de `asyncio` al arrancar Playwright dentro del kernel de Jupyter. Los tres se reprodujeron de forma aislada y se verificó cada fix antes de tocar el notebook — detalle completo, con causa raíz y código, en [[fix-browser-use-windows-jupyter]].

## Buenas prácticas (README)

1. Empezar con Agente para explorar, cambiar a control directo cuando el flujo sea predecible.
2. Usar modelos de salida estructurados (Pydantic) para datos type-safe.
3. Añadir esperas estratégicas (`asyncio.sleep(3)`) tras acciones que disparan cambios visibles de UI.
4. Capturar screenshots mientras se itera, para depuración visual.
5. Esperar que los sitios cambien: diseñar fallback para popups y cambios de layout.
6. Combinar Agente y Actor para tener flexibilidad y precisión a la vez.

## Project Opal — la misma idea en producción

El README cierra conectando el mini-agente de la lección con **[Project Opal (Frontier)](https://support.microsoft.com/en-us/microsoft-365-copilot/get-started-with-project-opal-frontier)**, un CUA de nivel empresarial en Microsoft 365 Copilot que opera sobre un Windows 365 Cloud PC, de forma asíncrona, sobre las apps y datos de la organización. Tabla de correspondencia con conceptos ya vistos en el curso:

| Concepto del curso | Cómo lo aplica Project Opal |
| --- | --- |
| Human-in-the-loop (lección 06, sin ingerir) | Se detiene ante credenciales, datos sensibles o instrucciones ambiguas; nunca envía formularios sin confirmación explícita. *Take Control* / *Return Control* en cualquier momento. |
| Agentes seguros y confiables (lecciones 06 y 18, sin ingerir) | Aislamiento en Cloud PC, solo navegador por defecto (bloqueado vía Intune), usa la identidad del usuario, registra cada acción. |
| Planificación y [[metacognicion|metacognición]] (lecciones 07, sin ingerir, y [[09-metacognition|09]]) | Genera un plan antes de actuar y se autosupervisa; se detiene si detecta actividad sospechosa. |
| [[tool-calling|Capacidades/tools reutilizables]] (lección 04, sin ingerir) | Las "Skills" de Opal son instrucciones reutilizables entre conversaciones, análogas a las tools que ya se vieron en la lección 04. |

Disponibilidad: solo para el programa Frontier early access con suscripción M365 Copilot; feature experimental, capacidades pueden cambiar.

Fuentes: `15-browser-use/README.md`, `15-browser-use/15-browser-user.ipynb` (recorrido y explicación didáctica completa, con log de ejecución real).
