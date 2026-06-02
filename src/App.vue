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
      <span>✨ 梵高《星月夜》抽象线条 · Vue + Canvas</span>
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

// ==================== Abstract Line System ====================

// 所有元素统一为流动线条，通过运动场驱动

interface FlowLine {
  points: { x: number; y: number }[]
  color: string
  width: number
  opacity: number
  speed: number
  life: number
  maxLife: number
  maxPoints: number
  phase: number
  amplitude: number
  freq: number
  element: 'sky' | 'swirl' | 'star' | 'village' | 'cypress' | 'moon'
}

// 色板 —— 全部用线条颜色
const PALETTES = {
  sky: ['#1a237e', '#283593', '#1565c0', '#0d47a1', '#01579b', '#1b3a5c', '#1e4880', '#c5cae9', '#9fa8da', '#7986cb'],
  swirl: ['#42a5f5', '#64b5f6', '#90caf9', '#bbdefb', '#1565c0', '#1976d2', '#1e88e5', '#2196f3', '#ffee58', '#fff176', '#fdd835'],
  star: ['#ffee58', '#fff176', '#fff9c4', '#ffffff', '#fdd835', '#f9a825', '#ffeb3b', '#ffc107'],
  village: ['#1b5e20', '#2e7d32', '#388e3c', '#1a1a2e', '#16213e', '#0f3460', '#4a6741'],
  cypress: ['#0d3b0d', '#1b5e20', '#2e7d32', '#0a2a0a', '#1a4a1a', '#0f3f0f'],
  moon: ['#fff9c4', '#ffee58', '#fff176', '#fdd835', '#fffde7', '#ffe082'],
}

function pick(arr: string[]) { return arr[Math.floor(Math.random() * arr.length)] }

let lines: FlowLine[] = []
let time = 0

// 漩涡中心
const swirlCenters = [
  { x: 0.25, y: 0.2, r: 0.12 },
  { x: 0.65, y: 0.15, r: 0.08 },
  { x: 0.45, y: 0.4, r: 0.15 },
  { x: 0.8, y: 0.3, r: 0.07 },
]

// 星星位置（用辐射线簇表示）
const starPositions = [
  { x: 0.15, y: 0.12 }, { x: 0.3, y: 0.08 }, { x: 0.5, y: 0.05 },
  { x: 0.7, y: 0.1 }, { x: 0.35, y: 0.25 }, { x: 0.6, y: 0.22 },
  { x: 0.12, y: 0.35 }, { x: 0.85, y: 0.18 }, { x: 0.55, y: 0.35 },
  { x: 0.25, y: 0.42 }, { x: 0.78, y: 0.38 }, { x: 0.42, y: 0.15 },
]

// 月亮位置
const moonPos = { x: 0.85, y: 0.12 }

function createLine(element: FlowLine['element']): FlowLine {
  const maxPoints = 30 + Math.floor(Math.random() * 50)
  let startX: number, startY: number, palette: string[]
  let spd = 0.5 + Math.random() * 2
  let amp = 5 + Math.random() * 15
  let freq = 0.02 + Math.random() * 0.04
  let w = 0.5 + Math.random() * 3
  let op = 0.3 + Math.random() * 0.5

  switch (element) {
    case 'star': {
      const star = starPositions[Math.floor(Math.random() * starPositions.length)]
      // 从星星位置出发，向四周辐射
      startX = star.x * width + (Math.random() - 0.5) * 20
      startY = star.y * height + (Math.random() - 0.5) * 20
      palette = PALETTES.star
      w = 0.5 + Math.random() * 2
      amp = 3 + Math.random() * 8
      spd = 0.3 + Math.random() * 1.5
      break
    }
    case 'swirl': {
      // 在漩涡区域附近出发
      const sc = swirlCenters[Math.floor(Math.random() * swirlCenters.length)]
      const angle = Math.random() * Math.PI * 2
      const dist = sc.r * width * (0.3 + Math.random() * 0.7)
      startX = sc.x * width + Math.cos(angle) * dist
      startY = sc.y * height + Math.sin(angle) * dist
      palette = PALETTES.swirl
      amp = 10 + Math.random() * 25
      spd = 0.8 + Math.random() * 2.5
      w = 0.5 + Math.random() * 2.5
      break
    }
    case 'moon': {
      // 月亮用同心弧线表示
      const angle = Math.random() * Math.PI * 2
      startX = moonPos.x * width + Math.cos(angle) * (20 + Math.random() * 40)
      startY = moonPos.y * height + Math.sin(angle) * (20 + Math.random() * 40)
      palette = PALETTES.moon
      w = 0.5 + Math.random() * 1.5
      spd = 0.2 + Math.random() * 0.8
      amp = 3 + Math.random() * 6
      break
    }
    case 'village': {
      startX = Math.random() * width
      startY = height * (0.7 + Math.random() * 0.25)
      palette = PALETTES.village
      spd = 0.3 + Math.random() * 1.2
      amp = 2 + Math.random() * 8
      w = 0.3 + Math.random() * 2
      break
    }
    case 'cypress': {
      // 柏树 —— 从底部向上的竖向流动线
      startX = width * 0.88 + (Math.random() - 0.5) * 40
      startY = height * (0.3 + Math.random() * 0.5)
      palette = PALETTES.cypress
      spd = 0.2 + Math.random() * 1
      amp = 3 + Math.random() * 10
      w = 0.5 + Math.random() * 2
      op = 0.4 + Math.random() * 0.4
      break
    }
    default: { // sky
      startX = Math.random() * width
      startY = Math.random() * height * 0.7
      palette = PALETTES.sky
      amp = 8 + Math.random() * 20
      spd = 0.5 + Math.random() * 2
      break
    }
  }

  return {
    points: [{ x: startX, y: startY }],
    color: pick(palette),
    width: w,
    opacity: op,
    speed: spd,
    life: 0,
    maxLife: 150 + Math.floor(Math.random() * 250),
    maxPoints,
    phase: Math.random() * Math.PI * 2,
    amplitude: amp,
    freq,
    element,
  }
}

function pickElement(): FlowLine['element'] {
  const r = Math.random()
  if (r < 0.2) return 'sky'
  if (r < 0.45) return 'swirl'
  if (r < 0.6) return 'star'
  if (r < 0.72) return 'village'
  if (r < 0.8) return 'cypress'
  if (r < 0.85) return 'moon'
  return 'sky'
}

// 线条运动 —— 每个元素有不同的流动场
function advanceLine(line: FlowLine) {
  const last = line.points[line.points.length - 1]
  const energy = getAudioEnergy()
  const audioBoost = 1 + energy.avg * 3
  let dx = 0, dy = 0
  const step = line.life

  switch (line.element) {
    case 'swirl': {
      // 找最近的漩涡中心，沿切线旋转
      let nearestIdx = 0, minDist = Infinity
      for (let i = 0; i < swirlCenters.length; i++) {
        const sc = swirlCenters[i]
        const d = Math.hypot(last.x - sc.x * width, last.y - sc.y * height)
        if (d < minDist) { minDist = d; nearestIdx = i }
      }
      const sc = swirlCenters[nearestIdx]
      const cx = sc.x * width, cy = sc.y * height
      const angle = Math.atan2(last.y - cy, last.x - cx)
      const tangent = angle + Math.PI / 2
      const pull = minDist > sc.r * width ? 0.015 : -0.008
      dx = Math.cos(tangent) * line.speed * audioBoost + (cx - last.x) * pull
      dy = Math.sin(tangent) * line.speed * audioBoost + (cy - last.y) * pull
      dx += Math.sin(step * line.freq + line.phase) * line.amplitude * 0.04 * audioBoost
      dy += Math.cos(step * line.freq + line.phase) * line.amplitude * 0.04 * audioBoost
      break
    }
    case 'star': {
      // 从最近的星星向外辐射
      let nearestStar = starPositions[0], minD = Infinity
      for (const s of starPositions) {
        const d = Math.hypot(last.x - s.x * width, last.y - s.y * height)
        if (d < minD) { minD = d; nearestStar = s }
      }
      const angle = Math.atan2(last.y - nearestStar.y * height, last.x - nearestStar.x * width)
        + Math.sin(step * 0.03 + line.phase) * 0.4
      dx = Math.cos(angle) * line.speed * 0.6 * audioBoost
      dy = Math.sin(angle) * line.speed * 0.6 * audioBoost
      dx += Math.sin(step * 0.08 + line.phase) * 0.3
      dy += Math.cos(step * 0.08 + line.phase) * 0.3
      break
    }
    case 'moon': {
      // 绕月亮做弧线运动
      const mx = moonPos.x * width, my = moonPos.y * height
      const angle = Math.atan2(last.y - my, last.x - mx)
      // 沿切线方向缓慢旋转 + 向外扩展
      const tangent = angle + Math.PI / 2
      dx = Math.cos(tangent) * line.speed * audioBoost * 0.5
      dy = Math.sin(tangent) * line.speed * audioBoost * 0.5
      // 微微向外
      dx += Math.cos(angle) * 0.2
      dy += Math.sin(angle) * 0.2
      dx += Math.sin(step * 0.05 + line.phase) * 0.3
      dy += Math.cos(step * 0.05 + line.phase) * 0.3
      break
    }
    case 'village': {
      // 水平方向为主，微起伏
      dx = line.speed * audioBoost * 0.8 * (Math.random() > 0.5 ? 1 : -1)
      dy = Math.sin(step * line.freq * 2 + line.phase) * 0.6 * audioBoost
      break
    }
    case 'cypress': {
      // 向上流动为主，微横向波动
      dy = -line.speed * audioBoost
      dx = Math.sin(step * line.freq + line.phase + time * 0.02) * line.amplitude * 0.08 * audioBoost
      break
    }
    default: { // sky
      dx = line.speed * 1.5 * audioBoost
      dy = Math.sin(step * line.freq + line.phase + time * 0.008) * line.amplitude * 0.12 * audioBoost
      break
    }
  }

  line.points.push({ x: last.x + dx, y: last.y + dy })
  if (line.points.length > line.maxPoints) line.points.shift()
}

// 初始化场景
function initScene() {
  lines = []
  // 各类元素分配不同数量
  const counts: Record<FlowLine['element'], number> = {
    sky: 40, swirl: 60, star: 30, village: 25, cypress: 20, moon: 15,
  }
  for (const [el, count] of Object.entries(counts)) {
    for (let i = 0; i < count; i++) {
      const line = createLine(el as FlowLine['element'])
      // 预推进
      line.life = Math.floor(Math.random() * line.maxLife)
      for (let j = 0; j < line.life && j < line.maxPoints; j++) {
        advanceLine(line)
      }
      lines.push(line)
    }
  }
}

// 绘制背景（深色渐变）
function drawBackground() {
  if (!ctx) return
  const grad = ctx.createLinearGradient(0, 0, 0, height)
  grad.addColorStop(0, '#050816')
  grad.addColorStop(0.3, '#0a0e2a')
  grad.addColorStop(0.6, '#0d1b3e')
  grad.addColorStop(0.75, '#0a1a0a')
  grad.addColorStop(1, '#061206')
  ctx.fillStyle = grad
  ctx.fillRect(0, 0, width, height)
}

// 绘制单条流动线
function drawLine(line: FlowLine) {
  if (!ctx || line.points.length < 3) return
  const pts = line.points
  const lifeRatio = line.life / line.maxLife
  const alpha = lifeRatio < 0.1 ? lifeRatio / 0.1 : lifeRatio > 0.8 ? (1 - lifeRatio) / 0.2 : 1
  const energy = getAudioEnergy()

  ctx.beginPath()
  ctx.strokeStyle = line.color
  ctx.lineWidth = line.width * (1 + energy.avg * 1.5)
  ctx.globalAlpha = alpha * line.opacity
  ctx.lineCap = 'round'
  ctx.lineJoin = 'round'

  ctx.moveTo(pts[0].x, pts[0].y)
  for (let i = 1; i < pts.length - 1; i++) {
    const xc = (pts[i].x + pts[i + 1].x) / 2
    const yc = (pts[i].y + pts[i + 1].y) / 2
    ctx.quadraticCurveTo(pts[i].x, pts[i].y, xc, yc)
  }
  ctx.lineTo(pts[pts.length - 1].x, pts[pts.length - 1].y)
  ctx.stroke()
  ctx.globalAlpha = 1
}

// 漩涡区域额外的装饰螺旋线
function drawSwirlSpirals() {
  if (!ctx) return
  const energy = getAudioEnergy()

  for (const sc of swirlCenters) {
    const cx = sc.x * width, cy = sc.y * height
    const baseR = sc.r * width

    // 每个漩涡画几条引导螺旋线
    for (let s = 0; s < 3; s++) {
      ctx.beginPath()
      ctx.strokeStyle = `rgba(100, 180, 255, ${0.04 + energy.mid * 0.03})`
      ctx.lineWidth = 0.5 + energy.bass * 0.5
      const offset = time * 0.008 * (s + 1) + s * Math.PI * 2 / 3
      for (let a = 0; a < Math.PI * 8; a += 0.08) {
        const r = (a / (Math.PI * 8)) * baseR * (1 + energy.mid * 0.5)
        const x = cx + Math.cos(a + offset) * r
        const y = cy + Math.sin(a + offset) * r
        if (a === 0) ctx.moveTo(x, y)
        else ctx.lineTo(x, y)
      }
      ctx.stroke()
    }
  }
}

// 星星的辐射线装饰
function drawStarBursts() {
  if (!ctx) return
  const energy = getAudioEnergy()

  for (const sp of starPositions) {
    const sx = sp.x * width, sy = sp.y * height
    const numRays = 6 + Math.floor(energy.high * 4)
    const baseLen = 8 + energy.high * 15
    const pulse = 1 + Math.sin(time * 0.04 + sp.x * 10) * 0.3

    for (let r = 0; r < numRays; r++) {
      const angle = (r / numRays) * Math.PI * 2 + time * 0.01 + sp.y * 5
      const len = baseLen * pulse * (0.5 + Math.random() * 0.5)
      ctx.beginPath()
      ctx.strokeStyle = `rgba(255, 238, 88, ${0.15 + energy.high * 0.2})`
      ctx.lineWidth = 0.3 + Math.random() * 0.5
      ctx.moveTo(sx + Math.cos(angle) * 2, sy + Math.sin(angle) * 2)
      ctx.lineTo(sx + Math.cos(angle) * len, sy + Math.sin(angle) * len)
      ctx.stroke()
    }
  }
}

// 月亮的同心弧线
function drawMoonArcs() {
  if (!ctx) return
  const energy = getAudioEnergy()
  const mx = moonPos.x * width, my = moonPos.y * height
  const numArcs = 6 + Math.floor(energy.bass * 4)

  for (let i = 0; i < numArcs; i++) {
    const r = 15 + i * 8 + energy.bass * 10
    const startAngle = time * 0.005 + i * 0.3
    const arcLen = Math.PI * (0.3 + Math.random() * 0.5)
    ctx.beginPath()
    ctx.strokeStyle = `rgba(255, 249, 196, ${0.06 + (numArcs - i) * 0.02})`
    ctx.lineWidth = 0.3 + Math.random() * 0.8
    ctx.arc(mx, my, r, startAngle, startAngle + arcLen)
    ctx.stroke()
  }
}

// 村庄区域 —— 山丘轮廓用线条
function drawVillageContours() {
  if (!ctx) return
  const energy = getAudioEnergy()
  const baseY = height * 0.75

  // 多层山丘轮廓线
  for (let layer = 0; layer < 4; layer++) {
    const yOff = layer * 12
    const alpha = 0.08 - layer * 0.015
    ctx.beginPath()
    ctx.strokeStyle = `rgba(26, 94, 32, ${alpha + energy.mid * 0.03})`
    ctx.lineWidth = 0.5 + layer * 0.3
    for (let x = 0; x <= width; x += 2) {
      const y = baseY - yOff
        - Math.sin(x * 0.008 + layer) * 15
        - Math.sin(x * 0.015 + 1 + layer) * 10
        - Math.sin(x * 0.003 + layer * 0.5) * 25
        + Math.sin(time * 0.01 + x * 0.01 + layer) * 2
      if (x === 0) ctx.moveTo(x, y)
      else ctx.lineTo(x, y)
    }
    ctx.stroke()
  }

  // 教堂尖塔 —— 用线条勾勒
  const cx = width * 0.45
  const cBase = baseY - 20
  ctx.strokeStyle = 'rgba(22, 33, 62, 0.4)'
  ctx.lineWidth = 1
  ctx.beginPath()
  ctx.moveTo(cx, cBase - 80)
  ctx.lineTo(cx - 10, cBase)
  ctx.moveTo(cx, cBase - 80)
  ctx.lineTo(cx + 10, cBase)
  ctx.stroke()

  // 小房子的线条
  for (let i = 0; i < 12; i++) {
    const hx = width * (0.05 + i * 0.075) + Math.sin(i * 2.5) * 20
    const hy = cBase - Math.sin(hx * 0.01) * 8
    const hw = 14 + (i % 3) * 4
    const hh = 10 + (i % 2) * 6
    ctx.strokeStyle = `rgba(22, 33, 62, ${0.3 + Math.sin(time * 0.03 + i) * 0.1})`
    ctx.lineWidth = 0.6
    // 房子轮廓
    ctx.beginPath()
    ctx.moveTo(hx - hw / 2, hy)
    ctx.lineTo(hx - hw / 2, hy - hh)
    ctx.lineTo(hx, hy - hh - 10)
    ctx.lineTo(hx + hw / 2, hy - hh)
    ctx.lineTo(hx + hw / 2, hy)
    ctx.stroke()
    // 窗户（小十字线）
    ctx.strokeStyle = `rgba(255, 238, 88, ${0.2 + Math.sin(time * 0.05 + i) * 0.1})`
    ctx.lineWidth = 0.4
    ctx.beginPath()
    ctx.moveTo(hx - 2, hy - hh + 3)
    ctx.lineTo(hx + 2, hy - hh + 3)
    ctx.moveTo(hx, hy - hh + 1)
    ctx.lineTo(hx, hy - hh + 5)
    ctx.stroke()
  }
}

// 柏树 —— 用密集的垂直流动线条
function drawCypressLines() {
  if (!ctx) return
  const energy = getAudioEnergy()
  const tx = width * 0.88
  const baseY = height * 0.78
  const treeH = 180 + energy.avg * 30
  const topY = baseY - treeH

  // 柏树轮廓引导线
  const numLines = 12 + Math.floor(energy.bass * 6)
  for (let i = 0; i < numLines; i++) {
    const t = i / numLines
    const x = tx - 30 + t * 60
    // 宽度在顶部窄底部宽
    const widthFactor = Math.abs(t - 0.5) * 2
    const xOff = (widthFactor * 0.5 + 0.2) * 30
    const wobble = Math.sin(time * 0.02 + i * 0.5) * 3 * (1 + energy.avg)

    ctx.beginPath()
    ctx.strokeStyle = `rgba(13, 59, 13, ${0.15 + Math.random() * 0.1})`
    ctx.lineWidth = 0.3 + Math.random() * 0.5
    ctx.moveTo(x + wobble, topY + Math.random() * 20)
    for (let y = topY; y < baseY; y += 4) {
      const progress = (y - topY) / treeH
      // xExpand is part of the shape calculation
      const wx = Math.sin(y * 0.03 + time * 0.03 + i) * 2 * (1 + energy.avg)
      ctx.lineTo(tx + (x - tx) * (1 + progress * 0.5) + wx, y)
    }
    ctx.stroke()
  }
}

// 主循环
function animate() {
  if (!ctx) return
  time++

  drawBackground()

  // 装饰层（半透明引导线）
  drawSwirlSpirals()
  drawStarBursts()
  drawMoonArcs()
  drawVillageContours()
  drawCypressLines()

  // 更新和绘制流动线条
  const energy = getAudioEnergy()

  for (let i = lines.length - 1; i >= 0; i--) {
    const line = lines[i]
    line.life++

    if (line.life < line.maxLife) {
      advanceLine(line)
      drawLine(line)
    } else {
      lines[i] = createLine(pickElement())
    }
  }

  // 音频能量高时增加线条
  if (energy.avg > 0.25 && lines.length < 250) {
    const extra = Math.floor(energy.avg * 4)
    for (let i = 0; i < extra; i++) {
      lines.push(createLine(pickElement()))
    }
  }

  animId = requestAnimationFrame(animate)
}

function resize() {
  const canvas = canvasRef.value
  if (!canvas) return
  width = window.innerWidth
  height = window.innerHeight
  canvas.width = width
  canvas.height = height
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
html, body { overflow: hidden; background: #050816 }
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
