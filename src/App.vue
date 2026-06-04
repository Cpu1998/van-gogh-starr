<template>
  <div class="app">
    <div class="controls">
      <button class="btn" @click="toggleAudio">
        {{ isPlaying ? '🔇 关闭音乐' : '🎵 开启音乐律动' }}
      </button>
      <label class="btn upload-btn">
        📁 上传音乐
        <input type="file" accept="audio/*" @change="handleAudioUpload" hidden />
      </label>
      <span class="hint">梵高星空 · 天体轮廓</span>
    </div>
    <canvas ref="canvasRef"></canvas>
    <div class="info">
      <span>✨ 星 · 月亮 · 旋涡 · 流动天空</span>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const canvasRef = ref<HTMLCanvasElement | null>(null)
const isPlaying = ref(false)

let ctx: CanvasRenderingContext2D | null = null
let animId = 0
let width = 0
let height = 0
let skyH = 0  // sky region height (top 65%)

// ==================== Audio ====================
let audioCtx: AudioContext | null = null
let analyser: AnalyserNode | null = null
let dataArray: Uint8Array = new Uint8Array(0)
let audioSource: MediaElementAudioSourceNode | MediaStreamAudioSourceNode | null = null
let audioElement: HTMLAudioElement | null = null
let micStream: MediaStream | null = null

async function initAudioFromFile(file: File) {
  cleanupAudio()
  audioCtx = new AudioContext()
  analyser = audioCtx.createAnalyser()
  analyser.fftSize = 512
  dataArray = new Uint8Array(analyser.frequencyBinCount)
  audioElement = new Audio()
  audioElement.src = URL.createObjectURL(file)
  audioElement.crossOrigin = 'anonymous'
  audioElement.loop = true
  audioSource = audioCtx.createMediaElementSource(audioElement)
  audioSource.connect(analyser)
  analyser.connect(audioCtx.destination)
  await audioElement.play()
  isPlaying.value = true
}

async function initMicAudio() {
  cleanupAudio()
  audioCtx = new AudioContext()
  analyser = audioCtx.createAnalyser()
  analyser.fftSize = 512
  dataArray = new Uint8Array(analyser.frequencyBinCount)
  micStream = await navigator.mediaDevices.getUserMedia({ audio: true })
  audioSource = audioCtx.createMediaStreamSource(micStream)
  audioSource.connect(analyser)
  isPlaying.value = true
}

function cleanupAudio() {
  if (audioElement) { audioElement.pause(); audioElement.src = ''; audioElement = null }
  if (micStream) { micStream.getTracks().forEach(t => t.stop()); micStream = null }
  if (audioCtx && audioCtx.state !== 'closed') { audioCtx.close() }
  audioCtx = null; analyser = null; audioSource = null
  isPlaying.value = false
}

async function toggleAudio() {
  if (isPlaying.value) { cleanupAudio() } else { await initMicAudio() }
}

function handleAudioUpload(e: Event) {
  const input = e.target as HTMLInputElement
  const file = input.files?.[0]
  if (file) initAudioFromFile(file)
}

function getAudioEnergy() {
  if (!analyser || !dataArray.length) return { bass: 0, mid: 0, high: 0, avg: 0 }
  analyser.getByteFrequencyData(dataArray as Uint8Array<ArrayBuffer>)
  const len = dataArray.length
  const third = Math.floor(len / 3)
  let bass = 0, mid = 0, high = 0
  for (let i = 0; i < third; i++) bass += dataArray[i]
  for (let i = third; i < third * 2; i++) mid += dataArray[i]
  for (let i = third * 2; i < len; i++) high += dataArray[i]
  bass /= third * 255; mid /= third * 255; high /= (len - third * 2) * 255
  return { bass, mid, high, avg: (bass + mid + high) / 3 }
}

// ==================== Color Palette (Van Gogh's actual colors) ====================
// Sky: deep indigo blue → cobalt blue → Prussian blue
// Stars/Moon: cadmium yellow → white-yellow
// Highlights: teal/turquoise touches
const INDIGO     = { r: 15, g: 10, b: 60 }     // deep sky
const COBALT     = { r: 30, g: 50, b: 140 }     // mid sky blue
const PRUSSIAN   = { r: 20, g: 35, b: 95 }      // dark blue
const TEAL       = { r: 50, g: 160, b: 150 }    // turquoise touches
const CAD_YELLOW = { r: 255, g: 220, b: 50 }    // cadmium yellow (stars)
const PALE_YELLOW= { r: 255, g: 245, b: 140 }   // pale yellow (star halos)
const WHITE_YELLOW= { r: 255, g: 252, b: 220 }  // near-white yellow

function lerpColor(a: {r:number;g:number;b:number}, b2: {r:number;g:number;b:number}, t: number) {
  t = Math.max(0, Math.min(1, t))
  return { r: Math.round(a.r+(b2.r-a.r)*t), g: Math.round(a.g+(b2.g-a.g)*t), b: Math.round(a.b+(b2.b-a.b)*t) }
}

function colorStr(c: {r:number;g:number;b:number}, alpha: number) {
  return `rgba(${c.r},${c.g},${c.b},${alpha})`
}

// ==================== THE GREAT SWIRL ====================
// Van Gogh's central vortex: a large spiral in the middle-left of sky
// It's the most iconic feature - concentric flowing curves spiraling inward

interface SpiralRing {
  baseRadius: number
  startAng: number
  sweep: number   // how much of the circle this arc covers
  color: {r:number;g:number;b:number}
  alpha: number
  width: number
  dir: number  // 1 or -1
  speed: number
}

interface StarHalo {
  rings: SpiralRing[]
}

interface FlowWave {
  yBase: number     // normalized y position in sky
  amplitude: number
  frequency: number
  phase: number
  color: {r:number;g:number;b:number}
  alpha: number
  strokeW: number
}

let th = 0
let mainVortex: SpiralRing[] = []
let secondaryVortex: SpiralRing[] = []
let stars: { x:number; y:number; size:number; halo: StarHalo; coreColor:{r:number;g:number;b:number} }[] = []
let moonRings: SpiralRing[] = []
let flowWaves: FlowWave[] = []

function rand(a: number, b: number) { return Math.random() * (b - a) + a }

function makeSpiralRings(cx: number, count: number, maxR: number, colorFrom: {r:number;g:number;b:number}, colorTo: {r:number;g:number;b:number}, baseSpeed: number): SpiralRing[] {
  const rings: SpiralRing[] = []
  for (let i = 0; i < count; i++) {
    const t = i / (count - 1)
    const r = maxR * (0.15 + t * 0.85)  // from inner to outer
    // Each ring is an arc covering most of the circle, with gaps creating the spiral look
    const sweep = rand(Math.PI * 1.2, Math.PI * 1.8)  // 60-90% of circle
    const startAng = rand(0, Math.PI * 2)
    const color = lerpColor(colorFrom, colorTo, t)
    const alpha = (0.2 + t * 0.3) + rand(-0.05, 0.05)
    const w = (maxR * 0.04) * (1 - t * 0.4)  // slightly thinner outer rings
    rings.push({
      baseRadius: r,
      startAng,
      sweep,
      color,
      alpha: Math.max(0.1, Math.min(0.7, alpha)),
      width: Math.max(1, w),
      dir: Math.random() < 0.7 ? 1 : -1,
      speed: baseSpeed * (0.5 + t * 0.8) * (Math.random() < 0.5 ? 1 : 0.8),
    })
  }
  return rings
}

function makeStarHalo(size: number): StarHalo {
  const rings: SpiralRing[] = []
  const ringCount = Math.floor(8 + size * 8)
  const maxR = size * 25
  for (let i = 0; i < ringCount; i++) {
    const t = i / (ringCount - 1)
    const r = maxR * (0.2 + t * 0.8)
    const sweep = rand(Math.PI * 1.5, Math.PI * 2)
    const startAng = rand(0, Math.PI * 2)
    // Stars glow yellow-white
    const color = lerpColor(PALE_YELLOW, CAD_YELLOW, t)
    const alpha = (0.6 - t * 0.4)
    rings.push({
      baseRadius: r,
      startAng,
      sweep,
      color,
      alpha: Math.max(0.05, alpha),
      width: Math.max(0.5, size * 3 * (1 - t * 0.6)),
      dir: Math.random() < 0.5 ? 1 : -1,
      speed: 0.3 + t * 0.3,
    })
  }
  return { rings }
}

function initScene() {
  mainVortex = []
  secondaryVortex = []
  stars = []
  moonRings = []
  flowWaves = []

  // ===== MAIN VORTEX =====
  // Position: roughly center of the sky, slightly left
  // In the painting it's the massive spiral taking up ~30% of sky
  // Center at approximately (35%, 40%) of sky region
  mainVortex = makeSpiralRings(
    0,
    55,      // 55 concentric arc rings
    Math.min(width, skyH) * 0.22,  // large radius
    lerpColor(COBALT, TEAL, 0.3),  // inner: blue-teal
    lerpColor(INDIGO, COBALT, 0.4), // outer: deeper blue
    0.4      // base speed
  )

  // ===== SECONDARY VORTEX =====
  // Smaller swirl to the right of main vortex, around (62%, 28%)
  secondaryVortex = makeSpiralRings(
    0,
    30,
    Math.min(width, skyH) * 0.10,
    lerpColor(COBALT, TEAL, 0.4),
    PRUSSIAN,
    0.3
  )

  // ===== MOON =====
  // Upper right corner: bright crescent moon with radiating rings
  // Position: (82%, 15%) of sky region
  // Moon has distinct concentric rings radiating outward, like a sun
  const moonMaxR = Math.min(width, skyH) * 0.08
  for (let i = 0; i < 25; i++) {
    const t = i / 24
    const r = moonMaxR * (0.1 + t * 0.9)
    const sweep = rand(Math.PI * 1.4, Math.PI * 1.9)
    const startAng = rand(0, Math.PI * 2)
    const color = lerpColor(WHITE_YELLOW, CAD_YELLOW, t)
    const alpha = 0.7 - t * 0.45
    moonRings.push({
      baseRadius: r,
      startAng,
      sweep,
      color,
      alpha: Math.max(0.1, alpha),
      width: Math.max(1, moonMaxR * 0.03 * (1 - t * 0.5)),
      dir: Math.random() < 0.5 ? 1 : -1,
      speed: 0.15 + t * 0.15,
    })
  }

  // ===== STARS with halos =====
  // Van Gogh painted several prominent stars, each with concentric ring halos
  // Positions based on actual painting:

  const starDefs = [
    // Upper-left bright star (very prominent in the painting)
    { x: 0.12, y: 0.18, size: 1.2, color: CAD_YELLOW },
    // Star in upper-center area
    { x: 0.28, y: 0.12, size: 0.8, color: PALE_YELLOW },
    // Star above the main vortex
    { x: 0.40, y: 0.15, size: 1.0, color: CAD_YELLOW },
    // Star to the right of main vortex
    { x: 0.55, y: 0.35, size: 0.9, color: PALE_YELLOW },
    // Star near upper-right, below moon
    { x: 0.72, y: 0.30, size: 1.1, color: CAD_YELLOW },
    // Star near the horizon on the right
    { x: 0.80, y: 0.50, size: 0.7, color: PALE_YELLOW },
    // Small star far left
    { x: 0.06, y: 0.35, size: 0.5, color: PALE_YELLOW },
    // Small star top area
    { x: 0.48, y: 0.08, size: 0.6, color: PALE_YELLOW },
    // Star between the two vortices
    { x: 0.50, y: 0.22, size: 0.7, color: CAD_YELLOW },
    // Star near moon
    { x: 0.88, y: 0.22, size: 0.6, color: WHITE_YELLOW },
  ]

  for (const s of starDefs) {
    const halo = makeStarHalo(s.size)
    stars.push({ x: s.x, y: s.y, size: s.size, halo, coreColor: s.color })
  }

  // ===== FLOW WAVES =====
  // The sky is filled with flowing wave-like brush strokes
  // These flow generally left-to-right with sinusoidal undulation
  for (let i = 0; i < 40; i++) {
    const yBase = 0.05 + (i / 39) * 0.90  // spread across entire sky
    const t = i / 39
    const color = lerpColor(
      lerpColor(INDIGO, COBALT, t),
      lerpColor(PRUSSIAN, TEAL, t * 0.5),
      Math.random() * 0.3
    )
    flowWaves.push({
      yBase,
      amplitude: 0.015 + rand(0, 0.02),
      frequency: 2 + rand(-0.5, 1),
      phase: rand(0, Math.PI * 2),
      color,
      alpha: 0.08 + rand(0, 0.08),
      strokeW: rand(2, 5),
    })
  }
}

// ==================== Rendering ====================

function drawBackground() {
  if (!ctx) return
  // Gradient: dark indigo at top → slightly lighter blue going down
  const grad = ctx.createLinearGradient(0, 0, 0, skyH)
  grad.addColorStop(0, '#0b0a30')
  grad.addColorStop(0.3, '#100e45')
  grad.addColorStop(0.6, '#151270')
  grad.addColorStop(1, '#1a1a55')
  ctx.fillStyle = grad
  ctx.fillRect(0, 0, width, skyH)
  // Below sky: dark village silhouette
  ctx.fillStyle = '#080615'
  ctx.fillRect(0, skyH, width, height - skyH)
}

function drawFlowWaves(energy: {bass:number;mid:number;high:number;avg:number}) {
  if (!ctx) return
  ctx.save()
  const audioBoost = 1 + energy.avg * 1.5
  for (const wave of flowWaves) {
    ctx.beginPath()
    ctx.strokeStyle = colorStr(wave.color, wave.alpha * (1 + energy.mid * 0.5))
    ctx.lineWidth = wave.strokeW
    const yCenter = wave.yBase * skyH
    const timePhase = th * 0.3 * audioBoost + wave.phase
    for (let x = 0; x <= width; x += 3) {
      const xNorm = x / width
      const y = yCenter + Math.sin(xNorm * Math.PI * wave.frequency + timePhase) * wave.amplitude * skyH
      if (x === 0) ctx.moveTo(x, y)
      else ctx.lineTo(x, y)
    }
    ctx.stroke()
  }
  ctx.restore()
}

function drawSpiral(cx: number, cy: number, rings: SpiralRing[], energy: {bass:number;mid:number;high:number;avg:number}, extraRotation: number = 0) {
  if (!ctx) return
  ctx.save()
  ctx.translate(cx, cy)

  for (const ring of rings) {
    const rotation = th * ring.speed * ring.dir + extraRotation
    const audioBoost = 1 + energy.bass * 0.5

    ctx.beginPath()
    ctx.strokeStyle = colorStr(ring.color, ring.alpha)
    ctx.lineWidth = ring.width
    ctx.arc(0, 0, ring.baseRadius * audioBoost, ring.startAng + rotation, ring.startAng + rotation + ring.sweep)
    ctx.stroke()
  }

  ctx.restore()
}

function drawStar(star: typeof stars[0], energy: {bass:number;mid:number;high:number;avg:number}) {
  if (!ctx) return
  const cx = star.x * width
  const cy = star.y * skyH

  // Draw halo rings
  drawSpiral(cx, cy, star.halo.rings, energy)

  // Draw bright core
  ctx.save()
  ctx.translate(cx, cy)
  const coreR = star.size * 4 * (1 + energy.high * 0.3)
  const grad = ctx.createRadialGradient(0, 0, 0, 0, 0, coreR)
  grad.addColorStop(0, colorStr(WHITE_YELLOW, 0.9))
  grad.addColorStop(0.3, colorStr(star.coreColor, 0.5))
  grad.addColorStop(1, colorStr(star.coreColor, 0))
  ctx.fillStyle = grad
  ctx.beginPath()
  ctx.arc(0, 0, coreR, 0, Math.PI * 2)
  ctx.fill()
  ctx.restore()
}

function drawMoon(energy: {bass:number;mid:number;high:number;avg:number}) {
  if (!ctx) return
  const cx = 0.82 * width
  const cy = 0.15 * skyH

  // Draw moon's radiating rings
  drawSpiral(cx, cy, moonRings, energy)

  // Draw moon crescent: bright circle shifted to create crescent effect
  ctx.save()
  ctx.translate(cx, cy)

  // Bright core
  const moonR = Math.min(width, skyH) * 0.015
  const grad = ctx.createRadialGradient(0, 0, 0, 0, 0, moonR * 2)
  grad.addColorStop(0, colorStr(WHITE_YELLOW, 0.95))
  grad.addColorStop(0.5, colorStr(CAD_YELLOW, 0.6))
  grad.addColorStop(1, colorStr(CAD_YELLOW, 0))
  ctx.fillStyle = grad
  ctx.beginPath()
  ctx.arc(0, 0, moonR * 2, 0, Math.PI * 2)
  ctx.fill()

  // Crescent: overlay a dark circle offset to the right to create crescent
  ctx.globalCompositeOperation = 'destination-out'
  ctx.beginPath()
  ctx.arc(moonR * 0.6, -moonR * 0.2, moonR * 0.85, 0, Math.PI * 2)
  ctx.fill()
  ctx.globalCompositeOperation = 'source-over'

  ctx.restore()
}

function animate() {
  if (!ctx) return
  const energy = getAudioEnergy()

  drawBackground()

  // 1. Flow waves (background layer)
  drawFlowWaves(energy)

  // 2. Main vortex (center of the painting)
  const vortexCX = 0.35 * width
  const vortexCY = 0.40 * skyH
  drawSpiral(vortexCX, vortexCY, mainVortex, energy)

  // 3. Secondary vortex
  const vortex2CX = 0.62 * width
  const vortex2CY = 0.28 * skyH
  drawSpiral(vortex2CX, vortex2CY, secondaryVortex, energy)

  // 4. Stars with halos
  for (const star of stars) {
    drawStar(star, energy)
  }

  // 5. Moon (topmost layer)
  drawMoon(energy)

  th += 0.006 * (1 + energy.avg * 0.5)
  animId = requestAnimationFrame(animate)
}

function resize() {
  const canvas = canvasRef.value
  if (!canvas) return
  width = window.innerWidth
  height = window.innerHeight
  canvas.width = width
  canvas.height = height
  skyH = height * 0.65
  initScene()
}

onMounted(() => {
  const canvas = canvasRef.value
  if (!canvas) return
  ctx = canvas.getContext('2d')!
  resize()
  window.addEventListener('resize', resize)
  animate()
})

onUnmounted(() => {
  cancelAnimationFrame(animId)
  cleanupAudio()
  window.removeEventListener('resize', resize)
})
</script>

<style>
* { margin: 0; padding: 0; box-sizing: border-box }
html, body { overflow: hidden; background: #0b0a30 }
.app { position: relative; width: 100vw; height: 100vh }
canvas { display: block; width: 100%; height: 100% }

.controls {
  position: fixed; top: 20px; left: 50%; transform: translateX(-50%);
  z-index: 10; display: flex; align-items: center; gap: 12px;
}
.btn {
  padding: 8px 18px; background: rgba(255,255,255,0.1);
  border: 1px solid rgba(255,255,255,0.2); border-radius: 20px;
  color: #e3f2fd; font-size: 14px; cursor: pointer;
  backdrop-filter: blur(10px); transition: all 0.3s; user-select: none;
}
.btn:hover { background: rgba(255,255,255,0.2); border-color: rgba(255,255,255,0.4) }
.upload-btn { display: inline-flex; align-items: center }
.hint { color: rgba(255,255,255,0.4); font-size: 12px }

.info {
  position: fixed; bottom: 16px; left: 50%; transform: translateX(-50%);
  color: rgba(255,255,255,0.3); font-size: 12px; z-index: 10;
}
</style>
