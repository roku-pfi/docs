# La población es por organización (C3, ADR-0046; 2026-11-03)

- **Qué cambió:** cada organización tiene su propia población de inicios
  aprendidos (`rba:population:<tenant_id>`). Los inicios de Roku Bank ya no
  mueven los puntajes de Kaze Labs, ni al revés. El modelo entrenado (el prior)
  sigue siendo uno para todos.
- **Aprendizaje en línea, "La población es la del despliegue":** la frase del
  24/10 ("es una para todo el despliegue… separarla por organización queda como
  trabajo futuro") ya no es cierta. Pasa a decir que cada organización tiene la
  suya, y que una organización nueva arranca sólo con el prior de entrenamiento
  hasta acumular inicios propios (con κ = 100, unos 100 inicios igualan al prior).
- **Síntesis de entrevistas:** lo que pidió el arquitecto de seguridad (no
  aprender entre organizaciones) queda cumplido para la población; puede decirse.
- **RF-20 (aislamiento):** ya cubre también las frecuencias de la población.
- **Trabajo futuro:** sacar "población por organización"; queda "modelo por
  organización".
- **Figuras:** §3.6.4 / `07-valkey-keys.png` muestra `rba:population:{tenant_id}`;
  la Figura 3.16 (flujo 1 de 2) lee `HMGET rba:population:{tenant_id}`.
