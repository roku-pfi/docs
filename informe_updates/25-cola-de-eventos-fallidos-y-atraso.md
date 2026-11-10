# Eventos fallidos y atraso del aprendizaje (Fase 4/5; 2026-11-04)

- **Qué cambió:** si el servicio de perfiles o el de auditoría no puede aplicar
  un evento, el evento ya no se descarta: queda en una cola aparte
  (`<cola>.dlq`) y se puede reprocesar una vez corregida la causa
  (`python -m rba_event_publisher.dlq replay <cola>`). Los consumidores son
  idempotentes por `event_id`, así que reprocesar es seguro.
- **Reintentos (agregado 05/11):** antes de mandar un evento a la cola de
  fallidos, el consumidor lo reintenta en el lugar tras 0,5, 2 y 8 segundos.
  Un corte breve de la base de datos o de Valkey ya no deja eventos para
  reprocesar a mano; un evento mal formado no se reintenta. Al reintentar en
  el lugar se conserva el orden de llegada (una decisión y su confirmación).
  Probado en k3d cortando la base durante un inicio de sesión: tres
  reintentos, aplicado al cuarto, cola de fallidos vacía.
- **Métricas nuevas (Grafana, tablero "RokuSec · motor de decisión"):** eventos del diario sin
  publicar y antigüedad del más viejo; profundidad de cada cola, incluidas las
  de fallidos; atraso p95 entre la decisión y su efecto en el perfil.
- **Alarmas nuevas:** `EventLagHigh` (hay decisiones de hace más de un minuto
  sin publicar) y `EventsDeadLettered` (hay eventos fallidos esperando). Se ven
  en Grafana y en el Resumen de la consola, como las de RF-16.
- **Capítulo 3, arquitectura asíncrona:** donde se dice que un fallo del
  consumidor descarta el mensaje, o que la DLQ queda pendiente, pasa a hecho.
  En la figura de la cola de mensajes alcanza con un recuadro "cola de
  fallidos" junto a cada consumidor.
- **Pruebas / CI (Cap. 5):** las pruebas de extremo a extremo corren además en
  GitHub Actions sobre un clúster k3d nuevo cada noche (`rba-infra`,
  workflow `e2e`), construyendo todos los servicios desde `main`.
