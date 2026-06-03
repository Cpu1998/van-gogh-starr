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
      <span class="hint">支持拖入音乐文件 | 默认使用麦克风律动</span>
    </div>
    <canvas ref="canvasRef"></canvas>
    <div class="info">
      <span>✨ 梵高星空 · 同心弧线律动</span>
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
// Deep purple → teal → warm yellow → white
const C1 = { r: 20, g: 12, b: 53 }    // #140c35
const C2 = { r: 29, g: 173, b: 164 }   // #1dada4
const C3 = { r: 237, g: 246, b: 131 }   // #edf683
const C4 = { r: 252, g: 253, b: 239 }   // #fcfdef

function lerpColor(a: { r: number; g: number; b: number }, b2: { r: number; g: number; b: number }, t: number) {
  const clamp = (v: number) => Math.max(0, Math.min(1, v))
  t = clamp(t)
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

// ==================== Arc Layer System ====================

interface ArcSegment {
  color: { r: number; g: number; b: number }
  alpha: number
  startAngle: number
  endAngle: number
  dir: number // 1 or -1
}

interface ArcLayer {
  segments: ArcSegment[]
}

// Layer counts
const BACK_N = 150
const MIDDLE_N = 60
const INTER_N = 80
const FRONT_N = 100
const POINTS_N = 100

let backLayer: { color: { r: number; g: number; b: number }; alpha: number }[] = []
let middleLayer: ArcLayer[] = []
let interLayer: ArcLayer[] = []
let frontLayer: ArcLayer[] = []
let pointsLayer: ArcLayer[] = []

let th = 0
let viewport = 0

function rand(min: number, max: number) { return Math.random() * (max - min) + min }

function initLayers() {
  backLayer = []
  middleLayer = []
  interLayer = []
  frontLayer = []
  pointsLayer = []

  // Back layer: filled circles with gradient colors
  for (let i = 0; i < BACK_N; i++) {
    const ratio = i / (BACK_N - 1)
    backLayer.push({
      color: paletteColor(ratio),
      alpha: (20 + 5 * ratio) / 255,
    })
  }

  // Middle layer: arcs (5-9 segments each)
  for (let i = 0; i < MIDDLE_N; i++) {
    const ratio = i / (MIDDLE_N - 1)
    const color = paletteColor(ratio, rand(0, 0.15))
    const alpha = (70 + 5 * ratio) / 255
    const r = Math.floor(rand(5, 9))
    const segs: ArcSegment[] = []
    let k = 0
    const slice = (Math.PI * 2 - 0.01) / r
    for (let j = 0; j < r; j++) {
      let x = rand(k, k + slice / 2)
      let y = rand(k + slice / 2, k + slice)
      if (y < x) { const tmp = x; x = y; y = tmp }
      const dir = Math.random() < 0.5 ? -1 : 1
      segs.push({ color, alpha, startAngle: x, endAngle: y, dir })
      k += slice
    }
    middleLayer.push({ segments: segs })
  }

  // Inter layer: arcs (9-15 segments each)
  for (let i = 0; i < INTER_N; i++) {
    const ratio = i / (INTER_N - 1)
    const color = paletteColor(ratio, rand(-0.3, 0))
    const alpha = (70 + 5 * ratio) / 255
    const r = Math.floor(rand(9, 15))
    const segs: ArcSegment[] = []
    let k = 0
    const slice = (Math.PI * 2 - 0.01) / r
    for (let j = 0; j < r; j++) {
      let x = rand(k, k + slice / 2)
      let y = rand(k + slice / 2, k + slice)
      if (y < x) { const tmp = x; x = y; y = tmp }
      const dir = Math.random() < 0.5 ? -1 : 1
      segs.push({ color, alpha, startAngle: x, endAngle: y, dir })
      k += slice
    }
    interLayer.push({ segments: segs })
  }

  // Front layer: arcs (3-6 segments, skip some)
  for (let i = 0; i < FRONT_N; i++) {
    const ratio = i / (FRONT_N - 1)
    const color = paletteColor(ratio, rand(0, 0.15))
    const alpha = (155 + 100 * ratio) / 255
    const r = Math.floor(rand(3, 6))
    const segs: ArcSegment[] = []
    let k = 0
    const slice = (Math.PI * 2 - 0.01) / r
    for (let j = 0; j < r; j++) {
      let x = rand(k, k + slice / 2)
      let y = rand(k + slice / 2, k + slice)
      if (y < x) { const tmp = x; x = y; y = tmp }
      const dir = Math.random() < 0.5 ? -1 : 1
      if (i % 4 < 1) { k += slice; continue }
      segs.push({ color, alpha, startAngle: x, endAngle: y, dir })
      k += slice
    }
    frontLayer.push({ segments: segs })
  }

  // Points layer: very short arcs (dots)
  for (let i = 0; i < POINTS_N; i++) {
    const ratio = i / (POINTS_N - 1)
    const color = paletteColor(ratio, rand(-0.3, 0.3))
    const alpha = Math.max(0, Math.min(1, (155 + 100 * rand(-1, 1)) / 255))
    const r = Math.floor(rand(8, 16))
    const segs: ArcSegment[] = []
    let k = 0
    const slice = (Math.PI * 2 - 0.01) / r
    for (let j = 0; j < r; j++) {
      const ang = rand(k, k + slice)
      const dir = rand(-1, 1)
      segs.push({ color, alpha, startAngle: ang, endAngle: ang, dir })
      k += slice
    }
    pointsLayer.push({ segments: segs })
  }
}

// ==================== Planet ====================
const PLANET_RATIO = 1 / 4.1
const PLANET_Y_RATIO = -1 / 5

// ==================== Rendering ====================

function drawBackground() {
  if (!ctx) return
  ctx.fillStyle = '#140c25'
  ctx.fillRect(0, 0, width, height)
}

function drawBackLayer() {
  if (!ctx) return
  ctx.save()
  ctx.translate(width / 2, height / 2)
  ctx.scale(1, -1)
  for (let i = 0; i < backLayer.length; i++) {
    const item = backLayer[i]
    const radius = ((viewport - 40) * (1 - i / (BACK_N + 1))) / 2
    ctx.beginPath()
    ctx.fillStyle = colorStr(item.color, item.alpha)
    ctx.arc(0, 0, radius, 0, Math.PI * 2)
    ctx.fill()
  }
  ctx.restore()
}

function drawArcLayer(
  layers: ArcLayer[],
  total: number,
  strokeW: number,
  speedFactor: number,
  energy: { bass: number; mid: number; high: number; avg: number },
  energyMul: number
) {
  if (!ctx) return
  ctx.save()
  ctx.translate(width / 2, height / 2)
  ctx.scale(1, -1)

  for (let i = 0; i < layers.length; i++) {
    const group = layers[i]
    const ratio = i / (total + 1)
    const baseRadius = (viewport - 40) * (1 - ratio) / 2
    const depthFactor = 1.5 + (1 - ratio)
    const audioBoost = 1 + energy.avg * energyMul

    for (let j = 0; j < group.segments.length; j++) {
      const seg = group.segments[j]
      const rotation = th * speedFactor * depthFactor * seg.dir * audioBoost

      ctx.beginPath()
      ctx.strokeStyle = colorStr(seg.color, seg.alpha)
      ctx.lineWidth = strokeW

      if (seg.startAngle === seg.endAngle) {
        // Point: draw a tiny arc
        const r = viewport * (1 - i / (total + 1)) / 2
        ctx.arc(0, 0, r, seg.startAngle + rotation, seg.startAngle + rotation + 0.0001)
      } else {
        ctx.arc(0, 0, baseRadius, seg.startAngle + rotation, seg.endAngle + rotation)
      }
      ctx.stroke()
    }
  }
  ctx.restore()
}

function drawPlanet() {
  if (!ctx) return
  const energy = getAudioEnergy()
  const sz = viewport / 7
  const msz = sz / 4
  const px = viewport * PLANET_RATIO
  const py = viewport * PLANET_Y_RATIO

  ctx.save()
  ctx.translate(width / 2, height / 2)
  ctx.scale(1, -1)
  ctx.rotate(2 * th / 3 * (1 + energy.high * 0.5))

  const norm = Math.hypot(px, py)
  const nx = px / norm, ny = py / norm

  for (let i = 0; i < BACK_N; i++) {
    const j = i / (BACK_N - 1)
    ctx.save()
    ctx.translate(px, py)
    ctx.translate(nx * msz * j / 2 + 0.1, ny * msz * j / 2 + 0.1)

    // Rotate to align ellipse with planet direction
    ctx.rotate(Math.atan2(py, px))

    const k1 = lerpColor(C2, C3, 0.5)
    const k2 = lerpColor(C1, C2, 0.2)
    const s = 1 - Math.hypot(px, py) / viewport
    const sMapped = s * 0.5
    const k = lerpColor(k1, k2, Math.pow(j, sMapped))
    const alpha = 25 / 255 + energy.mid * 0.1

    ctx.beginPath()
    ctx.fillStyle = colorStr(k, alpha)
    ctx.ellipse(0, 0, (sz - msz * j) / 2, (sz - msz * j * 0.36) / 2, 0, 0, Math.PI * 2)
    ctx.fill()
    ctx.restore()
  }

  ctx.restore()
}

function animate() {
  if (!ctx) return

  const energy = getAudioEnergy()

  drawBackground()

  // Back: filled gradient circles (bass affects opacity pulse)
  drawBackLayer()

  // Middle arcs: rotate at th/8 speed, bass-driven
  const midStroke = (viewport - 40) / 2 / (MIDDLE_N + 1)
  drawArcLayer(middleLayer, MIDDLE_N, midStroke, 1 / 8, energy, 2)

  // Inter arcs: rotate at th/6 speed, mid-driven
  const interStroke = (viewport - 40) / 2 / (INTER_N + 1)
  drawArcLayer(interLayer, INTER_N, interStroke, 1 / 6, energy, 1.5)

  // Front arcs: rotate at th/2 speed, high-driven
  const frontStroke = (viewport - 40) / 2 / (FRONT_N + 1)
  drawArcLayer(frontLayer, FRONT_N, frontStroke, 1 / 2, energy, 3)

  // Points: very short arcs, th/2 speed
  const pointsStroke = (viewport - 40) / (POINTS_N + 1) / 2
  drawArcLayer(pointsLayer, POINTS_N, pointsStroke, 1 / 2, energy, 2)

  // Planet
  drawPlanet()

  th += 0.01 * (1 + energy.avg * 0.5)
  animId = requestAnimationFrame(animate)
}

function resize() {
  const canvas = canvasRef.value
  if (!canvas) return
  width = window.innerWidth
  height = window.innerHeight
  canvas.width = width
  canvas.height = height
  viewport = Math.min(height, width)
  initLayers()
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
html, body { overflow: hidden; background: #140c25 }
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
