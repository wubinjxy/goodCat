<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref } from 'vue'
import { FaceLandmarker, FilesetResolver, HandLandmarker } from '@mediapipe/tasks-vision'

const PHASE = Object.freeze({
  intro: 'intro',
  playing: 'playing',
  end: 'end',
})

const phase = ref(PHASE.intro)
const errorText = ref('')
const loadingText = ref('')
const needsUserGesture = ref(true)
const secureHint = ref('')

const settings = reactive({
  rounds: 10,
  dwellMs: 700, // 脸移到左/右区域的“停留选中”时间
  showHint: true, // 察言观色：闪一下正确侧
  record: false,
  touchMs: 140, // 鼻尖/手指触碰答案的停留时间（防误触，越小越灵敏）
  handTouch: true, // 手触碰选中
  skin: 'cat', // 'cat' | 'dog' | 'none'
})

const videoEl = ref(null)
const canvasEl = ref(null)
const leftChoiceEl = ref(null)
const rightChoiceEl = ref(null)
const ctx = computed(() => canvasEl.value?.getContext('2d') ?? null)

let stream = null
let rafId = null
let landmarker = null
let handLandmarker = null
let lastTs = 0

const ITEMS = [
  { id: 'carrot', name: '萝卜', emoji: '🥕' },
  { id: 'tissue', name: '纸巾', emoji: '🧻' },
  { id: 'can', name: '罐头', emoji: '🥫' },
  { id: 'mickey', name: '米老鼠', emoji: '🐭' },
  { id: 'fish', name: '小鱼干', emoji: '🐟' },
  { id: 'chicken', name: '鸡腿', emoji: '🍗' },
]

const COMMANDS = ['选「{x}」', '把「{x}」给我', '去拿「{x}」', '挑「{x}」']

function randInt(n) {
  return Math.floor(Math.random() * n)
}
function pickOne(arr) {
  return arr[randInt(arr.length)]
}

const round = ref(0)
const score = ref(0)
const eaten = ref(0)
const awaitingEat = ref(false) // 答对后必须吃到奖励才进入下一题
const toast = ref(null) // {title, detail, type}
let toastTimer = null

function showToast(t) {
  if (toastTimer) clearTimeout(toastTimer)
  toast.value = t
  toastTimer = setTimeout(() => (toast.value = null), 1100)
}

// 两个选项：左/右
const question = ref(null) // { targetSide:'left'|'right', leftItem, rightItem, command } // left/right 为“屏幕左右”
const lock = ref(false)
const hintSide = ref(null)
let hintTimer = null

function makeQuestion() {
  hintSide.value = null
  if (hintTimer) clearTimeout(hintTimer)
  // 新题开始：清空计时，但不自动 re-arm
  // （必须先回到中间/离开卡片一次，才允许下一次触发，防止“停在选区里自动连答”）
  zoneStart = null
  touchHold.nose = null
  touchHold.hand = null
  const leftItem = pickOne(ITEMS)
  let rightItem = pickOne(ITEMS)
  while (rightItem.id === leftItem.id) rightItem = pickOne(ITEMS)
  const targetSide = Math.random() < 0.5 ? 'left' : 'right'
  const targetItem = targetSide === 'left' ? leftItem : rightItem
  const command = pickOne(COMMANDS).replace('{x}', targetItem.name)
  question.value = { targetSide, leftItem, rightItem, command }
  if (settings.showHint) {
    hintTimer = setTimeout(() => {
      hintSide.value = targetSide
      setTimeout(() => (hintSide.value = null), 700)
    }, 900)
  }
}

function startGame() {
  phase.value = PHASE.playing
  round.value = 0
  score.value = 0
  eaten.value = 0
  awaitingEat.value = false
  treats.splice(0, treats.length)
  // 新一局：重置所有“触发状态”，保证手/头立即可用
  zoneStart = null
  faceArmed = true
  touchHold.nose = null
  touchHold.hand = null
  touchArmed.nose = true
  touchArmed.hand = true
  handPt = null
  makeQuestion()
  showToast({ type: 'info', title: '开始', detail: '把脸移到左/右区域来选' })
}

function endGame() {
  phase.value = PHASE.end
}

function answer(side) {
  if (!question.value || lock.value) return
  if (awaitingEat.value) {
    showToast({ type: 'info', title: '先吃冻干', detail: '张嘴把掉下来的冻干吃掉，才会下一题' })
    return
  }
  lock.value = true
  // 防止“持续停留导致连发”：答题瞬间先全部锁住，必须离开选区才会重新 armed
  zoneStart = null
  faceArmed = false
  touchHold.nose = null
  touchHold.hand = null
  touchArmed.nose = false
  touchArmed.hand = false

  const ok = side === question.value.targetSide
  if (ok) {
    awaitingEat.value = true
    showToast({ type: 'good', title: '真棒', detail: '掉 1 个冻干，张嘴吃到才算过关！' })
    spawnOneTreat()
  } else {
    showToast({ type: 'bad', title: '再想想', detail: '（答对才会下一题）' })
  }

  setTimeout(() => {
    lock.value = false
    // 答错：保持同一题；答对：等待吃到奖励后再进入下一题
  }, 520)
}

// ====== 录制导出（可选） ======
const isRecording = ref(false)
let recorder = null
let recordedChunks = []

function startRecording() {
  if (!stream) return
  recordedChunks = []
  const mime = MediaRecorder.isTypeSupported('video/webm;codecs=vp9')
    ? 'video/webm;codecs=vp9'
    : 'video/webm'
  recorder = new MediaRecorder(stream, { mimeType: mime })
  recorder.ondataavailable = (e) => {
    if (e.data && e.data.size > 0) recordedChunks.push(e.data)
  }
  recorder.onstop = () => {
    const blob = new Blob(recordedChunks, { type: 'video/webm' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    a.download = `猫猫选择题-${Date.now()}.webm`
    a.click()
    setTimeout(() => URL.revokeObjectURL(url), 5000)
  }
  recorder.start(200)
  isRecording.value = true
  showToast({ type: 'info', title: '录制中', detail: '结束后会自动下载 webm' })
}

function stopRecording() {
  if (!recorder || recorder.state !== 'recording') return
  recorder.stop()
  isRecording.value = false
  showToast({ type: 'info', title: '已停止', detail: '正在生成视频…' })
}

// ====== 叠加动画：猫耳胡须 + 掉落食物 + “张嘴吃到” ======
const treats = [] // 只保留一个奖励
function spawnOneTreat() {
  const w = canvasEl.value?.width ?? 640
  const h = canvasEl.value?.height ?? 480
  treats.splice(0, treats.length)
  treats.push({
    x: w * (0.35 + Math.random() * 0.3),
    y: -30,
    vy: 220,
    emoji: '🐟', // 默认冻干小鱼
    size: Math.max(32, Math.round(h * 0.08)),
    alive: true,
  })
}

function drawText(ctx2d, text, x, y, size = 22, align = 'center') {
  ctx2d.save()
  ctx2d.font = `${size}px system-ui, sans-serif`
  ctx2d.textAlign = align
  ctx2d.textBaseline = 'middle'
  ctx2d.fillStyle = 'rgba(15,23,42,0.9)'
  ctx2d.strokeStyle = 'rgba(255,255,255,0.85)'
  ctx2d.lineWidth = 6
  ctx2d.strokeText(text, x, y)
  ctx2d.fillText(text, x, y)
  ctx2d.restore()
}

function clamp(v, a, b) {
  return Math.max(a, Math.min(b, v))
}

function mouthOpenRatio(lm) {
  // FaceMesh 索引：13/14 上下唇中点，61/291 嘴角
  const up = lm[13]
  const lo = lm[14]
  const l = lm[61]
  const r = lm[291]
  if (!up || !lo || !l || !r) return 0
  const dy = Math.hypot(up.x - lo.x, up.y - lo.y)
  const dx = Math.hypot(l.x - r.x, l.y - r.y)
  return dx > 0 ? dy / dx : 0
}

function drawCatOverlay(ctx2d, lm, w, h) {
  // 基于眼睛/鼻子估算位置
  const leftEye = lm[33]
  const rightEye = lm[263]
  const nose = lm[1]
  if (!leftEye || !rightEye || !nose) return

  const cx = ((leftEye.x + rightEye.x) / 2) * w
  const cy = ((leftEye.y + rightEye.y) / 2) * h
  const nx = nose.x * w
  const ny = nose.y * h
  const faceW = Math.hypot((leftEye.x - rightEye.x) * w, (leftEye.y - rightEye.y) * h) * 2.1

  // 猫耳（两个三角）
  const earY = cy - faceW * 0.6
  const earXOffset = faceW * 0.38
  const earW = faceW * 0.28
  const earH = faceW * 0.26

  ctx2d.save()
  // 更明显的猫猫特效：外耳+内耳+腮红+猫鼻+小嘴
  ctx2d.fillStyle = 'rgba(15,23,42,0.86)'
  ctx2d.strokeStyle = 'rgba(255,255,255,0.9)'
  ctx2d.lineWidth = 5

  function ear(x) {
    ctx2d.beginPath()
    ctx2d.moveTo(x, earY + earH)
    ctx2d.lineTo(x + earW / 2, earY)
    ctx2d.lineTo(x + earW, earY + earH)
    ctx2d.closePath()
    ctx2d.stroke()
    ctx2d.fill()

    // 内耳（粉色）
    ctx2d.save()
    ctx2d.fillStyle = 'rgba(244,114,182,0.75)'
    ctx2d.strokeStyle = 'rgba(255,255,255,0.65)'
    ctx2d.lineWidth = 3
    ctx2d.beginPath()
    ctx2d.moveTo(x + earW * 0.18, earY + earH * 0.92)
    ctx2d.lineTo(x + earW * 0.5, earY + earH * 0.18)
    ctx2d.lineTo(x + earW * 0.82, earY + earH * 0.92)
    ctx2d.closePath()
    ctx2d.stroke()
    ctx2d.fill()
    ctx2d.restore()
  }

  ear(cx - earXOffset - earW / 2)
  ear(cx + earXOffset - earW / 2)

  // 胡须
  const whiskY = ny + faceW * 0.06
  const whiskLen = faceW * 0.32
  const whiskGap = faceW * 0.08
  ctx2d.lineWidth = 4
  ctx2d.strokeStyle = 'rgba(15,23,42,0.78)'

  function whiskerRow(dir) {
    for (let i = -1; i <= 1; i++) {
      const y = whiskY + i * whiskGap
      ctx2d.beginPath()
      ctx2d.moveTo(nx, y)
      ctx2d.lineTo(nx + dir * whiskLen, y - i * whiskGap * 0.25)
      ctx2d.stroke()
    }
  }

  whiskerRow(-1)
  whiskerRow(1)

  // 腮红
  ctx2d.save()
  ctx2d.fillStyle = 'rgba(244,114,182,0.35)'
  ctx2d.beginPath()
  ctx2d.ellipse(nx - faceW * 0.22, ny + faceW * 0.14, faceW * 0.08, faceW * 0.055, 0, 0, Math.PI * 2)
  ctx2d.fill()
  ctx2d.beginPath()
  ctx2d.ellipse(nx + faceW * 0.22, ny + faceW * 0.14, faceW * 0.08, faceW * 0.055, 0, 0, Math.PI * 2)
  ctx2d.fill()
  ctx2d.restore()

  // 猫鼻（小三角）
  ctx2d.save()
  ctx2d.fillStyle = 'rgba(244,114,182,0.95)'
  ctx2d.strokeStyle = 'rgba(255,255,255,0.85)'
  ctx2d.lineWidth = 3
  ctx2d.beginPath()
  ctx2d.moveTo(nx, ny + faceW * 0.02)
  ctx2d.lineTo(nx - faceW * 0.045, ny + faceW * 0.055)
  ctx2d.lineTo(nx, ny + faceW * 0.082)
  ctx2d.lineTo(nx + faceW * 0.045, ny + faceW * 0.055)
  ctx2d.closePath()
  ctx2d.stroke()
  ctx2d.fill()
  ctx2d.restore()

  // 小嘴（两条弧线）
  ctx2d.save()
  ctx2d.strokeStyle = 'rgba(15,23,42,0.72)'
  ctx2d.lineWidth = 3
  const mouthY = ny + faceW * 0.12
  const mouthW = faceW * 0.06
  ctx2d.beginPath()
  ctx2d.arc(nx - mouthW, mouthY, mouthW, 0, Math.PI * 0.9)
  ctx2d.stroke()
  ctx2d.beginPath()
  ctx2d.arc(nx + mouthW, mouthY, mouthW, 0, Math.PI * 0.9)
  ctx2d.stroke()
  ctx2d.restore()

  ctx2d.restore()
}

// “脸移到左/右区域”选择：根据人脸中心 x 的停留时间判定
let zoneStart = null // {side, at}
let faceArmed = true // 需要先离开中间区域，才允许再次触发

function updateSelectionByFace(faceCenterX, now) {
  if (awaitingEat.value) return
  // faceCenterX 为“屏幕坐标”(已做镜像)，0~1
  const side = faceCenterX < 0.4 ? 'left' : faceCenterX > 0.6 ? 'right' : null
  if (!side) {
    zoneStart = null
    faceArmed = true
    return
  }
  if (!faceArmed) return
  if (!zoneStart || zoneStart.side !== side) zoneStart = { side, at: now }
  if (now - zoneStart.at >= settings.dwellMs) {
    zoneStart = null
    faceArmed = false
    answer(side)
  }
}

// ========== 触碰答案选中（鼻尖/手指） ==========
let choiceRectsNorm = null // { left:{x0,y0,x1,y1}, right:{...} } in [0..1]
let lastRectUpdate = 0
let touchHold = { nose: null, hand: null } // { side, at:number }
let touchArmed = { nose: true, hand: true } // 需要先离开卡片区域，才允许再次触发
let handPt = null // {x,y} smoothed in [0..1]

function rectFromDom(el, canvasRect) {
  const r = el.getBoundingClientRect()
  const x0 = (r.left - canvasRect.left) / canvasRect.width
  const y0 = (r.top - canvasRect.top) / canvasRect.height
  const x1 = (r.right - canvasRect.left) / canvasRect.width
  const y1 = (r.bottom - canvasRect.top) / canvasRect.height
  return { x0: clamp(x0, 0, 1), y0: clamp(y0, 0, 1), x1: clamp(x1, 0, 1), y1: clamp(y1, 0, 1) }
}

function updateChoiceRects(nowMs) {
  const c = canvasEl.value
  if (!c) return
  const canvasRect = c.getBoundingClientRect()
  if (!canvasRect.width || !canvasRect.height) return
  const leftEl = leftChoiceEl.value
  const rightEl = rightChoiceEl.value
  if (!leftEl || !rightEl) return
  choiceRectsNorm = {
    left: rectFromDom(leftEl, canvasRect),
    right: rectFromDom(rightEl, canvasRect),
  }
  lastRectUpdate = nowMs
}

function pointInRect(p, r) {
  return p.x >= r.x0 && p.x <= r.x1 && p.y >= r.y0 && p.y <= r.y1
}

function hitTestChoice(p, nowMs, who) {
  if (awaitingEat.value) return
  if (phase.value !== PHASE.playing || !question.value || lock.value) return
  if (!choiceRectsNorm) return
  const side = pointInRect(p, choiceRectsNorm.left) ? 'left' : pointInRect(p, choiceRectsNorm.right) ? 'right' : null
  if (!side) {
    touchHold[who] = null
    touchArmed[who] = true
    return
  }
  if (!touchArmed[who]) return
  if (!touchHold[who] || touchHold[who].side !== side) touchHold[who] = { side, at: nowMs }
  // 手部通常抖动更大，给它更短阈值 + 少量“深度进入”加速
  const base = who === 'hand' ? Math.max(80, settings.touchMs - 40) : settings.touchMs
  const r = choiceRectsNorm[side]
  const cx = (r.x0 + r.x1) / 2
  const cy = (r.y0 + r.y1) / 2
  const dx = Math.abs(p.x - cx) / Math.max(1e-6, (r.x1 - r.x0) / 2)
  const dy = Math.abs(p.y - cy) / Math.max(1e-6, (r.y1 - r.y0) / 2)
  const deep = dx < 0.55 && dy < 0.55 // 更靠近中心则加速触发
  const threshold = deep ? Math.max(60, base - 50) : base
  if (nowMs - touchHold[who].at >= threshold) {
    touchHold.nose = null
    touchHold.hand = null
    touchArmed.nose = false
    touchArmed.hand = false
    answer(side)
  }
}

function drawDogOverlay(ctx2d, lm, w, h) {
  // 基于眼睛/鼻子估算位置
  const leftEye = lm[33]
  const rightEye = lm[263]
  const nose = lm[1]
  if (!leftEye || !rightEye || !nose) return

  const cx = ((leftEye.x + rightEye.x) / 2) * w
  const cy = ((leftEye.y + rightEye.y) / 2) * h
  const nx = nose.x * w
  const ny = nose.y * h
  const faceW = Math.hypot((leftEye.x - rightEye.x) * w, (leftEye.y - rightEye.y) * h) * 2.15

  const earY = cy - faceW * 0.52
  const earXOffset = faceW * 0.42
  const earW = faceW * 0.34
  const earH = faceW * 0.34

  ctx2d.save()
  ctx2d.fillStyle = 'rgba(120,72,32,0.82)' // 棕色狗耳
  ctx2d.strokeStyle = 'rgba(255,255,255,0.88)'
  ctx2d.lineWidth = 5

  function floppyEar(x, flip) {
    // 垂耳：圆角矩形 + 小弧
    ctx2d.beginPath()
    roundRect(ctx2d, x - earW / 2, earY, earW, earH, earW * 0.45)
    ctx2d.stroke()
    ctx2d.fill()

    // 耳朵内侧
    ctx2d.save()
    ctx2d.fillStyle = 'rgba(244,114,182,0.35)'
    ctx2d.beginPath()
    roundRect(ctx2d, x - earW * 0.34, earY + earH * 0.18, earW * 0.68, earH * 0.68, earW * 0.35)
    ctx2d.fill()
    ctx2d.restore()

    // 轻微下垂弧线
    ctx2d.save()
    ctx2d.strokeStyle = 'rgba(15,23,42,0.18)'
    ctx2d.lineWidth = 3
    ctx2d.beginPath()
    ctx2d.arc(x + flip * earW * 0.08, earY + earH * 0.98, earW * 0.3, Math.PI, Math.PI * 1.85)
    ctx2d.stroke()
    ctx2d.restore()
  }

  floppyEar(cx - earXOffset, -1)
  floppyEar(cx + earXOffset, 1)

  // 狗鼻（更大更黑）
  ctx2d.save()
  ctx2d.fillStyle = 'rgba(15,23,42,0.82)'
  ctx2d.strokeStyle = 'rgba(255,255,255,0.78)'
  ctx2d.lineWidth = 3
  ctx2d.beginPath()
  ctx2d.ellipse(nx, ny + faceW * 0.04, faceW * 0.06, faceW * 0.045, 0, 0, Math.PI * 2)
  ctx2d.stroke()
  ctx2d.fill()
  ctx2d.restore()

  // 舌头（粉色小椭圆）
  ctx2d.save()
  ctx2d.fillStyle = 'rgba(244,114,182,0.75)'
  ctx2d.strokeStyle = 'rgba(255,255,255,0.6)'
  ctx2d.lineWidth = 3
  ctx2d.beginPath()
  ctx2d.ellipse(nx, ny + faceW * 0.15, faceW * 0.045, faceW * 0.06, 0, 0, Math.PI * 2)
  ctx2d.stroke()
  ctx2d.fill()
  ctx2d.restore()

  // 腮红（更淡）
  ctx2d.save()
  ctx2d.fillStyle = 'rgba(244,114,182,0.22)'
  ctx2d.beginPath()
  ctx2d.ellipse(nx - faceW * 0.22, ny + faceW * 0.12, faceW * 0.085, faceW * 0.06, 0, 0, Math.PI * 2)
  ctx2d.fill()
  ctx2d.beginPath()
  ctx2d.ellipse(nx + faceW * 0.22, ny + faceW * 0.12, faceW * 0.085, faceW * 0.06, 0, 0, Math.PI * 2)
  ctx2d.fill()
  ctx2d.restore()

  ctx2d.restore()
}

function cycleSkin() {
  const order = ['cat', 'dog', 'none']
  const idx = order.indexOf(settings.skin)
  settings.skin = order[(idx + 1 + order.length) % order.length]
  const name = settings.skin === 'cat' ? '小猫' : settings.skin === 'dog' ? '小狗' : '关闭特效'
  showToast({ type: 'info', title: '换装', detail: `已切换：${name}` })
}

function step(nowMs) {
  rafId = requestAnimationFrame(step)
  const v = videoEl.value
  const c = canvasEl.value
  const ctx2d = ctx.value
  if (!v || !c || !ctx2d) return
  if (v.readyState < 2) return

  const w = v.videoWidth || 640
  const h = v.videoHeight || 480
  if (c.width !== w || c.height !== h) {
    c.width = w
    c.height = h
  }

  // 只画特效/叠加层：视频由 <video> 自己显示，避免 canvas 重绘视频导致的重影/抖动
  ctx2d.clearRect(0, 0, w, h)

  let mouthRect = null
  if (landmarker) {
    const res = landmarker.detectForVideo(v, nowMs)
    const lms = res?.faceLandmarks?.[0]
    if (lms && lms.length) {
      // 人脸中心（用鼻尖近似）
      const nose = lms[1]
      // 因为画面镜像展示，所以把 landmark 的 x 反转成“屏幕坐标”
      if (nose) {
        const nxNorm = 1 - nose.x
        const nyNorm = nose.y
        updateSelectionByFace(nxNorm, nowMs)

        if (phase.value === PHASE.playing) {
          if (!choiceRectsNorm || nowMs - lastRectUpdate > 200) updateChoiceRects(nowMs)
          hitTestChoice({ x: nxNorm, y: nyNorm }, nowMs, 'nose')
        }
      }

      // 张嘴“吃到”判定区域：用嘴中心附近
      const mouthCenter = lms[13] && lms[14] ? { x: (lms[13].x + lms[14].x) / 2, y: (lms[13].y + lms[14].y) / 2 } : null
      const leftM = lms[61]
      const rightM = lms[291]
      if (mouthCenter && leftM && rightM) {
        const mouthW = Math.hypot((leftM.x - rightM.x) * w, (leftM.y - rightM.y) * h)
        mouthRect = {
          // 这里使用“原始视频坐标”，镜像由 CSS 统一处理
          x: mouthCenter.x * w,
          y: mouthCenter.y * h,
          r: mouthW * 0.35,
          open: mouthOpenRatio(lms) > 0.32,
        }
      }

      if (settings.skin === 'cat') drawCatOverlay(ctx2d, lms, w, h)
      else if (settings.skin === 'dog') drawDogOverlay(ctx2d, lms, w, h)
    }
  }

  // 手部：食指指尖触碰答案选中
  if (settings.handTouch && handLandmarker && phase.value === PHASE.playing) {
    const resH = handLandmarker.detectForVideo(v, nowMs)
    const hLm = resH?.landmarks?.[0]
    if (hLm && hLm.length >= 9) {
      const idx = hLm[8] // 食指指尖
      const hx = 1 - idx.x
      const hy = idx.y
      // 平滑一下手指点，减少抖动导致的“计时重置”，触发更灵敏
      if (!handPt) handPt = { x: hx, y: hy }
      const a = 0.62
      handPt.x = handPt.x * (1 - a) + hx * a
      handPt.y = handPt.y * (1 - a) + hy * a

      if (!choiceRectsNorm || nowMs - lastRectUpdate > 120) updateChoiceRects(nowMs)
      hitTestChoice({ x: handPt.x, y: handPt.y }, nowMs, 'hand')
    } else {
      // 手没被识别到时，视为“已离开卡片”，否则会卡在未 armed 状态导致手永远选不中
      touchHold.hand = null
      touchArmed.hand = true
      handPt = null
    }
  } else if (phase.value === PHASE.playing) {
    // 手功能关闭/模型未就绪时，同样不要让 hand 触发状态卡死
    touchHold.hand = null
    touchArmed.hand = true
    handPt = null
  }

  // 掉落食物更新与绘制 + 张嘴吃到
  const dt = Math.min(0.05, Math.max(0.0, (nowMs - lastTs) / 1000))
  lastTs = nowMs

  if (treats.length) {
    for (const t of treats) {
      if (!t.alive) continue
      t.y += t.vy * dt
      // “吃到”：需要张嘴且距离嘴巴圆形范围内
      if (mouthRect?.open) {
        const dx = t.x - mouthRect.x
        const dy = t.y - mouthRect.y
        if (dx * dx + dy * dy < mouthRect.r * mouthRect.r) {
          t.alive = false
          eaten.value += 1
          showToast({ type: 'good', title: '吃到啦', detail: '过关！下一题～' })

          if (awaitingEat.value) {
            awaitingEat.value = false
            score.value += 1
            round.value += 1

            // 清理奖励
            treats.splice(0, treats.length)

            // 进入下一题/结算
            if (round.value >= settings.rounds) endGame()
            else makeQuestion()
          }
        }
      }
      // 奖励没吃到不能过关：掉下去就重置回顶部，直到吃到为止
      if (t.y > h + 60) {
        if (awaitingEat.value) {
          t.y = -30
          t.x = w * (0.35 + Math.random() * 0.3)
        } else {
          t.alive = false
        }
      }
    }
    // 清理
    for (let i = treats.length - 1; i >= 0; i--) if (!treats[i].alive) treats.splice(i, 1)
  }

  if (treats.length) {
    for (const t of treats) {
      drawText(ctx2d, t.emoji, t.x, t.y, t.size, 'center')
    }
  }

  // no-op
}

function roundRect(ctx2d, x, y, w, h, r) {
  const rr = Math.min(r, w / 2, h / 2)
  ctx2d.beginPath()
  ctx2d.moveTo(x + rr, y)
  ctx2d.arcTo(x + w, y, x + w, y + h, rr)
  ctx2d.arcTo(x + w, y + h, x, y + h, rr)
  ctx2d.arcTo(x, y + h, x, y, rr)
  ctx2d.arcTo(x, y, x + w, y, rr)
  ctx2d.closePath()
}

async function setupCamera() {
  errorText.value = ''
  const v = videoEl.value
  if (!v) return
  const getMedia = navigator.mediaDevices.getUserMedia({
    video: { width: { ideal: 1280 }, height: { ideal: 720 }, facingMode: 'user' },
    audio: false,
  })
  // 某些 WebView 会“卡住不弹窗”，加超时提示
  const timeout = new Promise((_, rej) =>
    setTimeout(() => rej(new Error('getUserMedia_timeout')), 9000),
  )
  stream = await Promise.race([getMedia, timeout])
  v.srcObject = stream
  await v.play()
}

async function setupLandmarker() {
  loadingText.value = '正在加载人脸模型…'
  const vision = await FilesetResolver.forVisionTasks(
    'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.21/wasm',
  )
  landmarker = await FaceLandmarker.createFromOptions(vision, {
    baseOptions: {
      modelAssetPath:
        'https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/1/face_landmarker.task',
    },
    runningMode: 'VIDEO',
    numFaces: 1,
  })

  loadingText.value = '正在加载手部模型…'
  handLandmarker = await HandLandmarker.createFromOptions(vision, {
    baseOptions: {
      modelAssetPath:
        'https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task',
    },
    runningMode: 'VIDEO',
    numHands: 1,
  })
}

async function boot() {
  try {
    errorText.value = ''
    loadingText.value = '正在请求摄像头权限…（如果没弹窗，请用浏览器打开并允许权限）'
    await setupCamera()
    await setupLandmarker()
    loadingText.value = ''
    needsUserGesture.value = false
    lastTs = performance.now()
    rafId = requestAnimationFrame(step)
  } catch (e) {
    console.error(e)
    loadingText.value = ''
    const msg = String(e?.message || e)
    if (msg.includes('getUserMedia_timeout')) {
      errorText.value =
        '摄像头权限没有弹出/被拦截。请用系统浏览器（Chrome/Edge）打开，并手动允许摄像头权限。'
      return
    }
    if (!window.isSecureContext && location.hostname !== 'localhost') {
      errorText.value =
        '当前不是 HTTPS（安全环境），很多手机会直接禁用摄像头。请用 https://你的IP:5173 打开（首次会提示不安全，选择继续）。'
      return
    }
    errorText.value = `摄像头/模型初始化失败：${msg}`
  }
}

function cleanup() {
  if (rafId) cancelAnimationFrame(rafId)
  rafId = null
  if (hintTimer) clearTimeout(hintTimer)
  hintTimer = null
  if (toastTimer) clearTimeout(toastTimer)
  toastTimer = null
  if (recorder && recorder.state === 'recording') recorder.stop()
  recorder = null
  if (stream) {
    for (const t of stream.getTracks()) t.stop()
    stream = null
  }
  landmarker?.close?.()
  landmarker = null
  handLandmarker?.close?.()
  handLandmarker = null
}

onMounted(() => {
  // 移动端/内置浏览器常拦截“自动申请摄像头”，改为必须点击按钮触发
  if (!window.isSecureContext && location.hostname !== 'localhost') {
    secureHint.value =
      '提示：你现在是 http 打开。手机通常需要 https 才能启用摄像头。请改用 https://你的IP:5173'
  } else {
    secureHint.value = ''
  }
})
onBeforeUnmount(() => {
  cleanup()
})

const progressText = computed(() => `${round.value}/${settings.rounds}`)

const leftOption = computed(() => question.value?.leftItem ?? null)
const rightOption = computed(() => question.value?.rightItem ?? null)
</script>

<template>
  <div class="wrap" :data-phase="phase">
    <header class="top" v-show="phase !== PHASE.playing">
      <div class="title">自拍视频小游戏：萝卜还是纸巾？</div>
      <div class="sub">玩法：把脸移到左/右选区来选，选对就「真棒」；天上会掉吃的，张嘴还能吃到。</div>
    </header>

    <div class="stage" :class="{ fullscreen: phase === PHASE.playing }">
      <video ref="videoEl" class="video" playsinline muted></video>
      <canvas ref="canvasEl" class="canvas" />

      <div v-if="needsUserGesture && !errorText" class="overlay">
        <div class="box">
          <div class="boxTitle">点击启用摄像头</div>
          <div class="boxDesc">
            {{ secureHint || '首次需要你手动点一下，浏览器才会弹出摄像头权限。' }}
          </div>
          <div class="boxActions">
            <button class="btn primary" type="button" @click="boot">启用摄像头</button>
          </div>
          <div v-if="loadingText" class="boxDesc" style="margin-top: 8px">{{ loadingText }}</div>
        </div>
      </div>
      <div v-else-if="errorText" class="overlay">
        <div class="box err">{{ errorText }}</div>
      </div>

      <!-- 游戏态：把选项直接贴在视频上（更直观），同时保留“脸移到左右”交互 -->
      <div v-if="phase === PHASE.playing && question" class="gameOverlay">
        <div class="topBar">
          <div class="cmd">我说：{{ question.command }}</div>
          <div class="mini">
            <span>进度</span><b>{{ progressText }}</b>
            <span>得分</span><b>{{ score }}</b>
            <span>已吃</span><b>{{ eaten }}</b>
          </div>
            <div v-if="awaitingEat" class="waitEat">奖励已掉落：张嘴吃到 1 个冻干才能下一题</div>
            <div class="topBtns">
              <button class="iconBtn" type="button" @click="cycleSkin">换装</button>
              <button class="iconBtn" type="button" @click="phase = PHASE.end">结算</button>
            </div>
        </div>

        <div class="choices">
          <button
            class="choice"
            ref="leftChoiceEl"
            type="button"
            :data-hint="hintSide === 'left' ? 'on' : 'off'"
            :disabled="lock"
            @click="answer('left')"
          >
            <div class="cEmoji">{{ leftOption?.emoji }}</div>
            <div class="cName">{{ leftOption?.name }}</div>
            <div class="cTip">把脸移到左边 / 点击</div>
          </button>
          <button
            class="choice"
            ref="rightChoiceEl"
            type="button"
            :data-hint="hintSide === 'right' ? 'on' : 'off'"
            :disabled="lock"
            @click="answer('right')"
          >
            <div class="cEmoji">{{ rightOption?.emoji }}</div>
            <div class="cName">{{ rightOption?.name }}</div>
            <div class="cTip">把脸移到右边 / 点击</div>
          </button>
        </div>
      </div>

      <transition name="toast">
        <div v-if="toast" class="toast" :data-type="toast.type">
          <div class="toastTitle">{{ toast.title }}</div>
          <div class="toastDetail">{{ toast.detail }}</div>
        </div>
      </transition>
    </div>

    <div class="hud" v-show="phase !== PHASE.playing">
      <div class="pill"><span>进度</span><b>{{ progressText }}</b></div>
      <div class="pill"><span>得分</span><b>{{ score }}</b></div>
      <div class="pill"><span>已吃</span><b>{{ eaten }}</b></div>
      <div class="spacer" />
      <label class="pill toggle">
        <span>察言观色</span>
        <input type="checkbox" v-model="settings.showHint" />
      </label>
      <label class="pill toggle">
        <span>录制</span>
        <input type="checkbox" v-model="settings.record" />
      </label>
      <button
        v-if="settings.record && !isRecording"
        class="btn"
        type="button"
        @click="startRecording"
        :disabled="!stream"
      >
        开始录制
      </button>
      <button v-if="settings.record && isRecording" class="btn danger" type="button" @click="stopRecording">
        停止并下载
      </button>
    </div>

    <div class="controls" v-show="phase !== PHASE.playing">
      <template v-if="phase === PHASE.intro">
        <label class="select">
          <span>题量</span>
          <select v-model.number="settings.rounds">
            <option :value="5">5</option>
            <option :value="10">10</option>
            <option :value="15">15</option>
          </select>
        </label>
        <label class="select">
          <span>停留选中</span>
          <select v-model.number="settings.dwellMs">
            <option :value="450">快（0.45s）</option>
            <option :value="700">中（0.7s）</option>
            <option :value="950">慢（0.95s）</option>
          </select>
        </label>
        <label class="select">
          <span>触碰灵敏度</span>
          <select v-model.number="settings.touchMs">
            <option :value="90">很灵敏（0.09s）</option>
            <option :value="140">灵敏（0.14s）</option>
            <option :value="220">稳一点（0.22s）</option>
          </select>
        </label>
        <label class="select">
          <span>手触碰选中</span>
          <select v-model="settings.handTouch">
            <option :value="true">开</option>
            <option :value="false">关</option>
          </select>
        </label>
        <label class="select">
          <span>换装特效</span>
          <select v-model="settings.skin">
            <option value="cat">小猫</option>
            <option value="dog">小狗</option>
            <option value="none">关闭</option>
          </select>
        </label>
        <button class="btn primary" type="button" @click="startGame" :disabled="!!loadingText || !!errorText">
          开始考试
        </button>
      </template>

      <template v-else-if="phase === PHASE.playing">
        <button class="btn" type="button" @click="phase = PHASE.end">提前结算</button>
      </template>

      <template v-else>
        <div class="result">
          <div class="rTitle">考试结束</div>
          <div class="rRow">
            <span>得分</span><b>{{ score }}</b>
            <span>吃到</span><b>{{ eaten }}</b>
          </div>
        </div>
        <button class="btn primary" type="button" @click="startGame">再来一局</button>
        <button class="btn" type="button" @click="phase = PHASE.intro">回到设置</button>
      </template>
    </div>
  </div>
</template>

<style scoped>
.wrap {
  width: min(980px, 100%);
  display: grid;
  gap: 12px;
}
.top .title {
  font-size: 22px;
  font-weight: 900;
  letter-spacing: -0.02em;
}
.top .sub {
  margin-top: 4px;
  color: var(--muted);
  font-size: 13px;
}

.stage {
  position: relative;
  width: 100%;
  aspect-ratio: 16 / 9;
  border-radius: 18px;
  overflow: hidden;
  border: 1px solid var(--border);
  background: #0b1222;
  box-shadow: 0 12px 34px rgba(0, 0, 0, 0.12);
}
.stage.fullscreen {
  position: fixed;
  inset: 0;
  z-index: 50;
  width: 100vw;
  height: 100vh;
  aspect-ratio: auto;
  border-radius: 0;
  border: none;
  box-shadow: none;
  background: #000;
}
.video {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  transform: scaleX(-1);
}
.canvas {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  transform: scaleX(-1); /* 叠加层与视频统一镜像，避免坐标反复翻转导致的重影/错位 */
}

.gameOverlay {
  position: absolute;
  inset: 0;
  pointer-events: none;
}
.topBar {
  pointer-events: auto;
  position: absolute;
  left: 12px;
  right: 12px;
  top: 10px;
  display: grid;
  gap: 8px;
}
.cmd {
  padding: 10px 12px;
  border-radius: 14px;
  background: rgba(255, 255, 255, 0.85);
  border: 1px solid rgba(15, 23, 42, 0.16);
  font-weight: 950;
  text-align: center;
}
.mini {
  display: flex;
  gap: 10px;
  align-items: baseline;
  justify-content: center;
  padding: 8px 10px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.72);
  border: 1px solid rgba(15, 23, 42, 0.14);
  color: rgba(15, 23, 42, 0.7);
  font-size: 12px;
  flex-wrap: wrap;
}
.mini b {
  color: rgba(15, 23, 42, 0.92);
  font-variant-numeric: tabular-nums;
}
.waitEat {
  pointer-events: none;
  justify-self: center;
  text-align: center;
  padding: 8px 10px;
  border-radius: 999px;
  background: rgba(255, 214, 10, 0.28);
  border: 1px solid rgba(255, 214, 10, 0.6);
  color: rgba(15, 23, 42, 0.86);
  font-size: 12px;
  font-weight: 850;
}
.iconBtn {
  align-self: center;
  justify-self: center;
  pointer-events: auto;
  border: 1px solid rgba(255, 255, 255, 0.28);
  background: rgba(0, 0, 0, 0.35);
  color: rgba(255, 255, 255, 0.92);
  border-radius: 999px;
  padding: 8px 12px;
  font-weight: 850;
}
.topBtns {
  display: flex;
  gap: 10px;
  justify-content: center;
  align-items: center;
}

.choices {
  pointer-events: auto;
  position: absolute;
  left: 12px;
  right: 12px;
  bottom: 12px;
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px;
}
.choice {
  border: 1px solid rgba(255, 255, 255, 0.28);
  background: rgba(255, 255, 255, 0.82);
  border-radius: 18px;
  padding: 12px 12px 10px;
  text-align: center;
  backdrop-filter: blur(8px);
}
.choice:disabled {
  opacity: 0.9;
}
.choice[data-hint='on'] {
  border-color: rgba(255, 214, 10, 0.8);
  box-shadow: 0 0 0 6px rgba(255, 214, 10, 0.18);
}
.cEmoji {
  font-size: 34px;
  line-height: 1;
}
.cName {
  margin-top: 8px;
  font-weight: 950;
  font-size: 18px;
  letter-spacing: -0.01em;
}
.cTip {
  margin-top: 6px;
  color: rgba(15, 23, 42, 0.62);
  font-size: 12px;
  font-weight: 650;
}

.overlay {
  position: absolute;
  inset: 0;
  display: grid;
  place-items: center;
  background: rgba(0, 0, 0, 0.35);
}
.box {
  padding: 12px 14px;
  border-radius: 14px;
  background: rgba(255, 255, 255, 0.92);
  border: 1px solid rgba(15, 23, 42, 0.14);
  font-weight: 650;
  width: min(520px, calc(100% - 28px));
  text-align: center;
}
.boxTitle {
  font-weight: 950;
}
.boxDesc {
  margin-top: 8px;
  color: rgba(15, 23, 42, 0.72);
  font-size: 13px;
  line-height: 1.45;
  font-weight: 600;
}
.boxActions {
  margin-top: 12px;
  display: flex;
  justify-content: center;
}
.box.err {
  background: rgba(255, 255, 255, 0.96);
}

.hud {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  align-items: center;
}
.pill {
  display: inline-flex;
  gap: 8px;
  align-items: baseline;
  padding: 8px 10px;
  border: 1px solid var(--border);
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.78);
}
.pill span {
  color: var(--muted);
  font-size: 12px;
}
.pill b {
  font-variant-numeric: tabular-nums;
}
.pill.toggle input {
  width: 40px;
  height: 20px;
}
.spacer {
  flex: 1 1 auto;
}

.controls {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
  align-items: center;
}
.select {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 10px 12px;
  border: 1px solid var(--border);
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.78);
}
.select span {
  color: var(--muted);
  font-size: 12px;
}
.select select {
  border: 1px solid rgba(15, 23, 42, 0.18);
  border-radius: 10px;
  padding: 8px 10px;
  background: white;
}

.btn {
  border: 1px solid rgba(15, 23, 42, 0.18);
  background: white;
  border-radius: 12px;
  padding: 10px 14px;
  font-weight: 750;
  cursor: pointer;
  transition: transform 0.08s ease, box-shadow 0.12s ease;
}
.btn:hover {
  box-shadow: 0 10px 22px rgba(0, 0, 0, 0.08);
}
.btn:active {
  transform: translateY(1px);
}
.btn.primary {
  background: linear-gradient(135deg, color-mix(in srgb, var(--accent) 85%, white), var(--accent2));
}
.btn.danger {
  background: #fee2e2;
  border-color: rgba(220, 38, 38, 0.25);
}

.result {
  padding: 10px 12px;
  border: 1px solid var(--border);
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.78);
}
.rTitle {
  font-weight: 900;
}
.rRow {
  margin-top: 6px;
  display: flex;
  gap: 8px;
  align-items: baseline;
  flex-wrap: wrap;
  color: var(--muted);
}
.rRow b {
  color: #0f172a;
  font-variant-numeric: tabular-nums;
}

.toast {
  position: absolute;
  left: 50%;
  bottom: 14px;
  transform: translateX(-50%);
  border-radius: 14px;
  padding: 10px 12px;
  border: 1px solid rgba(15, 23, 42, 0.16);
  background: rgba(255, 255, 255, 0.9);
  min-width: min(420px, calc(100% - 24px));
}
.toastTitle {
  font-weight: 900;
}
.toastDetail {
  margin-top: 4px;
  color: var(--muted);
  font-size: 13px;
}
.toast[data-type='good'] {
  border-color: rgba(22, 163, 74, 0.35);
}
.toast[data-type='bad'] {
  border-color: rgba(220, 38, 38, 0.35);
}

.toast-enter-active,
.toast-leave-active {
  transition: opacity 160ms ease, transform 160ms ease;
}
.toast-enter-from,
.toast-leave-to {
  opacity: 0;
  transform: translateX(-50%) translateY(8px);
}
</style>

