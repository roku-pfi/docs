# Aviso de inicio inusual y respuestas sin señales (RF-17; 2026-11-03)

- **Qué cambió:** cuando el motor pide la passkey, el desafío muestra una
  etiqueta "Inicio poco habitual"; al pasar el cursor explica que el inicio no
  se parece a los habituales de la cuenta. No aparece si la organización pide
  passkey siempre ni si el motor no respondió.
- **Respuestas al navegador:** `/login`, `/console/login` y las verificaciones
  ya no devuelven puntaje, nivel, acción ni razones (antes viajaban en la
  respuesta JSON, visibles en las herramientas del navegador). Sólo queda la
  pausa por fallos propios, para el mensaje "esperá unos minutos". El detalle
  sigue en la consola. Contrato `LoginResponse.unusual` (contracts 0.19.0).
- **Tabla de trazabilidad (Cap. 5):** RF-17 pasa de "En parte: no menciona que
  el inicio es inusual" a **Cumplido**. Ajustar el párrafo de cierre que lo
  lista junto a HU-03 como parcialmente cumplido.
- **Cap. 3, transparencia (RNF-02):** "el usuario final recibe un aviso
  genérico, sin la señal que lo disparó" ahora es literal; nada que cambiar.
- **Figura 3.x (captura del desafío):** si se quiere, recapturar con la etiqueta.
