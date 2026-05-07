<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'

const props = defineProps({
  // Hauptfarbe der Tropfen
  color: {
    type: String,
    default: '#00ff9c'
  },
  // Farbe des führenden (vordersten) Zeichens – etwas heller / weißlich
  headColor: {
    type: String,
    default: '#d6ffe9'
  },
  // Hintergrund-Fade pro Frame (kleinerer Wert = längere Trails)
  fadeAlpha: {
    type: Number,
    default: 0.06
  },
  // Schriftgröße (= Zellengröße)
  fontSize: {
    type: Number,
    default: 24
  },
  // Geschwindigkeit als FPS-Cap
  fps: {
    type: Number,
    default: 15
  },
  // Gesamtdeckkraft des Canvas
  opacity: {
    type: Number,
    default: 0.55
  },
  // CRT-Effekte an / aus
  crt: {
    type: Boolean,
    default: true
  },
  // Stärke der Scanlines (0–1)
  scanlineStrength: {
    type: Number,
    default: 0.35
  },
  // Stärke der Vignette (0–1)
  vignetteStrength: {
    type: Number,
    default: 0.6
  },
  // Stärke des Flicker (0–1, 0 = aus)
  flickerStrength: {
    type: Number,
    default: 0.04
  }
})

const canvasRef = ref(null)

let ctx = null
let animationId = null
let resizeObserver = null
let drops = []
let speeds = []
let columns = 0
let dpr = 1
let lastTime = 0
const frameInterval = () => 1000 / props.fps

const charset =
  'ｱｲｳｴｵｶｷｸｹｺｻｼｽｾｿﾀﾁﾂﾃﾄﾅﾆﾇﾈﾉﾊﾋﾌﾍﾎﾏﾐﾑﾒﾓﾔﾕﾖﾗﾘﾙﾚﾛﾜﾝ' +
  'ABCDEFGHIJKLMNOPQRSTUVWXYZ' +
  '0123456789' +
  '@#$%&*+-=<>[]{}/\\|'

const chars = charset.split('')
const randomChar = () => chars[(Math.random() * chars.length) | 0]

function setup() {
  const canvas = canvasRef.value
  if (!canvas) return

  ctx = canvas.getContext('2d', { alpha: true })
  resize()
}

function resize() {
  const canvas = canvasRef.value
  if (!canvas || !ctx) return

  dpr = Math.min(window.devicePixelRatio || 1, 2)
  const w = canvas.clientWidth
  const h = canvas.clientHeight

  canvas.width = Math.floor(w * dpr)
  canvas.height = Math.floor(h * dpr)
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)

  columns = Math.ceil(w / props.fontSize)

  drops = new Array(columns)
  speeds = new Array(columns)
  for (let i = 0; i < columns; i++) {
    drops[i] = Math.floor(Math.random() * -50)
    speeds[i] = 0.5 + Math.random() * 0.9
  }

  ctx.fillStyle = 'rgba(0, 0, 0, 1)'
  ctx.fillRect(0, 0, w, h)
}

function draw(timestamp) {
  animationId = requestAnimationFrame(draw)

  if (timestamp - lastTime < frameInterval()) return
  lastTime = timestamp

  const canvas = canvasRef.value
  if (!canvas || !ctx) return

  const w = canvas.clientWidth
  const h = canvas.clientHeight

  // Trail-Fade
  ctx.fillStyle = `rgba(0, 0, 0, ${props.fadeAlpha})`
  ctx.fillRect(0, 0, w, h)

  ctx.font = `${props.fontSize}px ui-monospace, "JetBrains Mono", "Fira Code", Menlo, Consolas, monospace`
  ctx.textBaseline = 'top'

  for (let i = 0; i < columns; i++) {
    const x = i * props.fontSize
    const y = drops[i] * props.fontSize
    const ch = randomChar()

    // Kopf: hell mit Glow
    ctx.fillStyle = props.headColor
    ctx.shadowColor = props.color
    ctx.shadowBlur = 8
    ctx.fillText(ch, x, y)

    // Body: vorheriger Schritt in Hauptfarbe
    if (y - props.fontSize >= 0) {
      ctx.shadowBlur = 0
      ctx.fillStyle = props.color
      ctx.fillText(randomChar(), x, y - props.fontSize)
    }

    if (y > h && Math.random() > 0.975) {
      drops[i] = Math.floor(Math.random() * -20)
      speeds[i] = 0.5 + Math.random() * 0.9
    }

    drops[i] += speeds[i]
  }

  ctx.shadowBlur = 0
}

function start() {
  setup()
  lastTime = performance.now()
  animationId = requestAnimationFrame(draw)
}

function stop() {
  if (animationId) {
    cancelAnimationFrame(animationId)
    animationId = null
  }
}

function handleVisibilityChange() {
  if (document.hidden) {
    stop()
  } else if (!animationId) {
    lastTime = performance.now()
    animationId = requestAnimationFrame(draw)
  }
}

onMounted(() => {
  start()
  resizeObserver = new ResizeObserver(() => resize())
  resizeObserver.observe(canvasRef.value)
  document.addEventListener('visibilitychange', handleVisibilityChange)
})

onBeforeUnmount(() => {
  stop()
  if (resizeObserver) resizeObserver.disconnect()
  document.removeEventListener('visibilitychange', handleVisibilityChange)
})
</script>

<template>
  <div class="matrix-rain-wrapper" :class="{ 'crt': crt }" aria-hidden="true">
    <canvas
      ref="canvasRef"
      id="matrix-rain"
      class="matrix-rain"
    />
    <!-- CRT-Overlay-Layers: nur sichtbar wenn crt=true -->
    <div v-if="crt" class="crt-scanlines" />
    <div v-if="crt" class="crt-rgb-shift" />
    <div v-if="crt" class="crt-vignette" />
    <div v-if="crt && flickerStrength > 0" class="crt-flicker" />
  </div>
</template>

<style scoped>
.matrix-rain-wrapper {
  position: fixed;
  inset: 0;
  width: 100vw;
  height: 100vh;
  pointer-events: none;
  overflow: hidden;
  z-index: -1;
  background: #000;
}

.matrix-rain {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  display: block;
  opacity: v-bind(opacity);
}

/* Subtile Bildverbesserung passend zum Röhren-Look */
.crt .matrix-rain {
  filter: contrast(1.05) saturate(1.1);
}

/* Horizontale Scanlines */
.crt-scanlines {
  position: absolute;
  inset: 0;
  pointer-events: none;
  background-image: repeating-linear-gradient(
    to bottom,
    rgba(0, 0, 0, 0) 0px,
    rgba(0, 0, 0, 0) 1px,
    rgba(0, 0, 0, v-bind(scanlineStrength)) 2px,
    rgba(0, 0, 0, v-bind(scanlineStrength)) 3px
  );
  mix-blend-mode: multiply;
  z-index: 2;
  /* Sanft wandernde Scanlines – klassischer Röhren-Look */
  animation: scanline-shift 8s linear infinite;
}

/* Dezente Chromatic-Aberration */
.crt-rgb-shift {
  position: absolute;
  inset: 0;
  pointer-events: none;
  background:
    repeating-linear-gradient(
      90deg,
      rgba(255, 0, 0, 0.025) 0px,
      rgba(0, 255, 0, 0.02) 1px,
      rgba(0, 0, 255, 0.025) 2px,
      transparent 3px,
      transparent 4px
    );
  mix-blend-mode: screen;
  z-index: 3;
  opacity: 0.7;
}

/* Vignette: dunkle Ecken */
.crt-vignette {
  position: absolute;
  inset: 0;
  pointer-events: none;
  background: radial-gradient(
    ellipse at center,
    transparent 40%,
    rgba(0, 0, 0, calc(v-bind(vignetteStrength) * 0.5)) 75%,
    rgba(0, 0, 0, v-bind(vignetteStrength)) 100%
  );
  z-index: 4;
}

/* Flicker: minimales Helligkeits-Wackeln */
.crt-flicker {
  position: absolute;
  inset: 0;
  pointer-events: none;
  background: rgba(255, 255, 255, v-bind(flickerStrength));
  mix-blend-mode: overlay;
  z-index: 5;
  animation: crt-flicker 2.5s steps(3) infinite;
}

@keyframes scanline-shift {
  0% { background-position: 0 0; }
  100% { background-position: 0 100px; }
}

@keyframes crt-flicker {
  0%   { opacity: 0.7; }
  25%  { opacity: 0.5; }
  50%  { opacity: 0.85; }
  75%  { opacity: 0.6; }
  100% { opacity: 0.75; }
}

@media (prefers-reduced-motion: reduce) {
  .matrix-rain {
    display: none;
  }
  .crt-scanlines,
  .crt-flicker {
    animation: none;
  }
}
</style>
