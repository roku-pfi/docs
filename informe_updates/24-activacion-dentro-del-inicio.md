# Activación de la passkey dentro del inicio de sesión (ADR-0037; 2026-11-04)

- **Qué cambió:** quien todavía no tiene passkey y recibe "Necesitás una
  passkey" ya no va a otra página: escribe ahí mismo el código de activación,
  crea la passkey y el inicio se vuelve a evaluar solo (con la contraseña que
  ya había escrito). La página `/activate` sigue existiendo.
- **Consola:** al generar el código, además del código se muestra un enlace
  (`/activate#CÓDIGO`) para copiar. El código va después de `#`: el navegador no
  lo envía, así que no queda en registros del servidor ni de CloudFront. El
  enlace solo no alcanza: también pide la contraseña.
- **Seguridad, sin cambios:** las dos únicas puertas de ADR-0037 siguen siendo
  las mismas (inicio de bajo riesgo, o código emitido por un operador + la
  contraseña). Un paso adicional nunca se convierte en alta por sí solo.
- **Capítulo 3, alta de usuarios / passkeys:** reemplazar "el usuario entra a
  /activate con el código" por "el usuario escribe el código en la pantalla de
  inicio que se lo pide, o abre el enlace que le pasa el operador".
- **Pruebas (Cap. 5):** nueva prueba de extremo a extremo
  `test_new_customer_activates_inside_the_login` (cliente nuevo con "passkey
  siempre", sin passkey: código en el inicio, crea la passkey, entra al banco).
