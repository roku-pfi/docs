# Evaluación sobre ataques no usados para decidir (C1; 2026-11-03)

Finding: `findings/2026-11-03-backtest-untouched-takeovers.md`.

- **Qué se hizo:** los 103 ataques anteriores al conjunto de prueba nunca se
  usaron para elegir nada (Freeman aprende sin etiquetas). Se congeló la
  configuración adoptada (freeman-0.4.0) y se la reentrenó en ventanas
  anteriores, evaluando en la ventana siguiente, con la misma métrica.
- **Resultado:** 40–44 % de ataques detectados al 1 % de falsos positivos (12/30,
  21/48, 12/29; AUC ≈ 0,81), contra 53 % (20/38; AUC 0,889) en el conjunto de
  prueba. Los intervalos se superponen. El modelo anterior (0.3.0) pierde lo
  mismo entre ventanas, así que buena parte de la diferencia es el período (más
  historial por usuario más adelante), no la selección. El orden entre
  modelos se mantiene en todas las ventanas.
- **Capítulo 5, Protocolo:** la frase del 24/10 ("el conjunto de prueba también
  se usó para elegir versiones… evaluación de desarrollo") puede completarse:
  "Una evaluación posterior sobre los 103 ataques anteriores, no usados para
  ninguna elección, da 40–44 % al 1 % de FPR (AUC ≈ 0,81)". Una tabla chica
  con las tres ventanas alcanza.
- **Umbrales:** un umbral fijado al 1 % en un período marca 18–25 % de los
  inicios legítimos del siguiente. Coincide con el 21,4 % de desafíos de la
  comparación de políticas y justifica aprender la población del despliegue y
  arrancar en modo monitor. Puede ir como una oración en la misma sección.
- **No cubierto:** el umbral 0,869 de la regresión logística sigue declarado como
  medido sobre el conjunto de prueba.
- **Para la defensa:** "¿Cómo sabés que el 20/38 no está inflado?" → "Lo
  medimos sobre 103 ataques que no usamos para decidir nada: da 40–44 %. Está
  algo inflado, el orden de los modelos se mantiene."
