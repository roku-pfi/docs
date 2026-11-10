# Lo que falta en el informe vigente (2026-11-10)

Revisado contra la fuente LaTeX vigente
(`PFI/informe/pfi-informe/chapters/*.tex`, último commit 4171d18, 24/10), no
contra el PDF E90. Esa versión ya incluye el modelo `freeman-0.4.0` /
`logreg-0.2.0`, 20 de 38, la comparación de políticas (21,4 %, 31 de 38), el
bloqueo por origen, la arquitectura multi-tenant y la latencia de 24 ms. Lo que
sigue es lo que todavía no está. Cada punto remite a la nota que tiene el
detalle.

| Dónde | Falta | Nota |
|---|---|---|
| cap. 3, l. 1385–1392 y figura de Efectividad | Efectividad ahora es por organización: los inicios propios con contraseña correcta (sin pasos, con passkey, frenados) y la tabla "Con y sin RokuSec" frente a solo contraseña y passkey siempre. La evaluación con datos públicos queda plegada abajo, igual para todas. Recapturar la figura | 26 |
| cap. 5, l. 128 (escena 8) | "La cuenta queda en pausa y el intento siguiente se bloquea": ahora el quinto fallo ya responde con la pausa, y cualquier intento posterior, correcto o no, recibe la misma pantalla. Así no se confirma la contraseña (oráculo cerrado; vale una línea en amenazas) | 26 |
| cap. 3 (notificación al usuario) y cap. 5 | El aviso "Alguien intentó ingresar a tu cuenta": al iniciar sesión, y con la sesión abierta como aviso en el banco (consulta cada 10 s), con cambio de contraseña que pide la actual y cierra las otras sesiones. Texto (10/11): "Detectamos un ingreso sospechoso desde <país>, <fecha y hora>. Te recomendamos cambiar tu contraseña." Nunca dice cómo se frenó el intento (passkey o bloqueo) | 26 |
| cap. 3, política y consola | Cuando una regla sube la acción, el nivel mostrado sube con ella; el puntaje sigue siendo el de Freeman | 26 |
| cap. 5, protocolo y resultados | La evaluación sobre los 103 ataques no usados para elegir: 40–44 % al 1 % de FPR (AUC ≈ 0,81), contra 53 % en el conjunto de prueba. El umbral 0,869 de la logística se sostiene en tres ventanas anteriores (0,4–1,3 % de FPR, 44–55 % de detección) | 23 |
| cap. 3, arquitectura asíncrona | Reintentos de los consumidores (0,5, 2 y 8 s) y cola de eventos fallidos con reproceso; métricas y alarmas de atraso | 25 |
| cap. 3 (RF-02) | Identificador de dispositivo concreto, además de la clase de dispositivo | 21 |
| cap. 5, l. 239 (carga) | Opcional: la última corrida en AWS (2026-11-09) dio 18 603 evaluaciones, 0 errores, p95 24 ms, de 3 a 5 réplicas. La fila 14 (17 298) sigue siendo válida | 12 |

## Números que van a citar los videos

Todos coinciden con el informe vigente, salvo los de Efectividad de Roku Bank,
que salen de la sesión de grabación.

- **Encuesta, n = 161:** 82,0 % inicia sesión al menos a diario; 57,8 % siente
  fricción frecuente; 63,4 % prefiere la verificación adaptativa.
- **Comparación de políticas:** el motor adaptativo frena 31 de 38 ataques, con
  21,4 % de fricción (2–3 % para quien vuelve) y 0,02 % de legítimos bloqueados.
- **Conjunto de datos:** 31,27 millones de inicios, 4,3 millones de usuarios,
  141 ataques.
- **Carga en AWS:** p95 de 24 ms, sin errores, de 3 a 5 réplicas.
