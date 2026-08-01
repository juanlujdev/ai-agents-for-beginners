---
type: source-summary
date_updated: 2026-08-01
leccion: 12-context-engineering
sesiones: [b5f9d563]
fechas_origen: 2026-07-26
---

# 12 — Context engineering

Gestionar activamente **qué información** llega a la ventana de contexto, más allá de escribir un buen prompt. Ver [[context-engineering]].

## Prompt engineering vs context engineering

| | Prompt engineering | Context engineering |
| --- | --- | --- |
| Qué es | Conjunto **estático** de instrucciones | Sistema **dinámico** que gestiona información en el tiempo |
| Analogía | Dejarle a alguien una nota con instrucciones | Ser su asistente: recuerdas su historial, consultas fuentes, actualizas lo que sabes |

Con *"Resérvame un viaje a París"*: solo prompt engineering responde *"¿cuándo quieres ir?"*; con context engineering el agente ya ha mirado el calendario y las preferencias de viajes pasados y propone *"veo que estás libre la primera semana de octubre, ¿busco directos con tu aerolínea habitual?"*.

## Los 5 tipos de contexto

Instrucciones · Conocimiento (RAG, memoria) · Herramientas (definiciones y sus resultados) · Historial de conversación · Preferencias del usuario.

## Los 4 fallos de contexto

La parte más importante de la lección:

| Fallo | Qué pasa | Ejemplo | Solución |
| --- | --- | --- | --- |
| **Poisoning** | Una alucinación entra al contexto y se repite | Inventa un vuelo directo desde un aeropuerto que no existe y luego insiste en buscarlo | Validar contra API real *antes* de guardar; cuarentena si falla |
| **Distraction** | El contexto es tan largo que el modelo atiende al historial viejo | Sigue preguntando por tu equipo de mochilero de hace 2 años | Resumir periódicamente y descartar lo irrelevante |
| **Confusion** | Demasiadas tools → llama a la incorrecta | Preguntas cómo moverte por París y llama a `book_flight` | RAG sobre las descripciones de tools; **menos de 30 a la vez** |
| **Clash** | Información contradictoria conviviendo | Dijiste "económica" y luego "business", ambas siguen ahí | **Pruning** u **offloading** a scratchpad |

## Pruning vs scratchpad (la duda que costó)

Un **scratchpad** es un espacio de trabajo separado donde el agente escribe notas y razonamientos intermedios **sin contaminar el contexto que ve el modelo para responder**. Como cuando alguien te da instrucciones contradictorias: no contestas mezclando ambas, coges una hoja aparte, anotas "dijo económica… luego business… la última vale", y *después* actúas.

| | Pruning | Offloading / Scratchpad |
| --- | --- | --- |
| Qué hace | Borra o anula la info vieja en el contexto | Mantiene el historial intacto, razona la conclusión aparte |
| Cuándo | Conflictos simples (A reemplaza a B) | Conflictos que exigen reconciliar varias piezas |
| Resultado | Contexto más corto y limpio | Contexto igual de largo, con una conclusión clara añadida |

El pruning **no ocurre solo**: requiere lógica (una regla o un paso previo con el LLM) que compare la instrucción nueva contra las viejas. En la práctica se combinan: el scratchpad razona el conflicto y su conclusión es lo que poda la instrucción vieja.

## Tácticas de gestión

Scratchpad (dentro de la sesión) · Memorias (entre sesiones) · Compresión · Sistemas multi-agente (cada uno con su ventana) · Sandbox (procesar un PDF de 200 páginas fuera y traer solo el resumen) · Runtime State Objects.

**Inspección del contexto**: en vez de loguear prompts completos (riesgo de privacidad), guardar métricas — cuántos candidatos se filtraron, tokens antes/después de comprimir, ids de memoria usados.

## Notebook `12-chat_summarization.ipynb`

Implementa compresión y scratchpad. Tres estrategias frente a la ventana finita:

| Estrategia | Cómo | Ejemplo |
| --- | --- | --- |
| Truncation | Borrar los mensajes más antiguos | De 50 mensajes se envían los últimos 10 |
| Summarization | Condensar los viejos en un resumen | 20 mensajes → "Japón, presupuesto $3000" |
| Scratchpad | Guardar hechos clave fuera de la conversación | `presupuesto=$3000` en un objeto aparte |

La pieza central:

```python
@tool(approval_mode="never_require")
def summarize_preferences(conversation_notes: str) -> str:
    return f"[SUMMARY] User preferences recorded: {conversation_notes}"
```

No la llamas tú: **el propio LLM decide invocarla** cuando cree que ha acumulado suficientes preferencias, guiado por sus instrucciones. Es un scratchpad muy simplificado — en un caso real el resumen se guardaría en fichero, BD o vector store, algo que persista fuera de la ventana.

El notebook simula 6 turnos y en el turno 5 cambia "abril" por "octubre": un **Context Clash** deliberado, para que el turno 6 obligue a reconciliar. La celda 7 crea antes un agente que solo tiene instrucciones de "portarse bien" con el contexto, sin herramienta de resumen: es la versión ingenua, basada solo en prompting.

## La idea que resume la lección

> El objetivo no es meter **más** información en el contexto, sino la **correcta, en el momento correcto, sin ruido ni contradicciones**.

Ver también: [[13-agent-memory]], [[10-ai-agents-production]]
