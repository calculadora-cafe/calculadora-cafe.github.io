<script setup>
// Contador de dígitos: cada dígito es una columna 0–9 que se traslada en Y con un spring.
// La columna de decenas aparece/desaparece con AnimatePresence (ancho + opacidad).
import { computed, ref, watch } from 'vue'
import { motion, AnimatePresence } from 'motion-v'

const props = defineProps({
  value: { type: Number, required: true },
})

const ALTO = 1.1 // alto de cada celda en em

// Con prefers-reduced-motion se usa un spring sin rebote
const reducido =
  typeof window !== 'undefined' && window.matchMedia('(prefers-reduced-motion: reduce)').matches
const spring = { type: 'spring', bounce: reducido ? 0 : 0.2, duration: 0.55 }

const entero = computed(() => Math.round(props.value))
const unidades = computed(() => entero.value % 10)
const hayDecenas = computed(() => entero.value >= 10)

// Al bajar de 10 se conserva el último dígito de decenas mientras la columna se colapsa
const decenas = ref(Math.floor(entero.value / 10))
watch(entero, (v) => {
  if (v >= 10) decenas.value = Math.floor(v / 10)
})

const digitos = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
</script>

<template>
  <!-- Texto accesible: los dígitos visuales van en aria-hidden -->
  <span class="roll" aria-live="polite">
    <span class="sr">{{ entero }}</span>
    <span class="roll-visual" aria-hidden="true">
      <AnimatePresence>
        <motion.span
          v-if="hayDecenas"
          key="decenas"
          class="roll-envoltorio"
          :initial="{ width: 0, opacity: 0 }"
          :animate="{ width: '1ch', opacity: 1 }"
          :exit="{ width: 0, opacity: 0 }"
          :transition="{ type: 'spring', bounce: 0, duration: 0.4 }"
        >
          <span class="roll-col">
            <motion.span
              class="roll-pila"
              :animate="{ y: `${-decenas * ALTO}em` }"
              :transition="spring"
            >
              <span v-for="d in digitos" :key="d" class="roll-celda">{{ d }}</span>
            </motion.span>
          </span>
        </motion.span>
      </AnimatePresence>

      <span class="roll-col">
        <motion.span
          class="roll-pila"
          :animate="{ y: `${-unidades * ALTO}em` }"
          :transition="spring"
        >
          <span v-for="d in digitos" :key="d" class="roll-celda">{{ d }}</span>
        </motion.span>
      </span>
    </span>
  </span>
</template>

<style scoped>
.roll { display: inline-flex; font-variant-numeric: tabular-nums; }
.roll-visual { display: inline-flex; align-items: baseline; }
.roll-envoltorio { display: block; clip-path: inset(0); flex: none; }
.roll-col { display: block; width: 1ch; height: 1.1em; clip-path: inset(0); text-align: center; flex: none; }
.roll-pila { display: block; will-change: transform; }
.roll-celda { display: block; height: 1.1em; line-height: 1.1; }
.sr { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0 0 0 0); white-space: nowrap; }
</style>
