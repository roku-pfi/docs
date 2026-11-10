# Correcciones tras la prueba manual en AWS (2026-11-09)

Recorrido completo a mano sobre AWS. Lo que cambió:

- **La pausa ya no confirma la contraseña (seguridad, ADR-0045).** Antes,
  después de varios intentos fallidos, las contraseñas incorrectas decían
  "incorrecta" y solo la correcta decía "Demasiados intentos". Eso le
  confirmaba la contraseña a quien estaba probando. Ahora el intento que
  alcanza el umbral ya responde con la pausa (o con el bloqueo de la IP), y lo
  mismo cada intento siguiente, correcto o no. Para el capítulo de amenazas:
  es un oráculo de contraseña cerrado, un caso concreto del principio de no
  revelar (ADR-0023).
- **Aviso al dueño de la cuenta (nuevo).** Si alguien usa la contraseña
  correcta y lo frena el motor (un paso de passkey que nadie completó, o un
  bloqueo), en el próximo inicio de sesión el dueño ve "Alguien intentó
  ingresar a tu cuenta", con cuándo y desde dónde, y puede cambiar la
  contraseña ahí mismo (pide la actual y cierra las demás sesiones). Si el
  paso de passkey se completa, era el dueño y no hay aviso. "Passkey siempre"
  no genera avisos. Sin correo electrónico: queda como trabajo futuro.
- **Aviso con la sesión abierta (agregado).** Si el dueño está dentro del
  banco cuando ocurre el intento, lo ve en el momento: un aviso arriba de la
  página (el banco consulta al IdP cada 10 s) con "Cambiar contraseña" (en el
  mismo banco) y "Entendido". Un paso de passkey sin completar cuenta como
  intento después de un minuto; un bloqueo, en el acto.
- **Nivel de riesgo y reglas.** Cuando una regla (viaje imposible, VPN, IP
  bloqueada, segunda opinión) sube la acción, el nivel mostrado sube con ella
  (pedir passkey ⇒ al menos Medio). El puntaje sigue siendo el de Freeman. En
  la consola un viaje imposible ya no dice "Bajo".
- **Consola.** "Robos frenados" pasa a "Intrusiones frenadas" (toma de
  control de cuenta, ATO). El gráfico del límite de Bajo se muestrea cada 0,05
  y se dibuja en escalones: se ve que todo el efecto está en los primeros
  inicios de cada usuario (sin historial), que puntúan entre 0,45 y 0,50. La
  marca "actual" lee el límite de la política del tenant. La prueba de carga
  del presentador ya no aparece en Resumen ni en Eventos (un interruptor
  "Carga" la muestra). Los menús de acciones ya no cortan el texto.
- **Efectividad por organización (segunda pasada).** La página ya no muestra
  los 34 mil inicios del conjunto público como si fueran de la organización:
  muestra los inicios propios con contraseña correcta en el período (7/30/90
  días), qué hizo el motor (sin pasos, con passkey, frenados) y una tabla
  "Con y sin RokuSec" frente a solo contraseña y passkey siempre. La
  evaluación con datos públicos queda plegada abajo, rotulada como igual para
  todas. Para el informe: distinguir evidencia offline (capítulo de
  resultados) de la medición en operación (consola).
- **Banco de demostración.** Cada cliente ve sus propias cuentas (inventadas,
  estables entre visitas); Martín conserva las del guion.

**Dónde tocar el informe:** modelo de amenazas (oráculo de contraseña),
requisitos de notificación al usuario, figuras de la consola si muestran
"Robos frenados" o el gráfico anterior.
