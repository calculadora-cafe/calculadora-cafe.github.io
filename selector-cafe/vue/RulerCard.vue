<script setup>
// Tarjeta con número rodante + regla en canvas.
// Física: inercia con fricción exponencial (integrador propio, igual que la versión HTML)
// y snap/retorno con animate() de motion-v (spring stiffness 170, damping 24).
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'
import { animate } from 'motion-v'
import NumberRoll from './NumberRoll.vue'

const props = defineProps({
  etiqueta: { type: String, required: true },
  ariaLabel: { type: String, default: '' },
  min: { type: Number, required: true },
  max: { type: Number, required: true },
  modelValue: { type: Number, required: true },
  prefijo: { type: String, default: '' },
  unidad: { type: String, default: '' },
})
const emit = defineEmits(['update:modelValue'])

const SEP = 14          // separación entre ticks (px)
const SIGMA = 3         // ancho de la campana de escala (ticks)
const ESCALA_MAX = 0.7  // centro = 1.7x
const UMBRAL_V = 2      // unidades/s para terminar la inercia

const regla = ref(null)
const canvas = ref(null)
const valor = ref(props.modelValue)

const reducido = () => window.matchMedia('(prefers-reduced-motion: reduce)').matches
const acotar = (v, a, b) => Math.min(b, Math.max(a, v))

let ctx, ancho = 0, alto = 72, dpr = 1
let pos = props.modelValue   // posición visual en unidades
let vel = 0
let modo = 'reposo'          // reposo | arrastre | inercia | resorte
let control = null           // control de animate() activo
let raf = 0, ultimo = 0, ro = null

/* Rubber band estilo iOS */
const dim = () => Math.max(ancho, 1)
const elastico = (u) => ((1 - 1 / ((u * SEP * 0.55) / dim() + 1)) * dim()) / SEP
const elasticoInv = (u) => {
  const v = Math.min(u * SEP, dim() * 0.999)
  return ((dim() / 0.55) * (1 / (1 - v / dim()) - 1)) / SEP
}
const aVisual = (r) => (r < props.min ? props.min - elastico(props.min - r) : r > props.max ? props.max + elastico(r - props.max) : r)
const aCrudo = (p) => (p < props.min ? props.min - elasticoInv(props.min - p) : p > props.max ? props.max + elasticoInv(p - props.max) : p)

function actualizarValor() {
  const v = acotar(Math.round(pos), props.min, props.max)
  if (v !== valor.value) {
    valor.value = v
    emit('update:modelValue', v)
  }
}

function dibujar() {
  if (!ctx) return
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
  ctx.clearRect(0, 0, ancho, alto)
  const cx = ancho / 2, cy = alto / 2
  const desde = Math.max(props.min, Math.ceil(pos - cx / SEP - 1))
  const hasta = Math.min(props.max, Math.floor(pos + cx / SEP + 1))
  ctx.lineCap = 'round'
  ctx.lineWidth = 1.5
  for (let i = desde; i <= hasta; i++) {
    const d = i - pos, x = cx + d * SEP
    const mayor = i % 5 === 0
    const g = Math.exp(-(d * d) / (2 * SIGMA * SIGMA))
    const h = (mayor ? 26 : 16) * (1 + ESCALA_MAX * g)
    const e = Math.min(1, Math.abs(x - cx) / cx)
    ctx.globalAlpha = 0.08 + 0.92 * Math.pow(1 - e, 1.8)
    ctx.strokeStyle = mayor ? '#8A8A8A' : '#C4C4C4'
    ctx.beginPath(); ctx.moveTo(x, cy - h / 2); ctx.lineTo(x, cy + h / 2); ctx.stroke()
  }
  ctx.globalAlpha = 1
  ctx.strokeStyle = '#111'; ctx.lineWidth = 3
  ctx.beginPath(); ctx.moveTo(cx, cy - 32); ctx.lineTo(cx, cy + 32); ctx.stroke()
}

function redibujar() { actualizarValor(); dibujar() }

function ajustarTamano() {
  const el = regla.value
  dpr = window.devicePixelRatio || 1
  ancho = el.clientWidth; alto = 72
  canvas.value.width = Math.round(ancho * dpr)
  canvas.value.height = Math.round(alto * dpr)
  dibujar()
}

function detener() {
  if (control) { control.stop(); control = null }
  if (raf) { cancelAnimationFrame(raf); raf = 0 }
}

// Snap / retorno al límite: spring de motion-v desde pos hasta el entero destino
function resorteHacia(destino) {
  detener()
  modo = 'resorte'
  const obj = acotar(Math.round(destino), props.min, props.max)
  control = animate(pos, obj, {
    type: 'spring', stiffness: 170, damping: 24, mass: 1,
    velocity: vel,
    // con reduced-motion: amortiguación crítica (2·√170 ≈ 26) para evitar rebote
    ...(reducido() ? { damping: 2 * Math.sqrt(170) } : {}),
    onUpdate: (v) => { pos = v; redibujar() },
    onComplete: () => { pos = obj; vel = 0; modo = 'reposo'; control = null; redibujar() },
  })
}

// Inercia con fricción exponencial (integrador propio por rAF)
function inercia() {
  detener()
  modo = 'inercia'
  ultimo = performance.now()
  const tau = reducido() ? 0.12 : 0.35
  const cuadro = (t) => {
    const dt = Math.min((t - ultimo) / 1000, 1 / 30)
    ultimo = t
    vel *= Math.exp(-dt / tau)
    pos += vel * dt
    redibujar()
    if (pos < props.min || pos > props.max) return resorteHacia(pos < props.min ? props.min : props.max)
    if (Math.abs(vel) < UMBRAL_V) return resorteHacia(pos)
    raf = requestAnimationFrame(cuadro)
  }
  raf = requestAnimationFrame(cuadro)
}

/* Arrastre */
let idPuntero = null, xInicio = 0, crudoInicio = 0, muestras = []
function alBajar(e) {
  if (e.pointerType === 'mouse' && e.button !== 0) return
  detener()
  idPuntero = e.pointerId
  regla.value.setPointerCapture(idPuntero)
  modo = 'arrastre'; vel = 0
  xInicio = e.clientX; crudoInicio = aCrudo(pos)
  muestras = [{ t: performance.now(), p: pos }]
}
function alMover(e) {
  if (e.pointerId !== idPuntero || modo !== 'arrastre') return
  pos = aVisual(crudoInicio - (e.clientX - xInicio) / SEP) // izquierda = aumenta
  const t = performance.now()
  muestras.push({ t, p: pos })
  while (muestras.length > 2 && t - muestras[0].t > 120) muestras.shift()
  redibujar()
}
function alSoltar(e) {
  if (e.pointerId !== idPuntero) return
  idPuntero = null
  if (modo !== 'arrastre') return
  const ahora = performance.now()
  const ref0 = muestras.find((m) => ahora - m.t <= 80) // últimos ~80 ms
  let v = 0
  if (ref0 && ahora - ref0.t > 8) v = (pos - ref0.p) / ((ahora - ref0.t) / 1000)
  vel = acotar(v, -80, 80)
  if (pos < props.min || pos > props.max) resorteHacia(pos < props.min ? props.min : props.max)
  else if (Math.abs(vel) > UMBRAL_V * 2) inercia()
  else { vel = 0; resorteHacia(pos) }
}

/* Rueda y teclado: ±1 con snap */
let objetivo = props.modelValue
let ultimaRueda = 0
function ir(v) {
  if (modo === 'arrastre') return
  objetivo = acotar(Math.round(v), props.min, props.max)
  vel = 0
  resorteHacia(objetivo)
}
const base = () => (modo === 'resorte' ? objetivo : Math.round(pos))
function alRueda(e) {
  e.preventDefault() // solo corre con el puntero sobre la regla
  const t = performance.now()
  if (t - ultimaRueda < 40) return
  const delta = Math.abs(e.deltaX) > Math.abs(e.deltaY) ? e.deltaX : e.deltaY
  if (!delta) return
  ultimaRueda = t
  ir(base() + Math.sign(delta))
}
function alTeclado(e) {
  const b = base()
  const mapa = { ArrowLeft: b - 1, ArrowDown: b - 1, ArrowRight: b + 1, ArrowUp: b + 1, Home: props.min, End: props.max }
  if (e.key in mapa) { e.preventDefault(); ir(mapa[e.key]) }
}

const texto = computed(() => `${props.prefijo}${valor.value}${props.unidad ? ' ' + props.unidad : ''}`)

onMounted(() => {
  ctx = canvas.value.getContext('2d')
  regla.value.addEventListener('wheel', alRueda, { passive: false })
  ro = new ResizeObserver(ajustarTamano)
  ro.observe(regla.value)
  ajustarTamano()
})
onBeforeUnmount(() => {
  detener()
  ro?.disconnect()
  regla.value?.removeEventListener('wheel', alRueda)
})
</script>

<template>
  <section class="tarjeta">
    <div class="fila">
      <span class="etiqueta">{{ etiqueta }}</span>
      <div class="valor">
        <span v-if="prefijo" class="prefijo" aria-hidden="true">{{ prefijo }}</span>
        <NumberRoll :value="valor" />
        <span v-if="unidad" class="unidad" aria-hidden="true">{{ unidad }}</span>
      </div>
    </div>
    <div
      ref="regla"
      class="regla"
      tabindex="0"
      role="slider"
      :aria-label="ariaLabel || etiqueta"
      :aria-valuemin="min"
      :aria-valuemax="max"
      :aria-valuenow="valor"
      :aria-valuetext="texto"
      @pointerdown="alBajar"
      @pointermove="alMover"
      @pointerup="alSoltar"
      @pointercancel="alSoltar"
      @keydown="alTeclado"
    >
      <canvas ref="canvas"></canvas>
    </div>
  </section>
</template>

<style scoped>
.tarjeta { background: #fff; border-radius: 28px; padding: 28px; }
.fila { display: flex; align-items: baseline; gap: 16px; margin-bottom: 14px; }
.etiqueta {
  font-family: 'Space Mono', ui-monospace, SFMono-Regular, Menlo, monospace;
  text-transform: uppercase; color: #8A8A8A; font-size: 13px; letter-spacing: 0.14em;
}
.valor {
  display: flex; align-items: baseline;
  font-family: Inter, system-ui, sans-serif;
  font-size: 54px; font-weight: 300; color: #111; line-height: 1.1;
  font-variant-numeric: tabular-nums;
}
.prefijo { margin-right: 0.04em; }
.unidad { font-size: 22px; color: #8A8A8A; margin-left: 6px; font-weight: 400; }
.regla { height: 72px; border-radius: 14px; touch-action: pan-y; cursor: grab; user-select: none; }
.regla:focus { outline: none; }
.regla:focus-visible { outline: 2px solid #111; outline-offset: 3px; }
.regla canvas { display: block; width: 100%; height: 72px; touch-action: pan-y; }
</style>
