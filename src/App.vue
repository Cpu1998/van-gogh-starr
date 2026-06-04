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
      <span class="hint">梵高星空 · 笔触复刻 · 天空部分</span>
    </div>
    <canvas ref="canvasRef"></canvas>
    <div class="info">
      <span>✨ 忠实复刻梵高《星月夜》天空笔触</span>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const canvasRef = ref<HTMLCanvasElement | null>(null)
const isPlaying = ref(false)

let ctx: CanvasRenderingContext2D | null = null
let animId = 0
let W = 0
let H = 0

// ==================== Audio ====================
let audioCtx: AudioContext | null = null
let analyser: AnalyserNode | null = null
let dataArray: Uint8Array = new Uint8Array(0)
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
  const src = audioCtx.createMediaElementSource(audioElement)
  src.connect(analyser)
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
  const src = audioCtx.createMediaStreamSource(micStream)
  src.connect(analyser)
  isPlaying.value = true
}

function cleanupAudio() {
  if (audioElement) { audioElement.pause(); audioElement.src = ''; audioElement = null }
  if (micStream) { micStream.getTracks().forEach(t => t.stop()); micStream = null }
  if (audioCtx && audioCtx.state !== 'closed') { audioCtx.close() }
  audioCtx = null; analyser = null
  isPlaying.value = false
}

async function toggleAudio() {
  if (isPlaying.value) { cleanupAudio() } else { await initMicAudio() }
}

function handleAudioUpload(e: Event) {
  const file = (e.target as HTMLInputElement).files?.[0]
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

// ==================== Colors (Van Gogh's palette) ====================
const DEEP_BLUE    = { r: 18, g: 22, b: 82 }
const COBALT       = { r: 35, g: 55, b: 145 }
const MID_BLUE     = { r: 55, g: 80, b: 165 }
const LIGHT_BLUE   = { r: 90, g: 130, b: 195 }
const TEAL_BLUE    = { r: 60, g: 155, b: 165 }
const DARK_TEAL    = { r: 30, g: 100, b: 110 }
const CAD_YELLOW   = { r: 255, g: 215, b: 40 }
const PALE_YELLOW  = { r: 255, g: 240, b: 120 }
const WHITE_YELLOW = { r: 255, g: 252, b: 210 }
const WARM_WHITE   = { r: 255, g: 255, b: 240 }

function lerp(a: number, b: number, t: number) { return a + (b - a) * t }
function lerpC(a: {r:number;g:number;b:number}, b: {r:number;g:number;b:number}, t: number) {
  t = Math.max(0, Math.min(1, t))
  return { r: Math.round(lerp(a.r, b.r, t)), g: Math.round(lerp(a.g, b.g, t)), b: Math.round(lerp(a.b, b.b, t)) }
}
function cStr(c: {r:number;g:number;b:number}, a: number) { return `rgba(${c.r},${c.g},${c.b},${a})` }

// ==================== Brush Stroke System ====================
// Each stroke is a short curved line segment defined by:
// - position (x, y) in normalized coords (0-1)
// - direction angle
// - length
// - curvature (bezier control point offset)
// - color
// - alpha
// - width

interface BrushStroke {
  x: number; y: number           // start position (normalized)
  angle: number                   // direction in radians
  len: number                     // length (normalized, relative to canvas)
  curve: number                   // curvature offset (perpendicular to direction)
  color: {r:number;g:number;b:number}
  alpha: number
  width: number
  // animation
  waveGroup: number               // which wave group (for animation phase)
  waveSpeed: number               // individual animation speed
  waveAmp: number                 // how much it moves
}

let strokes: BrushStroke[] = []
let th = 0

function stroke(
  x: number, y: number, angle: number, len: number, curve: number,
  color: {r:number;g:number;b:number}, alpha: number, width: number,
  waveGroup: number = 0, waveSpeed: number = 0.3, waveAmp: number = 0.003
): BrushStroke {
  return { x, y, angle, len, curve, color, alpha, width, waveGroup, waveSpeed, waveAmp }
}

// ==================== Build all strokes ====================
// Van Gogh's Starry Night sky composition (canvas coordinates: origin top-left)
// The painting is roughly 2:1 ratio, sky occupies top 65%

function buildStrokes() {
  strokes = []
  const rng = mulberry32(42) // seeded random for consistency
  const rand = () => rng()
  const randRange = (a: number, b: number) => a + rand() * (b - a)

  // ===== BACKGROUND FLOW STROKES =====
  // The sky is filled with flowing, wavy parallel strokes moving generally left→right
  // They follow sinusoidal paths creating the "wind" effect
  // Colors: deep blue → cobalt blue, with teal touches

  // Define wave bands - each band is a horizontal flow of parallel strokes
  const waveBands = [
    // { yCenter, amplitude, direction, color, count }
    { yBase: 0.06, amp: 0.025, dir: 0.1, color: lerpC(DEEP_BLUE, COBALT, 0.3), n: 25 },
    { yBase: 0.11, amp: 0.03, dir: 0.15, color: lerpC(DEEP_BLUE, COBALT, 0.4), n: 30 },
    { yBase: 0.16, amp: 0.035, dir: 0.2, color: lerpC(COBALT, MID_BLUE, 0.3), n: 28 },
    { yBase: 0.22, amp: 0.04, dir: 0.15, color: lerpC(COBALT, MID_BLUE, 0.5), n: 32 },
    { yBase: 0.28, amp: 0.035, dir: 0.1, color: lerpC(COBALT, TEAL_BLUE, 0.2), n: 30 },
    // gap for main vortex area - strokes curve around it
    { yBase: 0.38, amp: 0.04, dir: -0.1, color: lerpC(MID_BLUE, LIGHT_BLUE, 0.3), n: 28 },
    { yBase: 0.44, amp: 0.035, dir: -0.15, color: lerpC(COBALT, MID_BLUE, 0.4), n: 30 },
    { yBase: 0.50, amp: 0.03, dir: -0.1, color: lerpC(DEEP_BLUE, COBALT, 0.5), n: 26 },
    { yBase: 0.56, amp: 0.025, dir: -0.05, color: lerpC(DEEP_BLUE, COBALT, 0.3), n: 24 },
    { yBase: 0.62, amp: 0.02, dir: 0.0, color: lerpC(DEEP_BLUE, COBALT, 0.2), n: 22 },
  ]

  for (const band of waveBands) {
    for (let i = 0; i < band.n; i++) {
      const xBase = randRange(-0.05, 1.05)
      const yOff = randRange(-band.amp, band.amp)
      const x = xBase
      const y = band.yBase + yOff
      // Strokes follow the wave direction + slight individual variation
      const angle = band.dir + randRange(-0.3, 0.3)
      const len = randRange(0.04, 0.09)
      const curve = randRange(-0.015, 0.015)
      const c = lerpC(band.color, COBALT, randRange(-0.1, 0.1))
      const alpha = randRange(0.25, 0.55)
      const w = randRange(1.5, 4)
      strokes.push(stroke(x, y, angle, len, curve, c, alpha, w, 0, randRange(0.2, 0.4), randRange(0.001, 0.004)))
    }
  }

  // ===== EXTRA FLOW STROKES around vortex =====
  // Strokes that curve around the main vortex center (0.37, 0.36)
  const vcx = 0.37, vcy = 0.36
  for (let i = 0; i < 60; i++) {
    const angle_around = randRange(0, Math.PI * 2)
    const dist = randRange(0.06, 0.18)
    const x = vcx + Math.cos(angle_around) * dist
    const y = vcy + Math.sin(angle_around) * dist * 0.7
    // Strokes tangent to the circle around vortex center
    const tangentAngle = angle_around + Math.PI / 2 + randRange(-0.4, 0.4)
    const len = randRange(0.03, 0.08)
    const curve = randRange(-0.012, 0.012)
    const t = dist / 0.18
    const c = lerpC(lerpC(MID_BLUE, LIGHT_BLUE, 1 - t), TEAL_BLUE, rand() * 0.3)
    const alpha = randRange(0.3, 0.6)
    strokes.push(stroke(x, y, tangentAngle, len, curve, c, alpha, randRange(2, 4.5), 1, randRange(0.3, 0.5), randRange(0.002, 0.005)))
  }

  // ===== MAIN VORTEX SPIRAL =====
  // The large spiral: starts from outside, curves inward in a tight spiral
  // Center: (0.37, 0.36), makes about 2 full turns
  const spiralCenterX = 0.37, spiralCenterY = 0.36
  const spiralStrokes = 90
  for (let i = 0; i < spiralStrokes; i++) {
    const t = i / spiralStrokes
    // Spiral: radius decreases as angle increases
    const angleStart = t * Math.PI * 4  // 2 full turns
    const radius = 0.14 * (1 - t * 0.85)  // outer to inner
    // Position along spiral
    const x = spiralCenterX + Math.cos(angleStart) * radius
    const y = spiralCenterY + Math.sin(angleStart) * radius * 0.75
    // Tangent direction (perpendicular to radius + spiral direction)
    const tangentAngle = angleStart + Math.PI / 2 + 0.3
    const len = randRange(0.02, 0.05)
    const curve = randRange(-0.008, 0.008)
    // Inner strokes are lighter/teal, outer are deeper blue
    const c = t < 0.5
      ? lerpC(MID_BLUE, LIGHT_BLUE, t * 2)
      : lerpC(LIGHT_BLUE, TEAL_BLUE, (t - 0.5) * 2)
    const alpha = randRange(0.35, 0.65)
    strokes.push(stroke(x, y, tangentAngle, len, curve, c, alpha, randRange(2, 4), 1, randRange(0.4, 0.7), randRange(0.002, 0.006)))
  }

  // Inner vortex glow (brighter, shorter strokes toward center)
  for (let i = 0; i < 25; i++) {
    const angleStart = rand() * Math.PI * 2
    const radius = randRange(0.005, 0.025)
    const x = spiralCenterX + Math.cos(angleStart) * radius
    const y = spiralCenterY + Math.sin(angleStart) * radius * 0.75
    const tangentAngle = angleStart + Math.PI / 2 + randRange(-0.5, 0.5)
    const c = lerpC(TEAL_BLUE, LIGHT_BLUE, rand())
    strokes.push(stroke(x, y, tangentAngle, randRange(0.01, 0.03), randRange(-0.005, 0.005), c, randRange(0.4, 0.7), randRange(1.5, 3), 1, 0.5, 0.003))
  }

  // ===== SECONDARY VORTEX =====
  // Smaller swirl in upper right area, around (0.62, 0.22)
  const svx = 0.62, svy = 0.22
  const spiralStrokes2 = 45
  for (let i = 0; i < spiralStrokes2; i++) {
    const t = i / spiralStrokes2
    const angleStart = t * Math.PI * 3 + 1
    const radius = 0.07 * (1 - t * 0.8)
    const x = svx + Math.cos(angleStart) * radius
    const y = svy + Math.sin(angleStart) * radius * 0.8
    const tangentAngle = angleStart + Math.PI / 2 + 0.2
    const c = lerpC(COBALT, MID_BLUE, t)
    const alpha = randRange(0.3, 0.55)
    strokes.push(stroke(x, y, tangentAngle, randRange(0.015, 0.035), randRange(-0.006, 0.006), c, alpha, randRange(1.5, 3.5), 2, randRange(0.3, 0.5), randRange(0.001, 0.004)))
  }

  // ===== FLOW STROKES connecting features =====
  // Strokes that flow from the main vortex toward the secondary vortex (the "river" in the sky)
  for (let i = 0; i < 40; i++) {
    const t = rand()
    // Path curves from vortex1 toward vortex2
    const x = lerp(vcx, svx, t) + randRange(-0.04, 0.04)
    const y = lerp(vcy, svy, t) + randRange(-0.03, 0.03) - t * 0.03
    // Direction follows the flow
    const dx = svx - vcx, dy = svy - vcy
    const angle = Math.atan2(dy, dx) + randRange(-0.4, 0.4)
    const c = lerpC(MID_BLUE, COBALT, t + randRange(-0.2, 0.2))
    strokes.push(stroke(x, y, angle, randRange(0.03, 0.06), randRange(-0.01, 0.01), c, randRange(0.2, 0.45), randRange(1.5, 3.5), 3, randRange(0.2, 0.4), randRange(0.001, 0.003)))
  }

  // Flow from secondary vortex toward moon area (0.82, 0.14)
  for (let i = 0; i < 30; i++) {
    const t = rand()
    const x = lerp(svx, 0.82, t) + randRange(-0.03, 0.03)
    const y = lerp(svy, 0.14, t) + randRange(-0.02, 0.02)
    const angle = Math.atan2(0.14 - svy, 0.82 - svx) + randRange(-0.3, 0.3)
    const c = lerpC(COBALT, MID_BLUE, t + randRange(-0.2, 0.2))
    strokes.push(stroke(x, y, angle, randRange(0.025, 0.055), randRange(-0.008, 0.008), c, randRange(0.2, 0.4), randRange(1.5, 3), 4, randRange(0.2, 0.35), randRange(0.001, 0.003)))
  }

  // ===== BRIGHT STAR (upper-left, 0.14, 0.15) =====
  // Van Gogh's largest star - radiating short strokes + concentric halo
  buildStar(0.14, 0.15, 1.3, 5)

  // ===== STAR (above vortex, 0.40, 0.13) =====
  buildStar(0.40, 0.13, 1.0, 4)

  // ===== STAR (upper area, 0.28, 0.10) =====
  buildStar(0.28, 0.10, 0.8, 4)

  // ===== STAR (between vortices, 0.52, 0.18) =====
  buildStar(0.52, 0.18, 0.7, 4)

  // ===== STAR (right side, 0.72, 0.28) =====
  buildStar(0.72, 0.28, 0.9, 4)

  // ===== STAR (right horizon area, 0.80, 0.48) =====
  buildStar(0.80, 0.48, 0.6, 3)

  // ===== STAR (far left, 0.06, 0.30) =====
  buildStar(0.06, 0.30, 0.5, 3)

  // ===== STAR (top center, 0.48, 0.06) =====
  buildStar(0.48, 0.06, 0.6, 3)

  // ===== STAR (near moon, 0.90, 0.18) =====
  buildStar(0.90, 0.18, 0.5, 3)

  // ===== MOON (upper right, 0.84, 0.12) =====
  buildMoon(0.84, 0.12)

  // ===== ADDITIONAL SKY TEXTURE STROKES =====
  // Fill in gaps with smaller strokes to create the dense textured sky
  for (let i = 0; i < 120; i++) {
    const x = rand()
    const y = randRange(0.03, 0.65)
    // Avoid placing on top of existing features
    const distToVortex = Math.hypot(x - vcx, (y - vcy))
    const distToSVortex = Math.hypot(x - svx, (y - svy))
    if (distToVortex < 0.05 || distToSVortex < 0.03) continue

    const angle = randRange(-0.5, 0.5)
    const c = lerpC(DEEP_BLUE, COBALT, rand())
    strokes.push(stroke(x, y, angle, randRange(0.01, 0.03), randRange(-0.005, 0.005), c, randRange(0.15, 0.35), randRange(1, 2.5), 0, randRange(0.15, 0.3), randRange(0.001, 0.002)))
  }
}

function buildStar(cx: number, cy: number, size: number, rings: number) {
  // Radiating short strokes from center
  const rayCount = Math.floor(8 + size * 6)
  for (let i = 0; i < rayCount; i++) {
    const angle = (i / rayCount) * Math.PI * 2 + Math.random() * 0.3
    const len = (0.01 + size * 0.012) * (0.7 + Math.random() * 0.6)
    const x = cx + Math.cos(angle) * len * 0.3
    const y = cy + Math.sin(angle) * len * 0.3
    const c = lerpC(CAD_YELLOW, PALE_YELLOW, Math.random())
    strokes.push(stroke(x, y, angle, len, (Math.random() - 0.5) * 0.005, c, 0.6 + Math.random() * 0.3, 1.5 + size, 10, 0.2, 0.001))
  }

  // Concentric halo rings (not perfect circles - made of curved strokes)
  for (let r = 0; r < rings; r++) {
    const radius = (0.01 + r * 0.008) * size * 2.5
    const segments = Math.floor(10 + r * 4)
    for (let i = 0; i < segments; i++) {
      const a = (i / segments) * Math.PI * 2
      // Skip some segments to make it look like brush strokes, not smooth circles
      if (Math.random() < 0.25) continue

      const x = cx + Math.cos(a) * radius
      const y = cy + Math.sin(a) * radius * 0.85
      // Tangent to the circle
      const tangentAngle = a + Math.PI / 2 + (Math.random() - 0.5) * 0.5
      const len = (Math.PI * 2 * radius / segments) * (0.8 + Math.random() * 0.5)
      const curve = (Math.random() - 0.5) * 0.004
      const t = r / rings
      const c = lerpC(WARM_WHITE, PALE_YELLOW, t * 0.7 + Math.random() * 0.3)
      const alpha = (0.5 - t * 0.25) * (0.8 + Math.random() * 0.2)
      strokes.push(stroke(x, y, tangentAngle, len, curve, c, alpha, 1.2 + (1 - t) * 1.5, 10 + r, 0.15 + r * 0.05, 0.001 + r * 0.001))
    }
  }
}

function buildMoon(cx: number, cy: number) {
  // Moon has a bright crescent shape with radiating glow strokes
  // The crescent is created by strokes concentrated on one side

  // Radiating glow strokes (like a sun in the painting)
  const rayCount = 30
  for (let i = 0; i < rayCount; i++) {
    const angle = (i / rayCount) * Math.PI * 2
    // Strokes are longer and brighter on the left side of the moon (the lit part)
    const litFactor = 0.5 + 0.5 * Math.cos(angle + 0.5)  // brighter at angle ~π
    const len = 0.02 + litFactor * 0.025
    const x = cx + Math.cos(angle) * 0.008
    const y = cy + Math.sin(angle) * 0.006
    const c = lerpC(CAD_YELLOW, PALE_YELLOW, litFactor * 0.5 + Math.random() * 0.3)
    strokes.push(stroke(x, y, angle, len, (Math.random() - 0.5) * 0.003, c, 0.5 + litFactor * 0.3, 1.5 + litFactor, 20, 0.12, 0.001))
  }

  // Concentric halo rings
  for (let r = 0; r < 6; r++) {
    const radius = 0.015 + r * 0.01
    const segments = Math.floor(14 + r * 5)
    for (let i = 0; i < segments; i++) {
      if (Math.random() < 0.2) continue
      const a = (i / segments) * Math.PI * 2
      const x = cx + Math.cos(a) * radius
      const y = cy + Math.sin(a) * radius * 0.85
      const tangentAngle = a + Math.PI / 2 + (Math.random() - 0.5) * 0.4
      const len = (Math.PI * 2 * radius / segments) * (0.8 + Math.random() * 0.4)
      const t = r / 6
      const c = lerpC(WHITE_YELLOW, PALE_YELLOW, t * 0.5 + Math.random() * 0.3)
      const alpha = (0.6 - t * 0.3) * (0.7 + Math.random() * 0.3)
      strokes.push(stroke(x, y, tangentAngle, len, (Math.random() - 0.5) * 0.003, c, alpha, 1.5 + (1 - t), 20 + r, 0.1 + r * 0.03, 0.001))
    }
  }
}

// Seeded PRNG for deterministic stroke placement
function mulberry32(a: number) {
  return function() {
    a |= 0; a = a + 0x6D2B79F5 | 0
    let t = Math.imul(a ^ a >>> 15, 1 | a)
    t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t
    return ((t ^ t >>> 14) >>> 0) / 4294967296
  }
}

// ==================== Rendering ====================

function drawStroke(s: BrushStroke, energy: {bass:number;mid:number;high:number;avg:number}) {
  if (!ctx) return

  // Apply subtle wave animation
  const waveOffset = Math.sin(th * s.waveSpeed + s.waveGroup * 0.7) * s.waveAmp * (1 + energy.avg * 2)
  const animAngle = s.angle + Math.sin(th * s.waveSpeed * 0.7 + s.x * 5) * 0.05 * (1 + energy.mid)

  const scale = Math.min(W, H)
  const px = s.x * W
  const py = s.y * H * 0.65  // sky region = top 65%
  const lenPx = s.len * scale
  const curvePx = s.curve * scale

  // End point
  const ex = px + Math.cos(animAngle) * lenPx
  const ey = (py + waveOffset * H) + Math.sin(animAngle) * lenPx

  // Control point (perpendicular offset for curvature)
  const mx = (px + ex) / 2 + Math.cos(animAngle + Math.PI / 2) * curvePx
  const my = ((py + waveOffset * H) + ey) / 2 + Math.sin(animAngle + Math.PI / 2) * curvePx

  ctx.beginPath()
  ctx.strokeStyle = cStr(s.color, s.alpha)
  ctx.lineWidth = s.width
  ctx.lineCap = 'round'
  ctx.moveTo(px, py + waveOffset * H)
  ctx.quadraticCurveTo(mx, my, ex, ey)
  ctx.stroke()
}

function drawBackground() {
  if (!ctx) return
  // Deep indigo gradient for sky
  const grad = ctx.createLinearGradient(0, 0, 0, H * 0.65)
  grad.addColorStop(0, '#0c0a2e')
  grad.addColorStop(0.3, '#11103d')
  grad.addColorStop(0.6, '#161455')
  grad.addColorStop(1, '#1a1860')
  ctx.fillStyle = grad
  ctx.fillRect(0, 0, W, H * 0.65)
  // Dark below (village area placeholder)
  ctx.fillStyle = '#060410'
  ctx.fillRect(0, H * 0.65, W, H * 0.35)
}

function drawStarCores(energy: {bass:number;mid:number;high:number;avg:number}) {
  if (!ctx) return
  // Draw bright core dots for each star and moon
  const cores = [
    { x: 0.14, y: 0.15, s: 1.3 },
    { x: 0.40, y: 0.13, s: 1.0 },
    { x: 0.28, y: 0.10, s: 0.8 },
    { x: 0.52, y: 0.18, s: 0.7 },
    { x: 0.72, y: 0.28, s: 0.9 },
    { x: 0.80, y: 0.48, s: 0.6 },
    { x: 0.06, y: 0.30, s: 0.5 },
    { x: 0.48, y: 0.06, s: 0.6 },
    { x: 0.90, y: 0.18, s: 0.5 },
  ]

  for (const c of cores) {
    const px = c.x * W
    const py = c.y * H * 0.65
    const r = c.s * 5 * (1 + energy.high * 0.3)
    const grad = ctx.createRadialGradient(px, py, 0, px, py, r)
    grad.addColorStop(0, cStr(WARM_WHITE, 0.95))
    grad.addColorStop(0.3, cStr(CAD_YELLOW, 0.7))
    grad.addColorStop(1, cStr(CAD_YELLOW, 0))
    ctx.fillStyle = grad
    ctx.beginPath()
    ctx.arc(px, py, r, 0, Math.PI * 2)
    ctx.fill()
  }

  // Moon core (crescent)
  const mx = 0.84 * W, my = 0.12 * H * 0.65
  const mr = 10 * (1 + energy.mid * 0.2)
  const grad = ctx.createRadialGradient(mx, my, 0, mx, my, mr * 3)
  grad.addColorStop(0, cStr(WARM_WHITE, 0.95))
  grad.addColorStop(0.4, cStr(CAD_YELLOW, 0.6))
  grad.addColorStop(1, cStr(CAD_YELLOW, 0))
  ctx.fillStyle = grad
  ctx.beginPath()
  ctx.arc(mx, my, mr * 3, 0, Math.PI * 2)
  ctx.fill()
  // Crescent cutout
  ctx.save()
  ctx.globalCompositeOperation = 'destination-out'
  ctx.beginPath()
  ctx.arc(mx + mr * 0.5, my - mr * 0.15, mr * 0.8, 0, Math.PI * 2)
  ctx.fill()
  ctx.restore()
}

function animate() {
  if (!ctx) return
  const energy = getAudioEnergy()

  drawBackground()

  // Draw all strokes
  for (const s of strokes) {
    drawStroke(s, energy)
  }

  // Star and moon cores on top
  drawStarCores(energy)

  th += 0.005 * (1 + energy.avg * 0.5)
  animId = requestAnimationFrame(animate)
}

function resize() {
  const canvas = canvasRef.value
  if (!canvas) return
  W = window.innerWidth
  H = window.innerHeight
  canvas.width = W
  canvas.height = H
  buildStrokes()
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
html, body { overflow: hidden; background: #0c0a2e }
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
