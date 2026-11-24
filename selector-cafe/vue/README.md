# Selector café + ratio (Vue 3 + motion-v)

Sin probar: se escribió sin poder instalar dependencias.

```bash
npm i vue motion-v
```

Copia `NumberRoll.vue`, `RulerCard.vue` y `App.vue` a `src/` (por ejemplo, en un proyecto Vite con `@vitejs/plugin-vue`) y usa `App.vue` como raíz. Opcional: carga Inter y Space Mono desde Google Fonts.

```vue
<script setup>
import { ref } from 'vue'
import RulerCard from './RulerCard.vue'
const cafe = ref(15)
</script>
<template>
  <RulerCard v-model="cafe" etiqueta="Café" aria-label="Café en gramos" :min="5" :max="40" unidad="g" />
</template>
```

- `NumberRoll.vue` (prop `value`): columnas de dígitos con `motion.span` y spring; las decenas entran/salen con `AnimatePresence`.
- `RulerCard.vue` (`v-model`, `etiqueta`, `min`, `max`, `prefijo`, `unidad`): regla en canvas. Inercia con integrador propio (fricción exponencial); snap y retorno del rubber band con `animate()` de motion-v.
