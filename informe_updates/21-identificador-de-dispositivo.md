# Identificador estable de dispositivo (RF-02, ADR-0047; 2026-11-03)

- **Qué cambió:** el IdP guarda en el navegador una cookie aleatoria de larga
  duración (`rokusec_device`, HttpOnly). El motor recibe sólo un resumen de esa
  cookie salado con la organización, así que un navegador es un dispositivo
  distinto para cada organización. El perfil recuerda los últimos 20
  dispositivos en los que el usuario completó un inicio.
- **Cómo pesa:** un dispositivo conocido baja el riesgo en un término fijo
  (−ln 10 ≈ −2,30 en log-riesgo: "diez veces más probable que sea el dueño").
  Uno desconocido no suma nada (las cookies se borran y los equipos cambian).
  Las reglas siguen pudiendo sólo subir la acción. Un atacante con la
  contraseña nunca tiene la cookie, y su navegador no se aprende porque no
  completa el inicio.
- **Capítulo 3, señales / RF-02:** el 24/10 se cambió la redacción a "clase de
  dispositivo". Ahora hay las dos cosas: la clase (tipo, sistema, navegador,
  dentro de Freeman) y el dispositivo concreto (término propio, fuera del
  modelo entrenado). RF-02 puede darse por cumplido.
- **Por qué no es parte de Freeman:** el conjunto de datos no tiene
  identificadores de dispositivo; el término es de diseño y se verifica en la
  prueba de punta a punta (Martín, en su navegador habitual, lo lleva).
- **ADR-0045:** una contraseña incorrecta desde un dispositivo conocido cuenta
  como del dueño (pausa), igual que desde una red habitual.
- **Privacidad (Ley 25.326):** es una cookie de seguridad, necesaria para el
  servicio; no identifica a la persona fuera de la organización y no se
  comparte. Una línea en la sección de privacidad alcanza.
- **Consola:** Eventos muestra "Este dispositivo: Habitual / Nuevo" en el
  detalle de cada inicio.
- **Trazabilidad (Cap. 5):** RF-02 suma la prueba unitaria y la escena 2 del e2e.
