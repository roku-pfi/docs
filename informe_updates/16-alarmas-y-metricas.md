# Alarmas, métricas del IdP y resultado de la passkey (RF-16/17, RF-20, RF-11; 2026-10-23)

- **Resultado del paso adicional en el registro (RF-11 / RF-18):** cada decisión
  que pidió passkey guarda cuándo se completó (`step_up_passed_at`, contracts
  0.17.0). La consola muestra "Passkey completada" o "Sin completar" debajo del
  resultado. "Sin completar" es un abandono, o alguien que tenía la contraseña
  pero no la passkey.
- **Métricas del IdP (RF-20 / RNF-04):** inicios por resultado y superficie,
  verificaciones por método y resultado (pasó / falló), y logins decididos por
  el respaldo porque el motor no respondió. Se sirven en un puerto interno del
  clúster, no en el público.
- **Alarmas (RF-16 / RF-17):** ocho reglas de Prometheus.
  - **Riesgo:**
    - credential stuffing (más de 20 contraseñas incorrectas en 5 min);
    - una IP bloqueada;
    - una cuenta atacada desde lugares que no usa;
    - más del 20 % de los inicios bloqueados.
  - **Plataforma:**
    - el motor en respaldo;
    - el IdP sin motor;
    - p95 por encima de 200 ms;
    - el relay de eventos caído.

  Se ven en una tabla de Grafana y, filtradas por organización, como avisos en
  el Resumen de la consola (nunca en rojo, ADR-0030).
- **Review:**
  - requisitos (RF-11, RF-16/17, RF-20: cumplidos);
  - arquitectura / observabilidad (qué se mide y dónde se ve);
  - el apartado de la consola (los avisos en Resumen y el estado de la passkey).
