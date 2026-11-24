# Calculadora de café filtrado

Calculadora para preparar café filtrado: pones los gramos de café y te da agua, temperatura, vertidos y tiempos.

## Qué incluye

- `index.html`: la calculadora completa en un solo archivo (abre directo en el navegador).
  - Niveles Simple, Intermedio (Personalizada + recetas de campeones: Kasuya 2016, Medina 2023, Peng 2025, Hoffmann) y Avanzado (preguntas escritas: país, variedad, altura, proceso, tueste, especie).
  - Tarjetas de café (5,0–40,0 g) y ratio (1:10,0–1:20,0) con número rodante y regla arrastrable con inercia.
  - Temporizador por vertido, sección "Qué esperar en la taza" (perfil, notas, cómo mover el sabor, recetas para explorar, Aprende).
  - Modo claro y oscuro.
- `selector-cafe/selector-cafe.html`: el componente de tarjetas con regla, aislado y sin librerías.
- `selector-cafe/vue/`: versión Vue 3 + motion-v del componente (sin compilar aún, ver su README).

## Uso

Abre `index.html` en el navegador. No necesita servidor ni dependencias.

## Fuentes

SCA Coffee Standards, Hario (Tetsu Kasuya), Slow Pour Supply (Carlos Medina), Honest Coffee Guide (George Peng), Hario USA (James Hoffmann). Los valores son puntos de partida; el perfil de sabor es una estimación orientativa.
