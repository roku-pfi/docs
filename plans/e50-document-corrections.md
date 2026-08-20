# E50 — correcciones al documento

> Cambios a aplicar en `E50pfi.pdf` (§1.2 Alcance y §3.2 Requerimientos). **Nada
> de esto es trabajo de código:** son requerimientos donde la implementación es
> la mejor respuesta y el texto quedó atrás, más las inconsistencias internas del
> documento. Los huecos reales (roles/permisos, LDAP, alarmas, métricas del IdP)
> **no** están acá — ésos se corrigen construyendo, no reescribiendo.
>
> Estado del código: 19 de agosto de 2026, `rba-*` polyrepo.

---

## 1. Reescribir: el código es la mejor respuesta

Cuatro requerimientos describen algo más débil que lo que efectivamente se
construyó. En los cuatro casos la decisión de implementación está documentada en
un ADR y es defendible; lo que falta es que el texto diga lo que se hizo.

### 1.1 RF-06 / HU-05 — TOTP o OTP por correo → **passkeys WebAuthn**

| | |
|---|---|
| **Dice** | RF-06: «El sistema debe desafiar al usuario con TOTP o con una contraseña de un solo uso por correo cuando la política exige MFA.» HU-05 nombra los mismos dos mecanismos. |
| **Hay** | Passkeys WebAuthn (registro + verificación con clave pública, `webauthn_credentials`, `MfaChallenge.webauthn_challenge`). El OTP simulado sobrevive sólo en los tests. Ni TOTP ni correo existen. |
| **Por qué el código gana** | El propio marco teórico (§2.1.3.2, *Passkeys y autenticación resistente al phishing*) argumenta que TOTP y OTP por correo son factores *phishable*: el atacante que consigue la contraseña también puede pedir el código. La passkey está ligada al origen y no es replicable. Implementar TOTP sería implementar el factor más débil de los dos que ya se discutieron en el capítulo 2. |
| **Referencia** | ADR-0027 y Demo-4 (`rba-idp/src/rba_idp/webauthn.py`). |

**Texto propuesto (RF-06):**

> El sistema debe desafiar al usuario con un autenticador resistente al phishing
> (passkey WebAuthn ligada al origen) cuando la política exige MFA, conforme al
> criterio de niveles de garantía desarrollado en §2.1.3.2.

**Texto propuesto (HU-05):**

> Como usuario, quiero confirmar mi identidad con la passkey del dispositivo
> cuando el riesgo lo justifica, para continuar el acceso sin depender de un
> código que pueda serme solicitado por un tercero.

> **Nota para la defensa:** si el tribunal pregunta por TOTP, la respuesta es que
> se evaluó y se descartó por §2.1.3.2, no que no se llegó a implementarlo.

---

### 1.2 RF-18 / HU-04 — justificación al usuario → **explicación que no revela la señal**

| | |
|---|---|
| **Dice** | RF-18: «El sistema debe mostrar al usuario final una justificación breve cuando se exige un paso adicional.» |
| **Hay** | Copy genérico deliberado («confirmá que sos vos»). Las razones por señal son visibles **sólo en el panel de administración**, nunca en la pantalla de login. |
| **Por qué el código gana** | Decidido en **ADR-0023** (*end-user login is opaque*). Quien ve la pantalla de MFA puede ser el atacante que ya tiene la contraseña. Decirle «se pidió un paso extra porque el país es desconocido» le enseña exactamente qué señal normalizar en el próximo intento. Explicar la decisión al operador y ocultarla al usuario no es una omisión: es la postura correcta. |
| **Tensión a resolver** | HU-04 viene de la encuesta y de las entrevistas — la necesidad de entender «por qué me pidieron algo más» es real. Lo que hay que reescribir no es la necesidad sino la forma en que se la satisface. |

**Texto propuesto (RF-18):**

> El sistema debe informar al usuario final que se detectó un inicio de sesión
> inusual y que por eso se solicita una verificación adicional, sin revelar qué
> señal disparó la decisión. El detalle por señal queda disponible para el
> operador de soporte en el panel de administración (RF-12).

**Texto propuesto (HU-04):**

> Como usuario, quiero saber que el paso extra responde a una verificación de
> seguridad y no a un error del servicio, para completarlo con confianza — aun
> cuando el motivo puntual no se me muestre.

Conviene además una frase en §1.2 o en el capítulo de diseño que enuncie el
principio: *la explicabilidad es hacia el operador, no hacia quien está
intentando entrar.*

---

### 1.3 RF-03 — «bajo, medio o alto» → **cuatro niveles**

| | |
|---|---|
| **Dice** | RF-03: «clasificarlo en un nivel (bajo, medio o alto)». |
| **Hay** | Cuatro: `LOW / MEDIUM / HIGH / CRITICAL` (`rba-contracts/src/rba_contracts/enums.py`). |
| **Por qué el código gana** | Con tres niveles no hay banda que llegue a `BLOCK`. La política mapea `HIGH → REAUTHENTICATE` y `CRITICAL → BLOCK`: el cuarto nivel es precisamente el que hace alcanzable la acción más severa que promete el alcance. Quitarlo obligaría a que «alto» signifique dos cosas distintas. |

**Texto propuesto (RF-03):**

> El sistema debe calcular un puntaje de riesgo combinando reglas explícitas con
> un modelo de aprendizaje automático, clasificarlo en un nivel (bajo, medio,
> alto o crítico) e incluir las razones o factores que explican el puntaje.

Los umbrales son configurables por aplicación; conviene mostrar en el capítulo de
diseño la tabla real (`config/policy-config.yaml`): por defecto 0.30 / 0.60 /
0.80 / 1.00, y para `demo-banking-app` (sensibilidad alta) 0.20 / 0.45 / 0.70 /
1.00.

---

### 1.4 RF-04 / §1.2 — «limitar la tasa» → **`REAUTHENTICATE`**

| | |
|---|---|
| **Dice** | Tanto el alcance (§1.2, módulo de decisión) como RF-04 listan cuatro acciones: permitir, exigir MFA, bloquear **o limitar la tasa**. |
| **Hay** | Cuatro acciones, ninguna de ellas es limitación de tasa: `ALLOW / REQUIRE_MFA / REAUTHENTICATE / BLOCK`. |
| **Por qué el código gana** | La limitación de tasa es un control de infraestructura (un *reverse proxy*, un *API gateway*), no una decisión de riesgo por intento: no depende del puntaje ni del usuario. `REAUTHENTICATE` sí es una respuesta graduada del motor — más fuerte que un MFA, más débil que un bloqueo — y es la que ocupa la banda intermedia frente a una ráfaga de fallos (≥3 en 24 h → `REAUTHENTICATE`; ≥10 → `BLOCK`, ADR-0027). La escalera de cuatro peldaños es más expresiva que la lista original. |

**Texto propuesto (RF-04):**

> El sistema debe aplicar una política que asocie el nivel de riesgo con una
> acción: permitir el acceso, exigir MFA, exigir una reautenticación completa o
> bloquear el intento.

Mismo cambio en la viñeta «Módulo de decisión» del §1.2. Si se quiere conservar
la idea de limitación de tasa, corresponde mencionarla como control de
infraestructura fuera del motor, no como acción de política.

---

## 2. Inconsistencias internas del documento

Contradicciones del documento **consigo mismo**. No requieren decisión técnica:
una de las dos frases está mal y hay que elegir cuál.

### 2.1 LDAP: opcional en el alcance, obligatorio en el requerimiento

- **§1.2, capa de integración:** «De forma **opcional**, el servicio puede
  integrarse con un LDAP existente como fuente adicional de usuarios y grupos,
  sin migrar el directorio.»
- **RF-19:** «El sistema **debe** integrarse con un LDAP existente para resolver
  usuarios y grupos, como fuente adicional y no como único almacén de
  identidades.»

Un requerimiento no puede ampliar el alcance; el propio §3.2 lo dice («El alcance
—módulos, integraciones y exclusiones— es el declarado en el Capítulo 1»). Como
además **no está implementado**, la corrección consistente es alinear RF-19 con el
alcance:

> RF-19 — El sistema **puede** integrarse con un LDAP existente para resolver
> usuarios y grupos, como fuente adicional y no como único almacén de
> identidades. La integración se declara opcional en el alcance (§1.2) y no forma
> parte del prototipo entregado.

Alternativa: mover RF-19 a §1.2.2 (Limitaciones y exclusiones) junto con SAML y
multi-tenant. Es la opción más limpia si no se va a construir.

### 2.2 Cantidad de niveles de riesgo

Ver §1.3. Es a la vez inconsistencia documento↔código y reescritura por mejora:
el alcance dice «asigna un nivel de riesgo» sin enumerar, RF-03 enumera tres, el
código tiene cuatro. Sólo RF-03 necesita cambiar.

### 2.3 Limitación de tasa como acción

Ver §1.4. Aparece dos veces en el documento (§1.2 y RF-04) y cero veces en el
código. Hay que corregir **ambas** ocurrencias, no sólo RF-04.

---

## 3. Fuera de este archivo

No son problemas de redacción y no se resuelven reescribiendo:

| Tema | Requerimientos | Qué pasa |
|---|---|---|
| Roles y permisos | RF-13, RF-21, §1.2 («control de acceso basado en roles») | Existen usuarios, grupos, aplicaciones y grants grupo→app. **No existen roles**, y `permission` es una columna fijada a la constante `"access"`. Hay que decidir: modelarlos, o angostar el alcance a «acceso por grupo a aplicaciones». **Decidir antes de escribir el capítulo de arquitectura.** |
| Identificador de dispositivo | RF-02 | Se recolecta la *clase* de dispositivo (device_type / os / browser del User-Agent), no un identificador estable. «Dispositivo conocido» significa hoy «tipo de dispositivo conocido». |
| Retardo progresivo | RF-07 | Las bandas de fallos escalan la acción, pero nada se retarda: no hay backoff ni `Retry-After`. |
| Alarmas | RF-17 | No hay reglas de alerta. Prometheus/Grafana ya están desplegados y el PDP ya exporta `rba_decisions_total` — es configuración, no código. |
| Métricas de autenticación | RF-20, RNF-04 | El PDP exporta puntaje y decisión; **el IdP no expone métricas** y el resultado del desafío MFA no se persiste en ningún lado. |
| MFA permanente | RF-08 | No hay interruptor de MFA siempre, ni por usuario ni por aplicación. Viene de la encuesta. |
| Modo de sólo monitoreo | RF-09, RNF-08 | En desarrollo. |
| Degradación a MFA | RF-10, RNF-03 | En desarrollo. |
