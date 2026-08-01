---
type: concept
date_updated: 2026-08-01
source_count: 4
---

# Autenticación en Azure sin claves de API

Todo el curso se autentica con la **sesión de Entra ID** de `az login`, no con claves en el código o el `.env`.

```python
credential = AzureCliCredential()      # solo la sesión del CLI
credential = DefaultAzureCredential()  # cadena: env vars → CLI → managed identity → …
```

Por eso, ante la pregunta *"¿hace falta la clave de API?"* la respuesta es **no**, siempre que `az login` esté hecho con la cuenta que tiene acceso al proyecto.

## Lo que hay que saber

- **La sesión caduca.** Un notebook que funcionaba ayer falla hoy sin haber tocado nada: esa es la primera hipótesis, antes de mirar el `.env` o el código.
- **La caché puede corromperse**, y entonces ni `az login` ni `--use-device-code` la arreglan. Se limpia con `az account clear` → [[fix-az-login-cache-msal]].
- **En Windows conviene `process_timeout=60`**: el `az` CLI puede tardar más de los 10 s por defecto en refrescar el token y lanza `CredentialUnavailableError` a mitad del workflow → [[fix-notebook-04-sdk-desactualizado]].
- `AzureCliCredential` **sí** es async context manager (`async with`), a diferencia de `FoundryChatClient`.

## El patrón elegante: token provider

En vez de pasar una clave fija, pasar una **función** que genera tokens frescos bajo demanda. Los tokens de Azure caducan cada hora y así se renuevan solos:

```python
token_provider = get_bearer_token_provider(credential, "https://ai.azure.com/.default")
ChatOpenAI(model=deployment, base_url=..., api_key=token_provider)
```

Es como pedirle al guardia una contraseña temporal nueva cada vez, en lugar de tener una fija guardada → [[14-microsoft-agent-framework]].

Fuentes: [[00-course-setup]], [[03-agentic-design-patterns]], [[08-multi-agent]], [[14-microsoft-agent-framework]]
