---
type: concept
date_updated: 2026-08-08
source_count: 1
---

# Recibos criptográficos (cryptographic receipts)

Un recibo es un JSON que registra qué hizo un agente (qué tool, con qué argumentos, qué resultado) **firmado digitalmente**, de forma que cualquiera con la clave pública puede verificar offline que el registro no fue alterado — sin tener que confiar en quien lo emitió. Fuente: [[18-securing-ai-agents]].

## El problema que resuelve

Un log de aplicación, un log de la nube o un log de base de datos siempre requieren confiar en alguien (el operador, el proveedor cloud). Para auditorías internas eso basta; para workloads regulados (finanzas, salud, EU AI Act) no. Los recibos hacen la acción **verificable de forma independiente**: el auditor solo necesita la clave pública y el recibo.

## Los tres mecanismos

1. **Firma Ed25519**: el agente firma el JSON con una clave privada (RFC 8032). Cualquiera con la clave pública verifica offline, sin llamada de red.
2. **Canonicalización JSON — JCS (RFC 8785)**: antes de firmar, el JSON se serializa de forma determinista (orden de claves fijo, sin ambigüedad de formato). Sin esto, dos serializadores distintos producirían bytes distintos para el mismo contenido lógico, y la firma no coincidiría aunque el dato sea "el mismo".
3. **Encadenamiento por hash**: cada recibo incluye `previous_receipt_hash`, el hash del recibo anterior. Borrar o reordenar un recibo rompe la cadena de todos los que vienen después — mismo principio que usa blockchain, aplicado a una secuencia de acciones de agente en vez de a transacciones.

```python
canonical_bytes = canonicalize(payload)          # JCS: bytes deterministas
message_hash = hashlib.sha256(canonical_bytes).digest()
signature_bytes = signing_key.sign(message_hash).signature
```

Verificación es la operación inversa: reconstruir el payload sin la firma, canonicalizar, hashear, y comprobar la firma contra la clave pública (`verify_key.verify(...)`, lanza excepción si no coincide). Cambiar un solo byte de cualquier campo (p. ej. `policy_id`) cambia el hash y invalida la firma — el atacante necesitaría la clave privada para re-firmar.

Los campos sensibles (argumentos, resultado) se guardan como **hash** (`tool_args_hash`, `result_hash`), no en crudo: evita filtrar PII/datos de negocio en el propio recibo y mantiene el tamaño acotado sin importar cuán grande sea el payload real.

## Implementación de referencia (notebook `18-signed-receipts.ipynb`)

Código real, ejecutado y verificado (ver sección de notebook en [[18-securing-ai-agents]]) — no solo diseño de README:

- **`b64url_nopad` / `b64url_decode`**: bytes ↔ texto (base64url sin relleno, RFC 4648 §5). Necesario porque firma y hash son binarios y no se pueden meter tal cual en JSON.
- **`sha256_canonical(obj)`**: `canonicalize(obj)` (JCS) → `hashlib.sha256(...).hexdigest()`, prefijado `"sha256:"`. Es la función que produce `tool_args_hash`/`result_hash`.
- **`sign_receipt(payload, signing_key, verify_key)`**: canonicaliza el payload, lo hashea, firma **el hash** (no el payload directamente) con `signing_key.sign(...)`, y devuelve `{**payload, "signature": {"alg": "EdDSA", "sig": ..., "public_key": ...}}`. Detalle importante: `signature` y `public_key` **no forman parte de los bytes canónicos firmados** — no se puede firmar algo que incluya su propia firma.
- **`verify_receipt(receipt)`**: operación inversa — reconstruye el payload quitando `signature`, recanonicaliza, rehashea, y llama a `verify_key.verify(hash, sig)`; captura `BadSignatureError` → `False`.
- **`receipt_hash(receipt)`**: igual que `sha256_canonical`, pero sobre el **recibo completo, firma incluida**. Es distinto de `tool_args_hash`/`result_hash` (que hashean solo argumentos/resultado) — este es el que se usa como `previous_receipt_hash` del siguiente recibo, así que el enlace de la cadena también ata a la firma, no solo al contenido.
- **`verify_chain(chain)`**: por cada recibo comprueba **tres cosas por separado** — firma válida (`verify_receipt`), enlace correcto (`previous_receipt_hash == receipt_hash(anterior)`, o `None` si es el primero), y `sequence` coincide con la posición real en la lista. Devuelve el detalle de cuál de las tres falló, no solo un booleano.

### El fallo en cascada al tamperar un recibo del medio

Manipular `tool_args_hash` del recibo del medio de una cadena de 3 produce **dos fallos distintos, no uno**:

- El recibo tocado falla por **firma** (su contenido ya no coincide con lo firmado).
- El recibo *siguiente* — cuya firma sigue siendo perfectamente válida, porque a él no se le tocó nada — falla por **enlace de cadena**, porque su `previous_receipt_hash` apunta a la huella del recibo del medio *original*, que ya no existe.

Consecuencia práctica: un atacante que quisiera ocultar la manipulación tendría que volver a firmar el recibo tocado *y* todos los que vienen después (para que sus `previous_receipt_hash` vuelvan a encajar) — y volver a firmar exige la clave privada en cada paso. Encadenar convierte "falsificar un dato" en "reescribir consistentemente toda la cola de la cadena", no solo un punto.

## `ReceiptedTool`: automatizar la firma sin tocar el agente

Patrón para producción: en vez de confiar en que cada autor de una tool llame a `make_receipt` a mano (se olvidará tarde o temprano), se envuelve la función original en una clase con `__call__`:

```python
class ReceiptedTool:
    def __init__(self, name, fn, signing_key, verify_key, agent_id, policy_id):
        self.fn = fn
        self.receipts = []
        ...
    def __call__(self, *args, **kwargs):
        result = self.fn(*args, **kwargs)          # ejecuta la tool real
        previous_hash = receipt_hash(self.receipts[-1]) if self.receipts else None
        receipt = make_receipt(..., sequence=len(self.receipts), previous_receipt_hash=previous_hash, ...)
        self.receipts.append(receipt)
        return result                                # transparente para quien la llama
```

Se llama exactamente igual que la función original; el recibo se genera y encadena como efecto colateral automático. El framework de agentes no necesita saber que existen los recibos: se le pasa la herramienta envuelta (`tools=[receipted_lookup]`) en vez de la función en bruto — sketch de integración con `agent_framework.foundry.FoundryChatClient` (Microsoft Agent Framework) en el propio notebook. Así se añade proveniencia a un agente ya existente sin reescribirlo.

## Las tres garantías (y sus límites — la parte más importante de la lección)

Un recibo válido prueba exactamente tres cosas:

- **Atribución**: esta clave firmó este contenido.
- **Integridad**: el contenido no cambió desde la firma.
- **Ordenación**: este recibo vino después de aquel, en la cadena.

Un recibo válido **no prueba**:

- **Corrección**: que la acción del agente fuera la correcta. Un recibo se firma igual de limpio para una respuesta buena que para una mala.
- **Cumplimiento de política**: `policy_id` registra qué política se *reclamó*, no que esa política se haya evaluado o que hubiera permitido la acción.
- **Identidad humana**: la firma dice "esta clave firmó esto", no "esta persona lo autorizó" — conectar clave con persona/organización es infraestructura de identidad aparte.
- **Veracidad de las entradas**: si el agente actúa sobre un prompt manipulado, el recibo registra la acción fielmente igual. Los recibos son *downstream* de la validación de entrada, no un sustituto.

Consecuencia explícita de la lección: **"tenemos recibos" no equivale a "estamos gobernados"**. Los recibos son cimiento forense (proveniencia honesta), no el sistema de gobernanza completo — eso necesita además validación de entrada, motor de políticas e infraestructura de identidad.

## Dónde encaja frente al resto de la wiki

Es la primera pieza de la wiki centrada en **auditoría/gobernanza post-hoc** en vez de en la ejecución del agente. Contraste útil con [[llm-as-judge]]: un juez evalúa si la respuesta fue *buena*; un recibo no evalúa nada — solo prueba *qué pasó exactamente y en qué orden*, de forma que ni el propio agente puede negarlo después. Son complementarios, no sustitutos: un recibo puede envolver también el veredicto de un juez, pero eso no lo vuelve "correcto", solo lo vuelve innegable.

También se relaciona con los controles de empresa de [[16-deploying-scalable-agents]] (`@tool(approval_mode=...)`): la aprobación humana decide *si* la acción ocurre; el recibo prueba *que* ocurrió tal cual, después del hecho.

## Producción

Checklist de la lección: clave privada fuera del código (Key Vault/KMS/HSM), clave pública publicada (JWK Set), cadena anclada externamente (transparency log), almacenamiento inmutable append-only, presupuesto de crecimiento (~500 bytes/recibo).

Patrones avanzados citados para cuando la gobernanza madura: **divulgación selectiva** (árbol Merkle, RFC 6962, revela solo algunos campos a un auditor concreto — útil para GDPR), **revocación de clave**, **firmas divididas** (autorización y resultado firmados por separado) y **migración post-cuántica** (`signature.alg` es agnóstico al algoritmo, permite `ML-DSA-65` más adelante).

**Referencia sin verificar** — el README enlaza como alternativa de producción un paquete `nobulex` (PyPI) de un repo de terceros no vinculado a Microsoft, junto a `protect-mcp`/`@veritasacta/verify` (npm). No auditado en esta sesión; no asumir que son seguros solo por aparecer en el README del curso.
