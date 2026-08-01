---
type: synthesis
date_updated: 2026-08-01
source_count: 1
---

# Fix: `az login` y la caché MSAL corrupta

## Síntoma

Un notebook que funcionaba ayer falla hoy al autenticarse. Al reintentar:

```
ERROR: User '<tu-cuenta>' does not exist in MSAL token cache. Run `az login`.
```

…y `az login` devuelve el mismo error, en bucle.

## Causa

Dos cosas encadenadas:

1. Los tokens de Azure CLI **caducan**. Los notebooks usan `AzureCliCredential` / `DefaultAzureCredential`, que necesitan una sesión activa → [[autenticacion-azure-sin-claves]].
2. La caché de cuentas de Azure CLI puede quedar en estado inconsistente, y entonces ni el login normal ni `--use-device-code` la reparan.

## El fix

```powershell
az account clear   # limpia la caché de cuentas corrupta
az login           # o: az login --use-device-code
az account show    # verificar
```

`az account clear` es la pieza que faltaba: sin ella, `--use-device-code` completa el flujo en el navegador y aun así devuelve el error de MSAL.

## Prevención

Si un notebook falla al autenticar y **el día anterior funcionaba sin haber tocado nada**, la sesión caducada es la primera hipótesis, antes de mirar el `.env` o el código.

Fuentes: [[00-course-setup]]
