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
      <span class="hint">梵高星空 · 天体轮廓 · 同心弧线风格</span>
    </div>
    <canvas ref="canvasRef"></canvas>
    <div class="info">
      <span>✨ 星 · 月亮 · 银河 — 大概轮廓与位置</span>
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

// ==================== Audio ====================
let audioCtx: AudioContext | null = null
let analyser: AnalyserNode | null = null
let dataArray: Uint8Array = new Uint8Array(0)
let audioSource: MediaElementAudioSourceNode | MediaStreamAudioSourceNode | null = null
let audioElement: HTMLAudioElement | null = null
let micStream: MediaStream | null = null
let sourceType: 'none' | 'file' | 'mic' = 'none'

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
  sourceType = 'file'
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
  sourceType = 'mic'
  isPlaying.value = true
}

function cleanupAudio() {
  if (audioElement) { audioElement.pause(); audioElement.src = ''; audioElement = null }
  if (micStream) { micStream.getTracks().forEach(t => t.stop()); micStream = null }
  if (audioCtx && audioCtx.state !== 'closed') { audioCtx.close() }
  audioCtx = null; analyser = null; audioSource = null; sourceType = 'none'
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

// ==================== Color Palette ====================
const C1 = { r: 20, g: 12, b: 53 }
const C2 = { r: 29, g: 173, b: 164 }
const C3 = { r: 237, g: 246, b: 131 }
const C4 = { r: 252, g: 253, b: 239 }

function lerpColor(a: { r: number; g: number; b: number }, b2: { r: number; g: number; b: number }, t: number) {
  t = Math.max(0, Math.min(1, t))
  return {
    r: Math.round(a.r + (b2.r - a.r) * t),
    g: Math.round(a.g + (b2.g - a.g) * t),
    b: Math.round(a.b + (b2.b - a.b) * t),
  }
}

function colorStr(c: { r: number; g: number; b: number }, alpha: number) {
  return `rgba(${c.r},${c.g},${c.b},${alpha})`
}

function paletteColor(ratio: number, jitter: number = 0): { r: number; g: number; b: number } {
  const t = Math.max(0, Math.min(1, ratio + jitter))
  if (t < 1 / 3) return lerpColor(C1, C2, t * 3)
  if (t < 2 / 3) return lerpColor(C2, C3, (t - 1 / 3) * 3)
  return lerpColor(C3, C4, (t - 2 / 3) * 3)
}

// ==================== Celestial Objects ====================
// Positions based on Van Gogh's Starry Night (normalized 0-1, origin top-left)
// Only upper 70% of canvas = sky region

interface CelestialObject {
  x: number   // normalized x (0=left, 1=right)
  y: number   // normalized y (0=top, 1=bottom of sky)
  size: number // relative size (1.0 = medium star)
  type: 'star' | 'moon' | 'vortex' | 'milkyway'
  brightness: number // 0-1
  layers: ArcGroup[]
}

interface ArcSegment {
  color: { r: number; g: number; b: number }
  alpha: number
  startAngle: number
  endAngle: number
  dir: number
}

interface ArcGroup {
  segments: ArcSegment[]
}

let th = 0
let objects: CelestialObject[] = []

function rand(min: number, max: number) { return Math.random() * (max - min) + min }

function makeArcLayers(count: number, segRange: [number, number], colorBias: number, alphaBase: number): ArcGroup[] {
  const groups: ArcGroup[] = []
  for (let i = 0; i < count; i++) {
    const ratio = i / (count - 1)
    const color = paletteColor(ratio, colorBias)
    const alpha = (alphaBase + ratio * 50) / 255
    const r = Math.floor(rand(segRange[0], segRange[1]))
    const segs: ArcSegment[] = []
    let k = 0
    const slice = (Math.PI * 2 - 0.01) / r
    for (let j = 0; j < r; j++) {
      let x = rand(k, k + slice / 2)
      let y = rand(k + slice / 2, k + slice)
      if (y < x) { const tmp = x; x = y; y = tmp }
      segs.push({ color, alpha, startAngle: x, endAngle: y, dir: Math.random() < 0.5 ? -1 : 1 })
      k += slice
    }
    groups.push({ segments: segs })
  }
  return groups
}

function makeDotLayers(count: number, colorBias: number, alphaBase: number): ArcGroup[] {
  const groups: ArcGroup[] = []
  for (let i = 0; i < count; i++) {
    const ratio = i / Math.max(1, count - 1)
    const color = paletteColor(ratio, colorBias)
    const alpha = Math.max(0, Math.min(1, (alphaBase + rand(-30, 30)) / 255))
    const r = Math.floor(rand(6, 14))
    const segs: ArcSegment[] = []
    let k = 0
    const slice = (Math.PI * 2) / r
    for (let j = 0; j < r; j++) {
      const ang = rand(k, k + slice)
      segs.push({ color, alpha, startAngle: ang, endAngle: ang, dir: rand(-1, 1) })
      k += slice
    }
    groups.push({ segments: segs })
  }
  return groups
}

function initObjects() {
  objects = []

  // ===== MILKY WAY: flowing band across the sky =====
  // The central swirl / milky way is a series of overlapping vortex centers
  // Main path: from left (~0.15, 0.35) curves up through center to right (~0.75, 0.15)
  const milkyWayPoints = [
    { x: 0.12, y: 0.42, size: 0.8 },
    { x: 0.22, y: 0.38, size: 1.0 },
    { x: 0.32, y: 0.35, size: 1.3 },  // main vortex area
    { x: 0.38, y: 0.30, size: 1.5 },  // peak of main swirl
    { x: 0.45, y: 0.28, size: 1.2 },
    { x: 0.52, y: 0.25, size: 1.0 },
    { x: 0.60, y: 0.22, size: 0.9 },
    { x: 0.68, y: 0.20, size: 0.8 },
    { x: 0.75, y: 0.18, size: 0.7 },
    { x: 0.82, y: 0.20, size: 0.6 },
    { x: 0.90, y: 0.25, size: 0.5 },
  ]

  for (const pt of milkyWayPoints) {
    objects.push({
      x: pt.x, y: pt.y, size: pt.size,
      type: 'milkyway',
      brightness: 0.3 + pt.size * 0.2,
      layers: makeArcLayers(40, [4, 10], rand(-0.2, 0.1), 40),
    })
  }

  // ===== MAIN VORTEX: the big swirl slightly left of center =====
  objects.push({
    x: 0.35, y: 0.33, size: 2.5,
    type: 'vortex',
    brightness: 0.6,
    layers: makeArcLayers(70, [5, 12], 0.1, 50),
  })

  // ===== SECONDARY VORTEX: smaller swirl to the right =====
  objects.push({
    x: 0.62, y: 0.25, size: 1.5,
    type: 'vortex',
    brightness: 0.4,
    layers: makeArcLayers(45, [5, 10], -0.1, 45),
  })

  // ===== MOON: upper right, crescent =====
  objects.push({
    x: 0.84, y: 0.14, size: 2.0,
    type: 'moon',
    brightness: 0.95,
    layers: makeArcLayers(50, [6, 12], 0.4, 80),
  })

  // ===== STARS: scattered across the sky =====
  // Bright stars (large halos)
  const brightStars = [
    { x: 0.18, y: 0.18 },  // upper left
    { x: 0.50, y: 0.12 },  // top center
    { x: 0.30, y: 0.50 },  // left-center, below main vortex
    { x: 0.72, y: 0.38 },  // right side
    { x: 0.14, y: 0.30 },  // far left
  ]

  for (const s of brightStars) {
    objects.push({
      x: s.x, y: s.y, size: 1.2,
      type: 'star',
      brightness: 0.8 + rand(0, 0.15),
      layers: makeArcLayers(35, [5, 10], rand(0.1, 0.4), 70),
    })
    // Add dot halo around each bright star
    const star = objects[objects.length - 1]
    star.layers.push(...makeDotLayers(20, 0.3, 120))
  }

  // Medium stars
  const medStars = [
    { x: 0.25, y: 0.10 },
    { x: 0.42, y: 0.42 },
    { x: 0.55, y: 0.35 },
    { x: 0.78, y: 0.30 },
    { x: 0.65, y: 0.45 },
    { x: 0.38, y: 0.15 },
    { x: 0.48, y: 0.48 },
  ]

  for (const s of medStars) {
    objects.push({
      x: s.x, y: s.y, size: 0.7,
      type: 'star',
      brightness: 0.5 + rand(0, 0.2),
      layers: makeArcLayers(20, [5, 8], rand(0, 0.3), 60),
    })
    const star = objects[objects.length - 1]
    star.layers.push(...makeDotLayers(10, 0.2, 100))
  }

  // Small stars / star dots
  const smallStars = [
    { x: 0.08, y: 0.20 }, { x: 0.22, y: 0.55 },
    { x: 0.35, y: 0.58 }, { x: 0.58, y: 0.50 },
    { x: 0.70, y: 0.12 }, { x: 0.85, y: 0.35 },
    { x: 0.92, y: 0.18 }, { x: 0.15, y: 0.48 },
    { x: 0.45, y: 0.55 }, { x: 0.75, y: 0.48 },
    { x: 0.05, y: 0.35 }, { x: 0.60, y: 0.08 },
    { x: 0.33, y: 0.22 }, { x: 0.88, y: 0.45 },
    { x: 0.28, y: 0.62 }, { x: 0.53, y: 0.05 },
  ]

  for (const s of smallStars) {
    objects.push({
      x: s.x, y: s.y, size: 0.35,
      type: 'star',
      brightness: 0.3 + rand(0, 0.2),
      layers: makeDotLayers(8, rand(-0.2, 0.4), 90),
    })
  }
}

// ==================== Rendering ====================

function drawBackground() {
  if (!ctx) return
  // Gradient sky: dark blue at top, slightly lighter at bottom
  const grad = ctx.createLinearGradient(0, 0, 0, height)
  grad.addColorStop(0, '#0a0620')
  grad.addColorStop(0.4, '#140c35')
  grad.addColorStop(0.7, '#1a1048')
  grad.addColorStop(1, '#0d1530')
  ctx.fillStyle = grad
  ctx.fillRect(0, 0, width, height)
}

function drawObject(
  obj: CelestialObject,
  energy: { bass: number; mid: number; high: number; avg: number }
) {
  if (!ctx) return

  const cx = obj.x * width
  const cy = obj.y * height * 0.75  // sky occupies top 75%
  const baseSize = Math.min(width, height) * 0.02 * obj.size

  // Speed varies by type
  let speedFactor = 1 / 6
  if (obj.type === 'vortex') speedFactor = 1 / 4
  else if (obj.type === 'moon') speedFactor = 1 / 10
  else if (obj.type === 'star') speedFactor = 1 / 3
  else if (obj.type === 'milkyway') speedFactor = 1 / 8

  // Energy influence
  let energyMul = 1
  if (obj.type === 'vortex') energyMul = 3
  else if (obj.type === 'moon') energyMul = 1.5
  else if (obj.type === 'star') energyMul = 2

  const audioBoost = 1 + energy.avg * energyMul

  ctx.save()
  ctx.translate(cx, cy)

  for (let i = 0; i < obj.layers.length; i++) {
    const group = obj.layers[i]
    const ratio = i / (obj.layers.length + 1)
    const radius = baseSize * (obj.layers.length * 0.6) * (1 - ratio)

    // Moon crescent offset
    let offsetX = 0, offsetY = 0
    if (obj.type === 'moon') {
      // Shift inner layers to create crescent illusion
      const crescentShift = ratio * baseSize * 1.5
      offsetX = crescentShift * 0.6
      offsetY = -crescentShift * 0.3
    }

    for (const seg of group.segments) {
      const rotation = th * speedFactor * (1 + ratio * 0.5) * seg.dir * audioBoost

      ctx.beginPath()
      ctx.strokeStyle = colorStr(seg.color, seg.alpha * obj.brightness)

      if (obj.type === 'moon') {
        // Use thinner strokes for moon, more ethereal
        ctx.lineWidth = Math.max(0.5, baseSize * 0.15)
      } else if (obj.type === 'vortex') {
        ctx.lineWidth = Math.max(0.8, baseSize * 0.2)
      } else {
        ctx.lineWidth = Math.max(0.5, baseSize * 0.15 * (1 - ratio))
      }

      if (seg.startAngle === seg.endAngle) {
        // Dot
        ctx.arc(offsetX, offsetY, radius, seg.startAngle + rotation, seg.startAngle + rotation + 0.001)
      } else {
        ctx.arc(offsetX, offsetY, radius, seg.startAngle + rotation, seg.endAngle + rotation)
      }
      ctx.stroke()
    }
  }

  // For bright stars and moon, add a glow center
  if ((obj.type === 'star' && obj.brightness > 0.7) || obj.type === 'moon') {
    const glowSize = baseSize * (obj.type === 'moon' ? 0.8 : 0.4)
    const grad = ctx.createRadialGradient(
      obj.type === 'moon' ? baseSize * 0.3 : 0,
      obj.type === 'moon' ? -baseSize * 0.15 : 0,
      0,
      obj.type === 'moon' ? baseSize * 0.3 : 0,
      obj.type === 'moon' ? -baseSize * 0.15 : 0,
      glowSize
    )
    const glowColor = obj.type === 'moon'
      ? lerpColor(C3, C4, 0.6)
      : lerpColor(C2, C3, 0.7)
    grad.addColorStop(0, colorStr(glowColor, 0.6 * obj.brightness))
    grad.addColorStop(0.5, colorStr(glowColor, 0.15 * obj.brightness))
    grad.addColorStop(1, colorStr(glowColor, 0))
    ctx.fillStyle = grad
    ctx.fillRect(-glowSize, -glowSize, glowSize * 2, glowSize * 2)
  }

  ctx.restore()
}

// Milky way connecting flow — draw a subtle flowing band between milky way points
function drawMilkyWayFlow(energy: { bass: number; mid: number; high: number; avg: number }) {
  if (!ctx) return

  ctx.save()
  ctx.globalAlpha = 0.15 + energy.avg * 0.1

  const milkyWayPoints = [
    { x: 0.12, y: 0.42 },
    { x: 0.22, y: 0.38 },
    { x: 0.32, y: 0.35 },
    { x: 0.38, y: 0.30 },
    { x: 0.45, y: 0.28 },
    { x: 0.52, y: 0.25 },
    { x: 0.60, y: 0.22 },
    { x: 0.68, y: 0.20 },
    { x: 0.75, y: 0.18 },
    { x: 0.82, y: 0.20 },
    { x: 0.90, y: 0.25 },
  ]

  // Draw flowing bezier curves
  for (let pass = 0; pass < 3; pass++) {
    ctx.beginPath()
    const wave = Math.sin(th * 0.3 + pass) * 0.02
    const startY = milkyWayPoints[0].y * height * 0.75 + wave * height
    ctx.moveTo(milkyWayPoints[0].x * width, startY)

    for (let i = 1; i < milkyWayPoints.length - 1; i++) {
      const curr = milkyWayPoints[i]
      const next = milkyWayPoints[i + 1]
      const cpx = curr.x * width
      const cpy = curr.y * height * 0.75 + Math.sin(th * 0.5 + i + pass) * height * 0.02
      const epx = ((curr.x + next.x) / 2) * width
      const epy = ((curr.y + next.y) / 2) * height * 0.75 + Math.sin(th * 0.4 + i * 2 + pass) * height * 0.015
      ctx.quadraticCurveTo(cpx, cpy, epx, epy)
    }

    const last = milkyWayPoints[milkyWayPoints.length - 1]
    ctx.lineTo(last.x * width, last.y * height * 0.75)

    const bandWidth = (15 + pass * 8) * (1 + energy.bass * 0.5)
    ctx.lineWidth = bandWidth
    const flowColor = paletteColor(0.3 + pass * 0.15, 0)
    ctx.strokeStyle = colorStr(flowColor, 0.08 + pass * 0.03)
    ctx.stroke()
  }

  ctx.restore()
}

function animate() {
  if (!ctx) return
  const energy = getAudioEnergy()

  drawBackground()

  // Draw milky way connecting flow first (behind everything)
  drawMilkyWayFlow(energy)

  // Sort: draw milkyway first, then vortex, then stars, then moon on top
  const order = { milkyway: 0, vortex: 1, star: 2, moon: 3 }
  const sorted = [...objects].sort((a, b) => order[a.type] - order[b.type])

  for (const obj of sorted) {
    drawObject(obj, energy)
  }

  th += 0.008 * (1 + energy.avg * 0.5)
  animId = requestAnimationFrame(animate)
}

function resize() {
  const canvas = canvasRef.value
  if (!canvas) return
  width = window.innerWidth
  height = window.innerHeight
  canvas.width = width
  canvas.height = height
  initObjects()
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
html, body { overflow: hidden; background: #0a0620 }
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
