# Sesiones de usabilidad: protocolo y plantilla (capítulo 3, pruebas)

Nota para el capítulo de pruebas. Las sesiones **no se hicieron todavía**: esto
es el protocolo para hacerlas sobre el despliegue en AWS y la plantilla de
resultados para volcarlas al informe. Los datos se completan después de
correrlas; no inventar cifras.

## 1. Objetivo

Comprobar con personas reales tres cosas:

- **El usuario final** entiende y completa el inicio de sesión, el alta y la
  verificación con passkey, y el aviso de bloqueo, sin ayuda.
- **El operador** puede leer una decisión ("¿por qué se le pidió la passkey a
  este usuario?"), cambiar la política de monitor a activo y encontrar un
  ataque en la consola.
- **La fricción del enfoque adaptativo** se percibe menor que la de "passkey
  siempre", que es la tesis del proyecto en experiencia de usuario.

## 2. Método

- **Tipo**: prueba moderada, presencial o por videollamada, con *thinking
  aloud* (la persona dice lo que piensa mientras usa el sistema).
- **Participantes**: 5 a 8 en total. Con 5 por perfil aparecen la mayoría de
  los problemas graves (Nielsen, 2000).
  - **Perfil A, usuario final**: usa homebanking o apps con login a menudo. No
    necesita conocimientos técnicos.
  - **Perfil B, operador**: alguien con experiencia en sistemas, soporte o
    seguridad (compañeros, docentes, colegas). Puede ser un subconjunto más
    chico (2 a 3).
- **Entorno**: el despliegue de AWS (URLs de CloudFront), un navegador Chrome
  o Safari actual y el celular propio del participante para la passkey (o una
  passkey creada en la máquina de prueba).
- **Duración**: unos 30 min por participante (A: unos 15 min de tareas; B:
  unos 20).
- **Consentimiento**: verbal o escrito, con estas condiciones:
  - se registran tiempos, errores y comentarios, sin datos personales;
  - las cuentas son de prueba;
  - grabar la pantalla solo con permiso.
- **Moderador**: lee las consignas tal cual, no ayuda salvo bloqueo total, y
  anota.

## 3. Tareas

Cada tarea tiene un criterio de éxito observable. El moderador anota:

| Campo | Valores |
|---|---|
| Resultado | **éxito / con ayuda / fracaso** |
| Tiempo | segundos |
| Errores | cantidad |
| Comentarios | textuales, entre comillas |

### Perfil A: usuario final (Roku Bank)

| # | Consigna (se lee en voz alta) | Éxito si… | Preparación |
|---|---|---|---|
| A1 | "Activá tu cuenta con este código y creá tu passkey." | llega a "listo" en `/activate` | código de activación recién emitido |
| A2 | "Entrá a tu banco." | ve el inicio del banco | política en monitor: pide la passkey |
| A3 | "Entrá otra vez." (el operador ya pasó a activo) | entra sin passkey y no se confunde | Diego pasa la política a activo |
| A4 | "Entrá desde tu celular." | completa la passkey y entra | red móvil; contexto nuevo |
| A5 | "Probá entrar con una contraseña que no es la tuya, varias veces, y después con la correcta." | entiende que la cuenta está en pausa y que es temporal | 5 contraseñas incorrectas, después la correcta (pausa de 15 min, ADR-0045) |

Preguntas tras A3: "¿Notaste alguna diferencia entre la segunda y la tercera
vez? ¿Qué te pareció?"

### Perfil B: operador (consola, Roku Bank)

| # | Consigna | Éxito si… |
|---|---|---|
| B1 | "Encontrá el último inicio de sesión de Martín y decime por qué se le pidió la passkey." | nombra al menos un motivo de la página del evento (red nueva, viaje, etc.) |
| B2 | "Hoy el banco pide passkey siempre. ¿Cuántas passkeys se habrían ahorrado con el motor?" | lee la cifra en Resumen |
| B3 | "Hacé que el motor empiece a decidir." | cambia Motor → Activo y guarda |
| B4 | (el presentador lanza el stuffing sostenido) "¿Pasó algo raro? ¿Qué hizo el sistema?" | identifica las IP bloqueadas y que Martín no quedó bloqueado |
| B5 | "Dale acceso de solo consulta a Sofía." | asigna el grupo o rol correcto en Directorio |

## 4. Cuestionario al final: SUS

System Usability Scale (Brooke, 1996), 10 ítems de 1 (muy en desacuerdo) a 5
(muy de acuerdo). Usar una versión en español validada y citarla. Cálculo:

- ítems impares: (respuesta − 1);
- ítems pares: (5 − respuesta);
- sumar y multiplicar por 2,5, para un puntaje de 0 a 100;
- referencia: **68 es el promedio** (Sauro, 2011); más de 80 es bueno.

Para el perfil A se agrega una pregunta propia:

- "Comparado con un banco que pide un código o passkey en **cada** ingreso,
  este fue: mucho más molesto / igual / mucho menos molesto" (escala de 1 a 5).

## 5. Qué medir y cómo reportarlo

| Métrica | Cálculo | Dónde va |
|---|---|---|
| Tasa de éxito por tarea | éxitos / participantes (con ayuda = 0,5) | tabla por tarea |
| Tiempo por tarea | mediana y rango, en segundos | misma tabla |
| Errores por tarea | promedio | misma tabla |
| SUS | media y desvío, por perfil | párrafo + gráfico de barras |
| Fricción percibida | distribución de la pregunta propia | párrafo |
| Problemas encontrados | lista con severidad 0–4 (Nielsen) y tarea | tabla de hallazgos |

**Plantilla de resultados para el informe:**

| Tarea | n | Éxito | Tiempo (mediana) | Errores | Observación principal |
|---|---|---|---|---|---|
| A1 alta y passkey | | | | | |
| A2 login (monitor) | | | | | |
| A3 login (activo) | | | | | |
| A4 dispositivo nuevo | | | | | |
| A5 pausa por errores propios | | | | | |
| B1 leer una decisión | | | | | |
| B2 passkeys ahorradas | | | | | |
| B3 activar el motor | | | | | |
| B4 detectar el ataque | | | | | |
| B5 asignar un rol | | | | | |

| Hallazgo | Tarea | Severidad (0–4) | Cambio propuesto |
|---|---|---|---|
| | | | |

## 6. Preparación de cada sesión (operador del estudio)

1. **Presentador → Limpiar fallos.** Política de Roku Bank en monitor con
   passkey siempre.
2. **Una cuenta de prueba por participante del perfil A.** Usar Martín con el
   perfil olvidado ("Olvidar perfil") o crear un usuario nuevo en Directorio, y
   emitir su código de activación.
3. **Historial habitual.** Para que A3 tenga sentido, el participante necesita
   inicios previos desde ese equipo: hacer 3 a 5 ingresos antes de la sesión,
   o usar `make warm-up` para la cuenta de prueba.
4. **Al terminar**: Limpiar fallos y volver la política a monitor.

## 7. Amenazas a la validez (para declararlas)

- **Muestra chica y por conveniencia** (conocidos del autor): los resultados
  son indicativos, no estadísticos.
- **Entorno de prueba**: el banco y las cuentas son ficticios, y el
  participante sabe que no arriesga dinero.
- **Moderador**: el autor modera su propio sistema (sesgo); mitigación:
  consignas leídas tal cual, sin ayuda.
- **El viaje imposible es simulado** (ADR-0035): queda fuera de las tareas.

## Para revisar en el informe

- **Capítulo 3, pruebas**: método (sección 2), tareas (sección 3), resultados
  (plantilla de la sección 5) y amenazas (sección 7).
- **Requisitos**: si el E50 tiene un requisito no funcional de usabilidad o de
  fricción, vincularlo con la pregunta propia y con B2.
- **Referencias**:
  - Brooke, J. (1996). *SUS: A "quick and dirty" usability scale*.
  - Nielsen, J. (2000). *Why You Only Need to Test with 5 Users*.
  - Sauro, J. (2011). *A Practical Guide to the System Usability Scale*.
