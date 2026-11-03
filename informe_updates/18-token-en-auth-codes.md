# El código de callback ya no guarda el token (C2, RF-14; 2026-11-03)

- **Qué cambió:** `auth_codes` guarda el hash de la sesión (`session_hash`), no
  el token en claro. Al canjear el código, el IdP rota el token: la aplicación
  recibe uno nuevo y el que tenía la página de login deja de valer. Ningún
  token de sesión se guarda en claro en ninguna tabla.
- **RF-14:** la redacción quieta del 24/10 ("la tabla de sesiones guarda el
  token como resumen") ya es cierta sin matices; puede volver a decir que el
  token se guarda **únicamente** como resumen.
- **Tabla `schema_idp_runtime` / Figura 3.20 (esquema de sesión):**
  `session_token varchar(128)` → `session_hash varchar(64)`. Figura nueva:
  `docs/diagrams/png/05-schema-idp-runtime.png`.
- **Figura 3.17 (flujo 2 de 2), paso 40:** dice "consumir auth_code y rotar el
  token de la sesión". Figura nueva: `docs/diagrams/png/02-session-authorization.png`.
- **"Sesión y factores":** si se quiere, una frase: el código es de un solo uso
  y su canje entrega un token nuevo a la aplicación.
- **Cookie de la aplicación (contrato de sesión):** puede decirse tal cual lo
  hace el código: `HttpOnly`, `SameSite=Lax` y `Secure` (obligatorio en la
  nube; en local, sin HTTPS, no). El panel del presentador usa `SameSite=Strict`
  y sólo vale bajo `/presenter`.
