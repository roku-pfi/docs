# Números vigentes: lista por página del informe (2026-11-09)

Los videos van a citar los números actuales del sistema. Esta lista reúne,
en el orden del informe (páginas de `e90.pdf`), lo que quedó desactualizado,
para que el informe y los videos digan lo mismo. Cada punto remite a la nota
que tiene el detalle; esta nota no agrega cambios nuevos.

## Lista por página

| Pág. | Dice hoy | Ahora | Nota |
|---|---|---|---|
| 8 (§1.2.2) | La arquitectura multi-tenant figura como excluida | Incluida: Roku Bank y Kaze Labs, aisladas por `tenant_id`, cada una con su política y su modo (monitor o activo) | 02 |
| 61 (Fig. 3.11) | `freeman-0.2.0`; umbral de la logística 0,903; "12 fallos, la banda empieza en 10" bloquea | Recapturar Eventos. El modelo es `freeman-0.4.0`, el umbral 0,869 y los fallos bloquean el origen, no la cuenta | 09, 10, 13, 17 |
| 62 (Fig. 3.12) | Bandas 0,20 / 0,45 / 0,70 e interruptor "modo de solo monitoreo" | Recapturar Política: política base (passkey siempre o solo contraseña) más modo del motor (monitor o activo) | 02, 17 |
| 68, 73–74, 121, 125 (§3.5.2, §3.6.3, glosario) | Decisión y `outbox` en la misma transacción de Postgres | Postgres ya no está en el camino de la decisión. El motor escribe en un diario en Valkey; el event-publisher llena la tabla de decisiones y publica en RabbitMQ. Los consumidores reintentan a los 0,5, 2 y 8 s y después mandan a una cola de fallidos con reproceso | 12, 25 |
| 78 (Tabla 3.4) | Freeman `freeman-0.2.0` (β = 5, con la hora); logística `logreg-0.1.0` | Freeman `freeman-0.4.0` híbrido: la red como jerarquía mundo → país → ASN → IP, enraizada en la población; dispositivo, SO y navegador planos, por familia; la hora sale del puntaje. Logística `logreg-0.2.0`. RF y LightGBM quedan solo como referencia | 09, 10 |
| 78 (§3.7.5) | "La logística queda 40 veces por encima; Freeman, 5 veces" (PR-AUC) | Logística: PR-AUC 0,0387, unas 47 veces la prevalencia. Freeman 0.4.0: regenerar el PR-AUC con `ml.hierarchical` antes de citarlo; no está en los reportes actuales | 09, 10 |
| 79 (Tabla 3.5) | Logística 0,8725 / 15 de 38; Freeman 0,7790 / 4 de 38 | Freeman 0.4.0: ROC-AUC 0,889 y recall 0,526 (20 de 38) al 1 % de FPR. Logística 0.2.0: ROC-AUC 0,865 y recall 0,474 (18 de 38), con 0,78 % de FPR y 0,82 % de desafíos | 09, 10 |
| 79 (texto bajo la Tabla 3.5) | "Servir solo Freeman reduciría la detección a casi un cuarto" | Ya no vale: Freeman detecta igual o más que la logística. La complementariedad sigue ("lugar nuevo, dispositivo ordinario"), pero el argumento pasa a ser qué ataques distintos ve cada uno, no cuántos | 10 |
| 79 (Tabla 3.6) y 79–80 | Coeficientes de `logreg-0.1.0`; ejemplo 0,64 / 0,94 contra el umbral 0,90; umbral 0,9028 | Regenerar los coeficientes y el ejemplo con `logreg-0.2.0`. El umbral es 0,869 y se verificó en tres ventanas anteriores: 0,4–1,3 % de FPR y 44–55 % de detección | 09, 23 |
| 79 (§3.7.5) | "Tres fallos en 24 h exigen reautenticación; diez, bloquean" | Por origen: 10 fallos en 15 min bloquean esa IP 30 min, incluso con la contraseña correcta. Los fallos desde contextos que el usuario nunca usó le piden passkey en su próximo inicio, nunca lo bloquean. Solo los fallos propios desde una red conocida (5 en 15 min) pausan la cuenta 15 min, y la pausa aparece desde el intento que alcanza el umbral: no confirma la contraseña | 13, 26 |
| Cap. 5, resultados | Sin comparación de políticas sobre el modelo actual | Sobre 34 193 inicios legítimos y 38 ataques: solo contraseña 0 % de fricción y 0 de 38 frenados; passkey siempre 100 % y 38 de 38; adaptativa 21,44 % (IC 21,0–21,9) y 31 de 38 (81,6 %, IC 66,6–90,8), con 0,02 % de legítimos bloqueados. Por historial: primer inicio 100 %; 1–2 inicios previos 2,74 %; 3–9, 2,14 %; 10 o más, 3,01 % | 10 |
| Cap. 5, protocolo | El conjunto de prueba se usó para elegir versiones | Sobre los 103 ataques no usados para elegir: 40–44 % al 1 % de FPR (AUC ≈ 0,81), contra 53 % en el conjunto de prueba | 23 |
| Cap. 5, RNF-07 | Sin medición en la nube con el camino actual | En AWS, en la última corrida (2026-11-09): 18 603 evaluaciones, 0 errores, p95 de 24 ms medido en el motor, réplicas de 3 a 5 | 12 |
| Cap. 3 (RF-17) y amenazas | Aviso de inicio inusual parcial; sin aviso de intentos al dueño | Aviso "Inicio poco habitual" en el desafío. "Alguien intentó ingresar a tu cuenta" al iniciar sesión y, con la sesión abierta, como aviso en el banco, con cambio de contraseña | 20, 26 |
| Consola (figuras) | "Robos frenados", gráfico anterior, 34 mil inicios para todas las organizaciones | "Intrusiones frenadas"; Efectividad con los inicios propios de cada organización y la tabla "Con y sin RokuSec"; la evaluación offline aparece plegada aparte | 26 |

## Números que van a citar los videos

Para que coincidan con lo corregido arriba:
- **Detención:** el motor adaptativo frena 31 de 38 ataques con contraseña correcta.
- **Fricción:** 21,4 % sobre los inicios legítimos, y 2–3 % para quien ya inició sesión antes.
- **Bloqueos de legítimos:** 0,02 %.
- **Carga en AWS:** p95 de 24 ms, sin errores, de 3 a 5 réplicas.
- **Conjunto de datos:** 31,27 millones de inicios, 4,3 millones de usuarios, 141 ataques.
- **Encuesta (sin cambios):** n = 157; 82,8 % inicia sesión a diario; 58,6 % siente fricción frecuente; 63,1 % prefiere la verificación adaptativa.
