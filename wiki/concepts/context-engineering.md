---
type: concept
date_updated: 2026-08-01
source_count: 3
---

# Context engineering

Gestionar **qué información** llega a la ventana de contexto, y cuándo. Frente al prompt engineering (conjunto estático de instrucciones), es un sistema dinámico que mantiene información a lo largo del tiempo.

> El objetivo no es meter **más** información, sino la **correcta, en el momento correcto, sin ruido ni contradicciones**.

## Los 5 tipos de contexto

Instrucciones · Conocimiento (RAG, memoria) · Herramientas y sus resultados · Historial de conversación · Preferencias del usuario.

## Los 4 fallos

| Fallo | Qué pasa | Solución |
| --- | --- | --- |
| **Poisoning** | Una alucinación entra al contexto y se repite, contaminando decisiones futuras | Validar contra fuente real *antes* de guardar; cuarentena si falla |
| **Distraction** | El contexto es tan largo que el modelo atiende al historial viejo en vez de a la petición actual | Resumir periódicamente, descartar lo irrelevante |
| **Confusion** | Demasiadas herramientas → llama a la incorrecta | RAG sobre las descripciones; **<30 tools a la vez** → [[tool-calling]] |
| **Clash** | Información contradictoria conviviendo | **Pruning** u **offloading** a scratchpad |

## Scratchpad

Espacio de trabajo separado donde el agente escribe notas y razonamientos intermedios **sin contaminar el contexto que ve el modelo para responder**. Cuando alguien te da instrucciones contradictorias no contestas mezclándolas: coges una hoja aparte, anotas cuál es la vigente, y *después* actúas.

| | Pruning | Offloading / Scratchpad |
| --- | --- | --- |
| Qué hace | Borra o anula la info vieja | Mantiene el historial, razona la conclusión aparte |
| Cuándo | Conflictos simples (A reemplaza a B) | Conflictos que exigen reconciliar varias piezas |

El pruning **no ocurre solo**: necesita lógica que compare la instrucción nueva contra las viejas. En la práctica se combinan — el scratchpad razona y su conclusión poda.

## Compresión

| Estrategia | Cómo |
| --- | --- |
| Truncation | Borrar los mensajes más antiguos |
| Summarization | Condensar los viejos en un resumen |
| Scratchpad | Guardar hechos clave fuera de la conversación |

En [[12-context-engineering]] la compresión se implementa como una **tool que el propio LLM decide invocar** (`summarize_preferences`), no como algo que llames tú.

## Frontera con memoria

Scratchpad y compresión operan **dentro** de la sesión; las **memorias** persisten entre sesiones. Esa es la frontera con [[13-agent-memory]]: `AgentSession` cubre lo primero, un almacén externo ([[cognee]], Mem0) lo segundo.

## Inspección sin comprometer privacidad

En vez de loguear prompts completos, guardar métricas: cuántos candidatos se filtraron, tokens antes/después de comprimir, ids de memoria usados. Encaja con la observabilidad de [[10-ai-agents-production]].

Fuentes: [[12-context-engineering]], [[13-agent-memory]]
