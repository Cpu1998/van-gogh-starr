<template>
  <div class="app">
    <!-- 音乐控制栏 -->
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
    <!-- 画布 -->
    <canvas ref="canvasRef"></canvas>
    <!-- 底部信息 -->
    <div class="info">
      <span>✨ 梵高《星月夜》流动线条 · Vue + Canvas</span>
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
  // 不连接 destination，避免回声

  sourceType = 'mic'
  isPlaying.value = true
}

function cleanupAudio() {
  if (audioElement) {
    audioElement.pause()
    audioElement.src = ''
    audioElement = null
  }
  if (micStream) {
    micStream.getTracks().forEach(t => t.stop())
    micStream = null
  }
  if (audioCtx && audioCtx.state !== 'closed') {
    audioCtx.close()
  }
  audioCtx = null
  analyser = null
  audioSource = null
  sourceType = 'none'
  isPlaying.value = false
}

async function toggleAudio() {
  if (isPlaying.value) {
    cleanupAudio()
  } else {
    await initMicAudio()
  }
}

function handleAudioUpload(e: Event) {
  const input = e.target as HTMLInputElement
  const file = input.files?.[0]
  if (file) initAudioFromFile(file)
}

function getAudioEnergy(): { bass: number; mid: number; high: number; avg: number } {
  if (!analyser || !dataArray.length) {
    return { bass: 0, mid: 0, high: 0, avg: 0 }
  }
  analyser.getByteFrequencyData(dataArray)
  const len = dataArray.length
  const third = Math.floor(len / 3)

  let bass = 0, mid = 0, high = 0
  for (let i = 0; i < third; i++) bass += dataArray[i]
  for (let i = third; i < third * 2; i++) mid += dataArray[i]
  for (let i = third * 2; i < len; i++) high += dataArray[i]

  bass /= third * 255
  mid /= third * 255
  high /= (len - third * 2) * 255
  const avg = (bass + mid + high) / 3

  return { bass, mid, high, avg }
}

// ==================== Flow Lines ====================
interface FlowLine {
  points: { x: number; y: number }[]
  color: string
  width: number
  speed: number
  life: number
  maxLife: number
  phase: number
  amplitude: number
  freq: number
  layer: 'sky' | 'swirl' | 'village' | 'star'
}

const VAN_GOGH_COLORS = {
  sky: [
    '#1a237e', '#283593', '#1565c0', '#0d47a1', '#01579b',
    '#1b3a5c', '#1e4880', '#0b3d91', '#1a3a6c', '#152d50',
    '#c5cae9', '#9fa8da', '#7986cb', '#5c6bc0', '#3f51b5',
  ],
  swirl: [
    '#42a5f5', '#64b5f6', '#90caf9', '#bbdefb', '#e3f2fd',
    '#1565c0', '#1976d2', '#1e88e5', '#2196f3',
    '#ffee58', '#fff176', '#fff59d', '#fdd835', '#f9a825',
  ],
  star: [
    '#ffee58', '#fff176', '#fff9c4', '#ffffff', '#fffde7',
    '#fdd835', '#f9a825', '#ffeb3b', '#ffc107',
  ],
  village: [
    '#1b5e20', '#2e7d32', '#388e3c', '#0d3b0d',
    '#1a1a2e', '#16213e', '#0f3460',
  ],
}

function randomColor(palette: string[]): string {
  return palette[Math.floor(Math.random() * palette.length)]
}

let lines: FlowLine[] = []
let time = 0

function createLine(layer: FlowLine['layer']): FlowLine {
  const maxPoints = 40 + Math.floor(Math.random() * 40)
  const startX = Math.random() * width
  const startY = layer === 'village'
    ? height * (0.7 + Math.random() * 0.25)
    : Math.random() * height * 0.75

  let palette: string[]
  switch (layer) {
    case 'star': palette = VAN_GOGH_COLORS.star; break
    case 'swirl': palette = VAN_GOGH_COLORS.swirl; break
    case 'village': palette = VAN_GOGH_COLORS.village; break
    default: palette = VAN_GOGH_COLORS.sky
  }

  return {
    points: [{ x: startX, y: startY }],
    color: randomColor(palette),
    width: layer === 'star' ? 2 + Math.random() * 3 : 1 + Math.random() * 4,
    speed: 0.5 + Math.random() * 2,
    life: 0,
    maxLife: 200 + Math.floor(Math.random() * 300),
    phase: Math.random() * Math.PI * 2,
    amplitude: layer === 'swirl' ? 15 + Math.random() * 25 : 5 + Math.random() * 15,
    freq: 0.02 + Math.random() * 0.04,
    layer,
  }
}

// 星空中的旋涡中心
interface Swirl {
  x: number
  y: number
  radius: number
  speed: number
  phase: number
}

const swirls: Swirl[] = []

// 星星位置
interface Star {
  x: number
  y: number
  size: number
  brightness: number
  phase: number
}

const stars: Star[] = []

function initScene() {
  // 创建旋涡
  swirls.length = 0
  swirls.push(
    { x: width * 0.25, y: height * 0.2, radius: 80, speed: 0.02, phase: 0 },
    { x: width * 0.65, y: height * 0.15, radius: 60, speed: 0.015, phase: 2 },
    { x: width * 0.45, y: height * 0.45, radius: 100, speed: 0.025, phase: 4 },
    { x: width * 0.8, y: height * 0.35, radius: 50, speed: 0.018, phase: 1 },
  )

  // 创建星星
  stars.length = 0
  for (let i = 0; i < 12; i++) {
    stars.push({
      x: width * (0.1 + Math.random() * 0.8),
      y: height * (0.05 + Math.random() * 0.45),
      size: 8 + Math.random() * 20,
      brightness: 0.5 + Math.random() * 0.5,
      phase: Math.random() * Math.PI * 2,
    })
  }

  // 创建初始线条
  lines.length = 0
  for (let i = 0; i < 80; i++) {
    const layer = pickLayer()
    const line = createLine(layer)
    line.life = Math.floor(Math.random() * line.maxLife) // 预推进
    for (let j = 0; j < line.life && j < maxPoints; j++) {
      advanceLine(line, j)
    }
    lines.push(line)
  }
}

function pickLayer(): FlowLine['layer'] {
  const r = Math.random()
  if (r < 0.35) return 'sky'
  if (r < 0.7) return 'swirl'
  if (r < 0.85) return 'star'
  return 'village'
}

function advanceLine(line: FlowLine, step: number) {
  const last = line.points[line.points.length - 1]
  const energy = getAudioEnergy()

  let dx: number, dy: number
  const audioBoost = 1 + energy.avg * 3

  switch (line.layer) {
    case 'swirl': {
      // 向最近的旋涡中心旋转
      let nearestSwirl = swirls[0]
      let minDist = Infinity
      for (const s of swirls) {
        const d = Math.hypot(last.x - s.x, last.y - s.y)
        if (d < minDist) { minDist = d; nearestSwirl = s }
      }
      const angle = Math.atan2(last.y - nearestSwirl.y, last.x - nearestSwirl.x)
      const tangent = angle + Math.PI / 2
      const pull = minDist > nearestSwirl.radius ? 0.02 : -0.01
      dx = Math.cos(tangent) * line.speed * audioBoost + (nearestSwirl.x - last.x) * pull
      dy = Math.sin(tangent) * line.speed * audioBoost + (nearestSwirl.y - last.y) * pull
      // 加波浪
      dx += Math.sin(step * line.freq + line.phase) * line.amplitude * 0.05 * audioBoost
      dy += Math.cos(step * line.freq + line.phase) * line.amplitude * 0.05 * audioBoost
      break
    }
    case 'star': {
      // 从星星向外辐射，带波动
      const star = stars[Math.floor(line.phase / Math.PI * 3) % stars.length]
      const angle = Math.atan2(last.y - star.y, last.x - star.x) + Math.sin(step * 0.05) * 0.3
      dx = Math.cos(angle) * line.speed * 0.8 * audioBoost
      dy = Math.sin(angle) * line.speed * 0.8 * audioBoost
      dx += Math.sin(step * 0.1 + line.phase) * 0.5
      dy += Math.cos(step * 0.1 + line.phase) * 0.5
      break
    }
    case 'village': {
      // 水平方向为主，带起伏
      dx = line.speed * audioBoost * (Math.random() > 0.5 ? 1 : -1)
      dy = Math.sin(step * line.freq * 2 + line.phase) * 0.8 * audioBoost
      break
    }
    default: { // sky
      // 大波浪横向流动
      dx = line.speed * 1.5 * audioBoost
      dy = Math.sin(step * line.freq + line.phase + time * 0.01) * line.amplitude * 0.15 * audioBoost
      break
    }
  }

  line.points.push({
    x: last.x + dx,
    y: last.y + dy,
  })

  // 限制点数
  if (line.points.length > 60) {
    line.points.shift()
  }
}

function drawBackground() {
  if (!ctx) return
  // 渐变背景：深蓝到更深蓝
  const grad = ctx.createLinearGradient(0, 0, 0, height)
  grad.addColorStop(0, '#0a0e2a')
  grad.addColorStop(0.3, '#0d1b3e')
  grad.addColorStop(0.6, '#102040')
  grad.addColorStop(0.75, '#1a3a1a')
  grad.addColorStop(1, '#0d2b0d')
  ctx.fillStyle = grad
  ctx.fillRect(0, 0, width, height)
}

function drawSwirls() {
  if (!ctx) return
  const energy = getAudioEnergy()

  for (const swirl of swirls) {
    swirl.phase += swirl.speed * (1 + energy.bass * 2)

    // 画旋涡的螺旋引导线（半透明）
    ctx.beginPath()
    ctx.strokeStyle = 'rgba(100, 180, 255, 0.03)'
    ctx.lineWidth = 1
    for (let a = 0; a < Math.PI * 6; a += 0.05) {
      const r = (a / (Math.PI * 6)) * swirl.radius * (1 + energy.mid)
      const x = swirl.x + Math.cos(a + swirl.phase) * r
      const y = swirl.y + Math.sin(a + swirl.phase) * r
      if (a === 0) ctx.moveTo(x, y)
      else ctx.lineTo(x, y)
    }
    ctx.stroke()
  }
}

function drawStars() {
  if (!ctx) return
  const energy = getAudioEnergy()

  for (const star of stars) {
    star.phase += 0.02
    const pulse = 1 + Math.sin(star.phase) * 0.3 + energy.high * 0.5
    const size = star.size * pulse

    // 光晕
    const grad = ctx.createRadialGradient(star.x, star.y, 0, star.x, star.y, size * 2)
    grad.addColorStop(0, `rgba(255, 238, 88, ${star.brightness * 0.8})`)
    grad.addColorStop(0.3, `rgba(255, 241, 118, ${star.brightness * 0.4})`)
    grad.addColorStop(0.6, `rgba(255, 249, 196, ${star.brightness * 0.15})`)
    grad.addColorStop(1, 'rgba(255, 249, 196, 0)')
    ctx.fillStyle = grad
    ctx.beginPath()
    ctx.arc(star.x, star.y, size * 2, 0, Math.PI * 2)
    ctx.fill()

    // 核心
    ctx.fillStyle = '#fffde7'
    ctx.beginPath()
    ctx.arc(star.x, star.y, size * 0.3, 0, Math.PI * 2)
    ctx.fill()
  }
}

function drawMoon() {
  if (!ctx) return
  const energy = getAudioEnergy()
  const mx = width * 0.85
  const my = height * 0.12
  const mr = 35 + energy.bass * 15

  // 月亮光晕
  const grad = ctx.createRadialGradient(mx, my, mr * 0.3, mx, my, mr * 3)
  grad.addColorStop(0, 'rgba(255, 249, 196, 0.9)')
  grad.addColorStop(0.2, 'rgba(255, 238, 88, 0.5)')
  grad.addColorStop(0.5, 'rgba(255, 238, 88, 0.15)')
  grad.addColorStop(1, 'rgba(255, 238, 88, 0)')
  ctx.fillStyle = grad
  ctx.beginPath()
  ctx.arc(mx, my, mr * 3, 0, Math.PI * 2)
  ctx.fill()

  // 月亮本体
  ctx.fillStyle = '#fff9c4'
  ctx.beginPath()
  ctx.arc(mx, my, mr, 0, Math.PI * 2)
  ctx.fill()
}

function drawLine(line: FlowLine) {
  if (!ctx || line.points.length < 3) return

  ctx.beginPath()
  ctx.strokeStyle = line.color
  ctx.lineWidth = line.width * (1 + getAudioEnergy().avg * 2)

  // 用二次贝塞尔曲线平滑
  const pts = line.points
  ctx.moveTo(pts[0].x, pts[0].y)

  for (let i = 1; i < pts.length - 1; i++) {
    const xc = (pts[i].x + pts[i + 1].x) / 2
    const yc = (pts[i].y + pts[i + 1].y) / 2
    ctx.quadraticCurveTo(pts[i].x, pts[i].y, xc, yc)
  }
  ctx.lineTo(pts[pts.length - 1].x, pts[pts.length - 1].y)

  // 透明度随生命衰减
  const lifeRatio = line.life / line.maxLife
  const alpha = lifeRatio < 0.1
    ? lifeRatio / 0.1
    : lifeRatio > 0.8
      ? (1 - lifeRatio) / 0.2
      : 1

  ctx.globalAlpha = alpha * 0.7
  ctx.lineCap = 'round'
  ctx.lineJoin = 'round'
  ctx.stroke()
  ctx.globalAlpha = 1
}

function drawVillage() {
  if (!ctx) return
  const baseY = height * 0.75
  const energy = getAudioEnergy()

  // 远处的山丘轮廓
  ctx.beginPath()
  ctx.fillStyle = '#0d2b0d'
  ctx.moveTo(0, baseY)
  for (let x = 0; x <= width; x += 2) {
    const y = baseY - 20
      - Math.sin(x * 0.008) * 15
      - Math.sin(x * 0.015 + 1) * 10
      - Math.sin(x * 0.003) * 25
      + Math.sin(time * 0.01 + x * 0.01) * 2 * (1 + energy.mid)
    ctx.lineTo(x, y)
  }
  ctx.lineTo(width, height)
  ctx.lineTo(0, height)
  ctx.closePath()
  ctx.fill()

  // 教堂尖塔
  const cx = width * 0.45
  ctx.fillStyle = '#1a1a2e'
  ctx.fillRect(cx - 8, baseY - 60, 16, 60)
  ctx.beginPath()
  ctx.moveTo(cx - 12, baseY - 60)
  ctx.lineTo(cx, baseY - 95)
  ctx.lineTo(cx + 12, baseY - 60)
  ctx.closePath()
  ctx.fill()

  // 小房子
  for (let i = 0; i < 15; i++) {
    const hx = width * (0.05 + i * 0.06) + Math.sin(i * 2.5) * 20
    const hy = baseY - 15 - Math.sin(hx * 0.01) * 10
    const hw = 18 + Math.random() * 12
    const hh = 14 + Math.random() * 10

    ctx.fillStyle = i % 2 === 0 ? '#16213e' : '#1a1a2e'
    ctx.fillRect(hx - hw / 2, hy - hh, hw, hh)

    // 屋顶
    ctx.beginPath()
    ctx.moveTo(hx - hw / 2 - 3, hy - hh)
    ctx.lineTo(hx, hy - hh - 12)
    ctx.lineTo(hx + hw / 2 + 3, hy - hh)
    ctx.closePath()
    ctx.fill()

    // 窗户（微弱灯光）
    ctx.fillStyle = `rgba(255, 238, 88, ${0.3 + Math.sin(time * 0.05 + i) * 0.15})`
    ctx.fillRect(hx - 3, hy - hh + 4, 6, 5)
  }

  // 大柏树（前景右侧）
  drawCypressTree(width * 0.88, baseY + 10, energy)
}

function drawCypressTree(x: number, baseY: number, energy: { avg: number }) {
  if (!ctx) return
  const treeHeight = 180 + energy.avg * 30

  ctx.fillStyle = '#0d3b0d'
  ctx.beginPath()
  ctx.moveTo(x, baseY - treeHeight)

  // 左侧轮廓（波浪形）
  for (let i = 0; i <= 20; i++) {
    const t = i / 20
    const y = baseY - treeHeight * (1 - t)
    const wobble = Math.sin(t * 8 + time * 0.03) * 5 * (1 + energy.avg)
    const xOff = 15 + t * 25 + wobble
    ctx.lineTo(x - xOff, y)
  }

  ctx.lineTo(x - 5, baseY + 10)

  // 右侧轮廓
  ctx.lineTo(x + 5, baseY + 10)
  for (let i = 20; i >= 0; i--) {
    const t = i / 20
    const y = baseY - treeHeight * (1 - t)
    const wobble = Math.sin(t * 8 + time * 0.03 + 1) * 5 * (1 + energy.avg)
    const xOff = 15 + t * 25 + wobble
    ctx.lineTo(x + xOff, y)
  }

  ctx.closePath()
  ctx.fill()
}

function animate() {
  if (!ctx) return
  time++

  // 半透明覆盖，产生拖尾效果
  drawBackground()
  drawSwirls()
  drawStars()
  drawMoon()
  drawVillage()

  // 更新和绘制流动线条
  const energy = getAudioEnergy()

  for (let i = lines.length - 1; i >= 0; i--) {
    const line = lines[i]
    line.life++

    if (line.life < line.maxLife) {
      advanceLine(line, line.life)
      drawLine(line)
    } else {
      // 替换死掉的线条
      lines[i] = createLine(pickLayer())
    }
  }

  // 根据音频能量添加额外线条
  if (energy.avg > 0.3 && lines.length < 200) {
    const extra = Math.floor(energy.avg * 5)
    for (let i = 0; i < extra; i++) {
      lines.push(createLine(pickLayer()))
    }
  }

  animId = requestAnimationFrame(animate)
}

// ==================== Resize ====================
function resize() {
  const canvas = canvasRef.value
  if (!canvas) return
  width = window.innerWidth
  height = window.innerHeight
  canvas.width = width
  canvas.height = height
  initScene()
}

// ==================== Lifecycle ====================
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
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body {
  overflow: hidden;
  background: #0a0e2a;
}

.app {
  position: relative;
  width: 100vw;
  height: 100vh;
}

canvas {
  display: block;
  width: 100%;
  height: 100%;
}

.controls {
  position: fixed;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 10;
  display: flex;
  align-items: center;
  gap: 12px;
}

.btn {
  padding: 8px 18px;
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 20px;
  color: #e3f2fd;
  font-size: 14px;
  cursor: pointer;
  backdrop-filter: blur(10px);
  transition: all 0.3s;
  user-select: none;
}

.btn:hover {
  background: rgba(255, 255, 255, 0.2);
  border-color: rgba(255, 255, 255, 0.4);
}

.upload-btn {
  display: inline-flex;
  align-items: center;
}

.hint {
  color: rgba(255, 255, 255, 0.4);
  font-size: 12px;
}

.info {
  position: fixed;
  bottom: 16px;
  left: 50%;
  transform: translateX(-50%);
  color: rgba(255, 255, 255, 0.3);
  font-size: 12px;
  z-index: 10;
}
</style>
