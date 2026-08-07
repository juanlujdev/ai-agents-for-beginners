---
type: source-summary
date_updated: 2026-08-08
leccion: 18-securing-ai-agents
fuente: [18-securing-ai-agents/README.md, 18-securing-ai-agents/code_samples/18-signed-receipts.ipynb]
---

# 18 — Securing AI Agents with Cryptographic Receipts

Ingerida en dos sesiones: primero el README (introducción, cada sección, ejemplos), después el notebook completo (`code_samples/18-signed-receipts.ipynb`), recorrido celda a celda en sesión didáctica en castellano dirigida a un lector sin experiencia previa en programación. Mismo patrón que las lecciones 16 y 17: README primero, notebook en una sesión posterior.

## La idea que ordena la lección

Última lección del curso listado en el README (cierra el arco tras [[15-browser-use]]/[[16-deploying-scalable-agents]]/[[17-creating-local-ai-agents]]). Frente al auditor que pregunta "¿cómo sé que estos logs no fueron editados?", la respuesta no puede ser "confía en mí" — tiene que ser matemáticamente verificable por un tercero sin acceso al sistema emisor. Concepto central, con página propia: [[recibos-criptograficos]].

## Estructura del README

1. **El problema**: logs de aplicación/nube/BD requieren confiar en alguien; para workloads regulados (EU AI Act, finanzas, salud) eso no basta.
2. **Qué es un recibo**: JSON firmado (Ed25519) + canonicalización JCS (RFC 8785) + encadenamiento por hash. Ver [[recibos-criptograficos]] para el detalle de los tres mecanismos y las tres garantías.
3. **Producir un recibo en Python**: ~15 líneas con `nacl.signing` (Ed25519) y `jcs.canonicalize` (JCS) — sin librería propia del curso, primitivas estándar.
4. **Verificar y detectar manipulación**: función inversa; cambiar un byte de cualquier campo invalida la firma (`BadSignatureError`).
5. **Encadenar recibos**: `previous_receipt_hash` liga cada recibo al anterior; borrar uno de en medio rompe la cadena para todos los posteriores sin necesitar re-verificar cada firma una a una.
6. **Qué prueban y qué no** — sección que el propio README marca como "la más importante": atribución/integridad/orden sí, corrección/cumplimiento de política/identidad humana/veracidad de entradas no.
7. **Referencias de producción**: Key Vault/KMS/HSM para la clave privada, PyNaCl + `jcs` como base, o una librería de producción — incluye un paquete de terceros (`nobulex`, PyPI) sin vínculo verificado con Microsoft; ver nota de cautela en [[recibos-criptograficos]].
8. **Checklist de producción** y **patrones avanzados** (divulgación selectiva vía Merkle/RFC 6962, revocación de clave, firmas divididas, migración post-cuántica) — resumidos en [[recibos-criptograficos]].

## Notebook: `18-signed-receipts.ipynb` (leído celda a celda, ya ejecutado)

El archivo, tal y como está en disco, **ya tiene outputs de una ejecución real** (comparado contra el commit en git: `execution_count` con números reales, no `null`, y valores concretos como el timestamp `2026-08-07T22:51:02Z` o la clave pública `Nm1vdvDkEDKpWp-PfRDRH4LJQa_iv6TUMMFlYPAGYww`) — no se generaron en esta sesión de conversación ni se sabe con certeza quién los produjo, pero son resultados genuinos, no una plantilla. Todos coinciden exactamente con lo que el propio notebook predice antes de cada celda:

- Recibo firmado → `Receipt is valid: True`.
- Recibo tamperado (`policy_id` cambiado) → `Tampered receipt valid? False`.
- Cadena de 3 recibos (`lookup_flights` → `hold_seat` → `confirm_booking`) → los 3 `VALID`.
- Cadena con el recibo del medio tamperado → recibo 0 `VALID` (intacto), recibo 1 `INVALID (failed: signature)` (se le cambió `tool_args_hash`), recibo 2 `INVALID (failed: chain link)` (su firma sigue siendo válida, pero su `previous_receipt_hash` ya no coincide con el recibo 1 modificado).
- Herramienta envuelta con `ReceiptedTool`, llamada 3 veces → 3 recibos, los 3 `VALID`.

El diseño descrito en el README (ver [[recibos-criptograficos]]) queda así confirmado por comportamiento observado, no solo por lectura de la documentación.

### Estructura de las 4 secciones + reto

1. **Firmar y verificar** — generar par de llaves Ed25519 (`signing.SigningKey.generate()`), construir el payload (con `tool_args_hash`/`result_hash` en vez de los datos en crudo, para no filtrar contenido sensible), firmar (`sign_receipt`) y verificar (`verify_receipt`) sin depender de ningún servicio externo.
2. **Tamperear y ver fallar** — cambiar `policy_id` tras firmar invalida la firma; cualquier campo produce el mismo efecto.
3. **Cadena de 3 recibos** — `receipt_hash()` calcula la huella del recibo completo (firma incluida), que el siguiente recibo guarda como `previous_receipt_hash`; `verify_chain()` hace tres comprobaciones independientes por recibo (firma, enlace con el anterior, número de secuencia), lo que permite distinguir *qué* falló exactamente al romper la cadena.
4. **`ReceiptedTool`** — clase envoltorio (`__call__`) que convierte cualquier función Python en una herramienta que genera y encadena su recibo automáticamente en cada llamada, sin que el código que la usa note diferencia. Boceto de integración con `agent_framework.foundry.FoundryChatClient` (Microsoft Agent Framework): se pasa la herramienta envuelta (`tools=[receipted_lookup]`), no la función en bruto — el framework y el modelo no saben que existen los recibos.

**Reto adicional del notebook** (no resuelto en esta sesión): añadir un campo `request_id` (UUID) al esquema y comprobar que sigue verificando, y que falla si se modifica tras firmar.

Detalle técnico completo (funciones, firmas de cada una, mecanismo de fallo en cascada) en [[recibos-criptograficos]].

Fuentes: `18-securing-ai-agents/README.md`, `18-securing-ai-agents/code_samples/18-signed-receipts.ipynb`.
