---
type: source-summary
date_updated: 2026-08-07
leccion: 17-creating-local-ai-agents
fuente: [17-creating-local-ai-agents/README.md, 17-creating-local-ai-agents/code_samples/17-local-agent-foundry-local.ipynb]
---

# 17 — Creating Local AI Agents (Microsoft Foundry Local + Qwen)

Ingerida en dos sesiones. La primera, solo desde el README (diseño documentado). La segunda: recorrido **celda a celda del notebook real** (`code_samples/17-local-agent-foundry-local.ipynb`, 20 celdas), leyendo y explicando cada función tal y como está escrita en el archivo. Sigue **sin ejecutarse contra un Foundry Local real** (no está instalado/verificado en esta máquina) — lo nuevo respecto a la primera sesión es que el código quedó verificado por lectura directa del fuente, no solo descrito desde el README.

## La idea que ordena la lección

La 16 escaló agentes *hacia arriba*, a la nube. Esta los escala *hacia abajo*, a una sola máquina: un agente que razona, llama tools, lee archivos y busca en documentación **sin ninguna llamada de inferencia a la nube**.

Tres motivadores que se repiten en trabajo de ingeniería real: **privacidad** (el código y los datos no salen de la máquina), **coste** (sin factura por token) y **funcionamiento offline**. El precio a pagar: se cambia un modelo de frontera por un [[slm|SLM]], más débil en conocimiento general y razonamiento largo.

## División de trabajo: el SLM orquesta, las tools cargan con el peso

Concepto central de la lección, con página propia: [[slm]]. Un SLM no necesita "saber" el codebase — necesita saber cuándo llamar a `read_file` o `search_docs`. Es la misma relación LLM-decide / framework-ejecuta que ya describe [[tool-calling]] en general, pero aquí es la única forma de que un modelo pequeño rinda.

## Foundry Local: el mismo código de agente, endpoint distinto

[[foundry-local]] sirve modelos en el propio equipo tras un **endpoint HTTP compatible con OpenAI**. El código de agente de las lecciones en la nube se reutiliza cambiando solo `base_url`:

```python
from foundry_local import FoundryLocalManager
from openai import OpenAI

manager = FoundryLocalManager("qwen2.5-7b-instruct")
client = OpenAI(base_url=manager.endpoint, api_key=manager.api_key)  # api_key es un placeholder local
```

Ojo con el nombre: **Foundry Local no es** [[azure-ai-foundry]]. Comparten prefijo ("Foundry") y ninguna relación de infraestructura — mismo tipo de confusión de nombres que ya documentan "los dos flujos" en la página de Azure AI Foundry.

## Qwen: por qué un modelo concreto, no cualquier SLM

Un agente solo es agente si puede llamar tools de forma fiable. Muchos SLMs conversan bien pero producen tool calls malformadas. [[qwen]] está entrenado específicamente para function calling consistente — es lo que convierte un chat local en un agente local.

## RAG local con Chroma

Mismo patrón de Agentic RAG de la Lección 5 (sin ingerir todavía), con cada pieza del pipeline en local: documentos → embeddings locales → [[chroma]] (vector DB embebida, en disco) → retrieval local → SLM local.

## MCP local

[[mcp]] es un transporte, no un servicio de nube: un servidor MCP puede correr como proceso local sobre `stdio`. La lección remarca que **local no es sinónimo de seguro por defecto** — un servidor MCP local corre con los permisos del usuario, así que hay que acotarlo (un directorio de proyecto, no el `home` completo) y tratar sus salidas como entrada a validar. Mismo principio de "MCP como límite no confiable" ya documentado como control de empresa en [[16-deploying-scalable-agents]], aplicado aquí a una máquina de desarrollador en vez de a producción.

## El sandbox del laboratorio

El notebook scopa cada tool a un directorio de proyecto (`PROJECT_ROOT`) con un chequeo de *path traversal* aislado en su propia función, `_safe_path`, que las tres tools de archivo reutilizan en vez de repetir la comprobación cada una por su cuenta:

```python
def _safe_path(path: str) -> Path | None:
    full = (PROJECT_ROOT / path).resolve()
    if full == PROJECT_ROOT or PROJECT_ROOT in full.parents:
        return full
    return None
```

`list_files`, `read_file` y una tercera tool no descrita en el README — `analyze_code` — pasan por aquí antes de tocar disco. `analyze_code` es puro conteo de líneas sobre el texto ya leído (sin ningún analizador de código real): cuenta líneas totales, líneas que empiezan por `def ` (funciones) y líneas que contienen `TODO`/`FIXME`, y devuelve el resultado como JSON para que el modelo lo lea. Es la misma clase de comprobación de sandbox que evita que un tool de lectura de archivos se salga de su directorio permitido, esté el agente en la nube o en local.

## El bucle de tool-calling, mecánica exacta

El README describe la idea; el notebook la implementa en una función, `run_agent`, que es el ejemplo concreto que le faltaba a [[tool-calling]] de "cómo se ve el bucle por dentro":

```python
messages = [{"role": "system", "content": SYSTEM_PROMPT}, {"role": "user", "content": user_query}]
for _ in range(max_iterations):          # tope de 5 vueltas: red de seguridad, no bucle infinito
    response = client.chat.completions.create(model=MODEL_ID, messages=messages, tools=TOOLS_SCHEMA)
    msg = response.choices[0].message
    if not msg.tool_calls:
        return msg.content or "(no answer)"          # el modelo ya tiene respuesta final
    messages.append({"role": "assistant", "content": msg.content,
                      "tool_calls": [tc.model_dump() for tc in msg.tool_calls]})
    for tc in msg.tool_calls:
        name, args = tc.function.name, json.loads(tc.function.arguments or "{}")
        result = TOOL_IMPL[name](**args) if name in TOOL_IMPL else f"Unknown tool: {name}"
        messages.append({"role": "tool", "tool_call_id": tc.id, "content": str(result)})
```

Tres detalles que no se ven en la descripción de alto nivel del README:
- El historial completo (`messages`) se reenvía entero en cada vuelta — el modelo no "recuerda" nada por sí mismo entre llamadas, la memoria de la conversación la lleva el código Python, no el modelo.
- `tool_call_id` enlaza cada resultado con la petición que lo originó, necesario porque el modelo puede pedir varias tools a la vez en una sola respuesta.
- El `SYSTEM_PROMPT` incluye explícitamente *"Prefer calling a tool over guessing"* — instrucción directa contra la alucinación: sin ella, nada impide que el modelo se invente el contenido de un archivo en vez de leerlo.

Las tools se registran con el schema JSON estándar de OpenAI (`TOOLS_SCHEMA`), el mismo formato que ya describe [[tool-calling]] en general — `name` + `description` + `parameters` (JSON Schema, con `required`) — solo que aquí son 4 entradas concretas (`list_files`, `read_file`, `analyze_code`, `search_docs`) en vez de una tool aislada.

Tres preguntas de prueba en el notebook confirman que el mismo bucle genérico elige la tool correcta según la pregunta, sin instrucciones explícitas de cuál usar: una pregunta sobre un archivo dispara `read_file`, una pregunta "según la documentación..." dispara `search_docs` (RAG), y una pregunta por números concretos dispara `analyze_code`.

## MCP local: el código, no solo el principio

La celda de MCP local no es solo la afirmación de que "MCP es un transporte, no un servicio de nube" — el notebook la respalda con una conexión real por `stdio` (condicionada a una variable de entorno `LOCAL_MCP_COMMAND`, así que se salta con elegancia si no hay servidor configurado):

```python
params = StdioServerParameters(command=parts[0], args=parts[1:])
async with stdio_client(params) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()
        tools = await session.list_tools()
```

Sin puerto de red: el servidor MCP es otro proceso local, hablado por el mismo canal de entrada/salida estándar que usaría cualquier programa de terminal. Ver [[mcp]] para el resto del protocolo.

## Híbrido local/nube: la misma idea de model routing, con la máquina propia como una opción más

La lección enruta por sensibilidad y dificultad (sensible u offline → local; simple y acotado → local; razonamiento difícil sobre datos no sensibles → nube; nube caída → local como degradación elegante). Es explícitamente el **model routing** que ya describe [[16-deploying-scalable-agents]] en "Estrategias de escalado" — solo que ahí las dos opciones eran dos modelos de nube (`gpt-5-nano` / `gpt-5-mini`) y aquí una de las opciones es la propia máquina.

Fuentes: `17-creating-local-ai-agents/README.md`, `17-creating-local-ai-agents/code_samples/17-local-agent-foundry-local.ipynb`.
