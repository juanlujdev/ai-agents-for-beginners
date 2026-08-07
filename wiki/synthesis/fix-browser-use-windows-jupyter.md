---
type: synthesis
date_updated: 2026-08-07
source_count: 1
---

# Fix: `15-browser-user.ipynb` no arrancaba en Windows (3 fallos encadenados)

Sesión de depuración real ejecutando el notebook de [[15-browser-use]] en Windows. Cada fallo se reprodujo de forma aislada fuera del notebook (con un script suelto) antes de aplicar el fix, para confirmar la causa raíz en vez de adivinar.

## 1 — `start_chrome_with_cdp`: "Chrome not found"

El notebook original adivinaba la ruta de Chrome con una lista fija (`/Applications/Google Chrome.app/...`, `/usr/bin/google-chrome`, y los strings `'chrome'`/`'chromium'` esperando encontrarlos en el PATH). En Windows ninguna entrada de esa lista sirve: no hay Chrome en esas rutas, y `chrome`/`chromium` no suelen estar en el PATH (confirmado con `Get-Command chrome` → nada).

**Causa raíz**: la lista no cubre Windows, y encima ignora que **Playwright ya descargó su propio Chromium** en la celda `!playwright install chromium` (queda en `%LOCALAPPDATA%\ms-playwright\chromium-<versión>\chrome-win64\chrome.exe`).

**Fix** (probado con `os.path.exists()` + lanzamiento real de Chrome con `--remote-debugging-port` y verificación de `http://localhost:<puerto>/json/version`, antes de tocar el notebook):

```python
async with async_playwright() as p:
    chrome_exe = p.chromium.executable_path   # fuente de verdad, no rutas adivinadas
```

Además de arreglar Windows, queda más corto y funciona igual en las 3 plataformas — no hace falta ninguna lista de rutas.

## 2 — LLM con `Deployment: None` / sin API key

`ChatAzureOpenAI` (el de `browser_use`, no el de `langchain_openai` — ver [[browser-use]]) lee 4 variables de entorno concretas: `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_CHAT_DEPLOYMENT_NAME`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_API_VERSION`. El `.env` del repo tenía `AZURE_OPENAI_DEPLOYMENT` (el nombre que usan las otras ~35 referencias del curso, vía `AzureCliCredential`/Entra ID — ver [[autenticacion-azure-sin-claves]]) pero no `AZURE_OPENAI_CHAT_DEPLOYMENT_NAME`, y **no tenía `AZURE_OPENAI_API_KEY` en absoluto**. Esta lección concreta necesita autenticación por clave, no Entra ID, porque usa el cliente propio de `browser_use` en vez del SDK estándar de Azure AI.

**Causa raíz**: dos convenciones de variable conviviendo en el mismo `.env` — el resto del curso usa `AZURE_OPENAI_DEPLOYMENT` + Entra ID; esta lección necesita `AZURE_OPENAI_CHAT_DEPLOYMENT_NAME` + `AZURE_OPENAI_API_KEY` porque el `.env.example` del repo documenta la key como *"Optional: use a key instead of Entra ID"*, pero `browser_use.ChatAzureOpenAI` no soporta Entra ID — la key deja de ser opcional para esta lección.

**Fix**: añadir las 3 variables que faltaban al `.env`, sin tocar `AZURE_OPENAI_DEPLOYMENT` (usada por 35 archivos de otras lecciones — confirmado con `grep` antes de editar nada). `AZURE_OPENAI_API_VERSION` se dejó sin definir a propósito: el propio comentario del notebook indica que sin ella usa la versión más reciente por defecto.

## 3 — `NotImplementedError` al arrancar Playwright

Con Chrome ya localizable y el LLM ya configurado, `async_playwright()` fallaba en `asyncio.create_subprocess_exec` con `NotImplementedError` — dentro de `async with async_playwright() as p:`, antes incluso de tocar Chrome.

**Causa raíz** (confirmada leyendo el código fuente de `ipykernel`, `kernelapp.py:718-730`, método `_init_asyncio_patch`): en Windows, `ipykernel` fuerza `WindowsSelectorEventLoopPolicy` porque `Tornado 6` (la librería que gestiona la comunicación del kernel) no es compatible con `ProactorEventLoop`. El problema: `WindowsSelectorEventLoop` **no soporta lanzar subprocesos** (documentado en la propia librería `asyncio`), y `async_playwright()` necesita lanzar su driver como subproceso para arrancar.

Reproducido de forma aislada (fuera del notebook, forzando la misma política) antes de tocar nada, y verificado el fix con un script real que sí lanza Chromium:

```python
def run_in_worker_thread():
    asyncio.set_event_loop_policy(asyncio.WindowsProactorEventLoopPolicy())
    asyncio.run(launch_playwright())   # en un hilo aparte, con loop aparte, sí funciona

t = threading.Thread(target=run_in_worker_thread)
t.start(); t.join()
```

**Fix aplicado** al notebook: en la celda final, solo en Windows (`sys.platform == "win32"`), ejecutar `main()` dentro de un hilo nuevo con su propia `WindowsProactorEventLoopPolicy`, mientras el hilo del kernel (que sigue en `Selector`, sin tocarlo) espera con `thread.join()`. En Mac/Linux se mantiene `await main()` directo, porque ahí el problema no existe — `ipykernel` solo hace este parche en Windows.

## Patrón general

Los tres fallos comparten forma: el notebook (a juzgar por los paths de Mac/Linux hardcodeados en el original) se escribió y probó fuera de Windows, y asumía comportamiento que **no se sostiene ahí** — ni las rutas de Chrome, ni la política de event loop de `asyncio` dentro de un kernel de Jupyter. Cualquier notebook de este curso que combine `asyncio` con lanzar subprocesos (Playwright, drivers de navegador, procesos externos) y se ejecute vía Jupyter en Windows es candidato al mismo problema del punto 3.

Fuentes: [[15-browser-use]]
