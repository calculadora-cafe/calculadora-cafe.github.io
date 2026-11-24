<script setup>
// Las dos tarjetas + línea de agua calculada (café × ratio)
import { computed, ref } from 'vue'
import RulerCard from './RulerCard.vue'

const cafe = ref(15)
const ratio = ref(15)
const agua = computed(() => cafe.value * ratio.value)
</script>

<template>
  <main class="conjunto">
    <div class="tarjetas">
      <RulerCard v-model="cafe" etiqueta="Café" aria-label="Café en gramos" :min="5" :max="40" unidad="g" />
      <RulerCard v-model="ratio" etiqueta="Ratio" aria-label="Ratio" :min="10" :max="20" prefijo="1:" />
    </div>
    <p class="agua" aria-live="polite">Agua: {{ agua }} ml</p>
  </main>
</template>

<style>
html, body { margin: 0; }
body {
  background: #EBEBEB; min-height: 100vh; display: flex; align-items: center; justify-content: center;
  padding: 24px; box-sizing: border-box; font-family: Inter, system-ui, sans-serif; color: #111;
}
* { box-sizing: border-box; }
.conjunto { width: 100%; max-width: 880px; }
.tarjetas { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
@media (max-width: 640px) { .tarjetas { grid-template-columns: 1fr; } }
.agua { margin: 18px 4px 0; font-size: 15px; color: #8A8A8A; font-variant-numeric: tabular-nums; }
</style>
