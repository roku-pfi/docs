# "Passkey siempre" por usuario y por aplicación (RF-08, ADR-0048; 2026-11-03)

- **Qué cambió:** además de la política base de la organización, la consola
  permite marcar un usuario o una aplicación como "Passkey siempre" (menú de
  cada fila en Usuarios y Aplicaciones; etiqueta azul con la explicación al
  pasar el cursor).
- **Quién lo aplica:** el IdP, también con el motor apagado o caído. El motor
  lo registra como motivo ("passkey siempre (usuario o aplicación)") sin cambiar
  su propio veredicto; por eso el desafío no dice "Inicio poco habitual".
- **RF-08 / trazabilidad (Cap. 5):** pasa a cumplido en los tres niveles
  (organización, aplicación, usuario). Pruebas unitarias en IdP y PDP.
- **Capítulo 3, política:** donde se describe "Passkey siempre" como política
  base, agregar que también puede fijarse por usuario o por aplicación.
