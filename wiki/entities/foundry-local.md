---
type: entity
date_updated: 2026-08-07
source_count: 1
---

# Microsoft Foundry Local

Runtime ligero que descarga, gestiona y sirve modelos **enteramente en la máquina local** (CPU, GPU o NPU), eligiendo automáticamente el build óptimo para el hardware disponible. Expone un **endpoint HTTP compatible con OpenAI**, así que el SDK de OpenAI y el cliente OpenAI de [[microsoft-agent-framework]] funcionan contra él cambiando solo `base_url` — el código de agente escrito para la nube se reutiliza sin cambios estructurales.

**Distinto de [[azure-ai-foundry]]**: esa es la plataforma cloud donde se despliegan modelos y se registran Hosted Agents. Foundry Local es la contraparte que corre **sin nube, sin cuenta, sin `az login`**. El nombre compartido ("Foundry") es fuente real de confusión — el mismo tipo de trampa que ya documentan "los dos flujos" en la página de Azure AI Foundry (dos productos con prefijo idéntico y APIs distintas).

## Uso

```bash
foundry model run qwen2.5-7b-instruct
foundry service status
```

```python
from foundry_local import FoundryLocalManager
from openai import OpenAI

manager = FoundryLocalManager("qwen2.5-7b-instruct")
client = OpenAI(base_url=manager.endpoint, api_key=manager.api_key)  # api_key es un placeholder local
```

El SDK `foundry-local-sdk` descubre el puerto del endpoint automáticamente — no hay que hardcodearlo.

Fuentes: [[17-creating-local-ai-agents]]
