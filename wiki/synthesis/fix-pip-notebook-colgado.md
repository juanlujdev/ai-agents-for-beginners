---
type: synthesis
date_updated: 2026-08-01
source_count: 2
---

# Fix: celda de `pip install` que parece colgada en un notebook

Dos situaciones distintas que se parecen mucho pero no son lo mismo.

## Caso 1 — Lento, pero vivo

`%pip install` tardando 8-10 minutos. **No está colgado.** El resolvedor de dependencias de pip recalcula compatibilidad entre ~100 paquetes con muchas dependencias transitivas, sobre todo si `requirements.txt` ya se instaló antes.

Cómo verificarlo (el parpadeo del cursor **no** es un indicador — puede ser solo pérdida de foco de la terminal):

```powershell
Get-Process pip, python | Select-Object Id, CPU, StartTime
```

Si el tiempo de CPU crece, sigue trabajando. Al terminar, **reiniciar el kernel** para que se carguen las versiones nuevas.

## Caso 2 — Colgado de verdad: `!pip` en vez de `%pip`

Celda parada 8 horas, sin ningún proceso `pip` vivo (solo un `conhost.exe` huérfano). El pip ya había terminado; lo que estaba bloqueado era el kernel.

```python
! pip install agent-framework azure-ai-projects -U -q   # ← cuelga en Windows
%pip install agent-framework azure-ai-projects azure-identity -q   # ← correcto
```

**Causa:** en Windows, `!pip` lanza el subproceso vía `cmd.exe`/conhost, mientras que `%pip` es el hook de IPython que ejecuta pip dentro del propio intérprete del kernel. Con `!`, al terminar el subproceso el pipe de stdout a veces no se cierra bien y el kernel queda en estado *busy* para siempre.

**Fix:** reiniciar el kernel (no hace falta matar procesos a mano) y sustituir `!pip` por `%pip`. Quitar también el `-U`, que fuerza a re-resolver toda la cadena de dependencias sin necesidad.

Fuentes: [[00-course-setup]], [[02-explore-agentic-frameworks]]
