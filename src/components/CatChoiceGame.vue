<script setup>
import { computed, onBeforeUnmount, reactive, ref, watch } from 'vue'

const PHASE = Object.freeze({
  intro: 'intro',
  playing: 'playing',
  end: 'end',
})

const settings = reactive({
  rounds: 10,
  optionsPerRound: 3,
  hintMode: true, // 察言观色：延迟提示正确答案
  sound: false, // 默认关闭，避免浏览器权限/打断
})

const phase = ref(PHASE.intro)
const round = ref(0)
const score = ref(0)
const streak = ref(0)
const bestStreak = ref(0)
const unlockedCount = ref(2)

const toast = ref(null) // { type:'good'|'bad'|'info', title, detail }
const selectedId = ref(null)
const locked = ref(false)
const hintId = ref(null)

let hintTimer = null
let toastTimer = null

const ITEMS = [
  { id: 'carrot', name: '萝卜', emoji: '🥕' },
  { id: 'tissue', name: '纸巾', emoji: '🧻' },
  { id: 'can', name: '罐头', emoji: '🥫' },
  { id: 'mickey', name: '米老鼠', emoji: '🐭' },
  { id: 'fish', name: '小鱼干', emoji: '🐟' },
  { id: 'ball', name: '小球', emoji: '⚽' },
  { id: 'key', name: '钥匙', emoji: '🗝️' },
  { id: 'sock', name: '袜子', emoji: '🧦' },
  { id: 'cucumber', name: '黄瓜', emoji: '🥒' },
]

const COMMANDS = [
  '把「{x}」给我',
  '选「{x}」',
  '去拿「{x}」',
  '把「{x}」叼过来',
]

function randInt(n) {
  return Math.floor(Math.random() * n)
}

function pickOne(arr) {
  return arr[randInt(arr.length)]
}

function shuffle(arr) {
  const a = [...arr]
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[a[i], a[j]] = [a[j], a[i]]
  }
  return a
}

function clearHintTimer() {
  if (hintTimer) {
    clearTimeout(hintTimer)
    hintTimer = null
  }
}

function showToast(next) {
  if (toastTimer) clearTimeout(toastTimer)
  toast.value = next
  toastTimer = setTimeout(() => {
    toast.value = null
  }, 1200)
}

function trySpeak(text) {
  if (!settings.sound) return
  // 仅在用户手势触发的点击流程里调用，避免被浏览器拦截
  if (!('speechSynthesis' in window)) return
  const u = new SpeechSynthesisUtterance(text)
  u.lang = 'zh-CN'
  try {
    window.speechSynthesis.cancel()
    window.speechSynthesis.speak(u)
  } catch {
    // ignore
  }
}

const pool = computed(() => ITEMS.slice(0, unlockedCount.value))

const question = ref(null) // { targetId, options:[item], command }

const progressText = computed(() => `${round.value}/${settings.rounds}`)
const accuracyText = computed(() => {
  const answered = round.value
  if (!answered) return '—'
  return `${Math.round((score.value / answered) * 100)}%`
})

function makeQuestion() {
  clearHintTimer()
  hintId.value = null
  selectedId.value = null
  locked.value = false

  const basePool = pool.value
  const target = pickOne(basePool)

  const opts = new Map()
  opts.set(target.id, target)
  while (opts.size < Math.min(settings.optionsPerRound, basePool.length)) {
    const it = pickOne(basePool)
    opts.set(it.id, it)
  }

  const options = shuffle([...opts.values()])
  const commandTpl = pickOne(COMMANDS)
  const command = commandTpl.replace('{x}', target.name)

  question.value = {
    targetId: target.id,
    options,
    command,
  }

  if (settings.hintMode) {
    // 彩蛋：猫猫“察言观色”，过一会儿才更容易蒙对
    hintTimer = setTimeout(() => {
      hintId.value = target.id
      setTimeout(() => {
        hintId.value = null
      }, 900)
    }, 900)
  }
}

function maybeUnlock() {
  if (unlockedCount.value >= ITEMS.length) return
  // “道具池不断扩充”：每连续答对 3 次，加入一个新道具
  if (streak.value > 0 && streak.value % 3 === 0) {
    const next = ITEMS[unlockedCount.value]
    unlockedCount.value += 1
    showToast({
      type: 'info',
      title: '道具池扩充',
      detail: `新增「${next.name}」${next.emoji}`,
    })
  }
}

function startGame() {
  phase.value = PHASE.playing
  round.value = 0
  score.value = 0
  streak.value = 0
  bestStreak.value = 0
  unlockedCount.value = 2
  toast.value = null
  makeQuestion()
}

function endGame() {
  clearHintTimer()
  locked.value = true
  phase.value = PHASE.end
}

function nextRound() {
  if (round.value >= settings.rounds) {
    endGame()
    return
  }
  makeQuestion()
}

function choose(item) {
  if (!question.value || locked.value) return
  locked.value = true
  selectedId.value = item.id

  const isGood = item.id === question.value.targetId
  round.value += 1

  if (isGood) {
    score.value += 1
    streak.value += 1
    bestStreak.value = Math.max(bestStreak.value, streak.value)
    showToast({ type: 'good', title: '真棒', detail: '（懂了！也可能只是蒙对了）' })
    trySpeak('真棒')
    maybeUnlock()
  } else {
    streak.value = 0
    showToast({ type: 'bad', title: '再想想', detail: '（其实是在看你脸色…）' })
  }

  setTimeout(() => {
    locked.value = false
    nextRound()
  }, 650)
}

function replayCommand() {
  if (!question.value) return
  trySpeak(question.value.command)
  showToast({ type: 'info', title: '指令', detail: question.value.command })
}

watch(
  () => settings.optionsPerRound,
  () => {
    if (phase.value === PHASE.playing) makeQuestion()
  },
)

onBeforeUnmount(() => {
  clearHintTimer()
  if (toastTimer) clearTimeout(toastTimer)
})
</script>

<template>
  <div class="wrap">
    <header class="top">
      <div class="brand">
        <div class="title">萝卜还是纸巾？</div>
        <div class="sub">选对就「真棒」：年末治愈系选择题</div>
      </div>

      <div class="stats" v-if="phase !== PHASE.intro">
        <div class="pill"><span>进度</span><b>{{ progressText }}</b></div>
        <div class="pill"><span>得分</span><b>{{ score }}</b></div>
        <div class="pill"><span>命中</span><b>{{ accuracyText }}</b></div>
        <div class="pill"><span>连对</span><b>{{ streak }}</b></div>
      </div>
    </header>

    <main class="panel">
      <section v-if="phase === PHASE.intro" class="intro">
        <div class="hero">
          <div class="heroEmoji">🐱</div>
          <div class="heroText">
            <div class="heroTitle">你来当小猫，做选择题</div>
            <div class="heroDesc">
              地上摆一堆东西，我喊名字，你去选；选对就夸你「真棒」。
              <br />
              彩蛋：打开“察言观色模式”，小猫会更容易“蒙对”。
            </div>
          </div>
        </div>

        <div class="settings">
          <label class="opt">
            <span>题量</span>
            <select v-model.number="settings.rounds">
              <option :value="5">5</option>
              <option :value="10">10</option>
              <option :value="15">15</option>
            </select>
          </label>
          <label class="opt">
            <span>每题摆几样</span>
            <select v-model.number="settings.optionsPerRound">
              <option :value="2">2</option>
              <option :value="3">3</option>
              <option :value="4">4</option>
            </select>
          </label>
          <label class="opt toggle">
            <span>察言观色模式</span>
            <input type="checkbox" v-model="settings.hintMode" />
          </label>
          <label class="opt toggle">
            <span>语音/读指令</span>
            <input type="checkbox" v-model="settings.sound" />
          </label>
        </div>

        <div class="actions">
          <button class="btn primary" @click="startGame">开始考试</button>
        </div>
      </section>

      <section v-else-if="phase === PHASE.playing" class="game">
        <div class="commandRow">
          <div class="command">
            <div class="cmdLabel">我说：</div>
            <div class="cmdText">{{ question?.command }}</div>
          </div>
          <button class="btn ghost" type="button" @click="replayCommand">再说一遍</button>
        </div>

        <div class="grid" :style="{ '--cols': Math.min(settings.optionsPerRound, pool.length) }">
          <button
            v-for="it in question?.options || []"
            :key="it.id"
            class="card"
            type="button"
            :disabled="locked"
            :data-state="
              selectedId
                ? it.id === question.targetId
                  ? 'good'
                  : it.id === selectedId
                    ? 'bad'
                    : 'idle'
                : 'idle'
            "
            :data-hint="hintId === it.id ? 'on' : 'off'"
            @click="choose(it)"
          >
            <div class="emoji" aria-hidden="true">{{ it.emoji }}</div>
            <div class="name">{{ it.name }}</div>
          </button>
        </div>

        <div class="hintline">
          <span class="muted">道具池：</span>
          <span v-for="it in pool" :key="it.id" class="chip">{{ it.emoji }} {{ it.name }}</span>
        </div>
      </section>

      <section v-else class="end">
        <div class="result">
          <div class="resultTitle">考试结束</div>
          <div class="resultNums">
            <div class="big"><span>得分</span><b>{{ score }}</b></div>
            <div class="big"><span>命中率</span><b>{{ accuracyText }}</b></div>
            <div class="big"><span>最高连对</span><b>{{ bestStreak }}</b></div>
          </div>
          <div class="resultText">
            现实逻辑版：不会分辨也没关系，<b>会配合</b>就已经很乖了。
            <br />
            成年人的童话：选对一个“萝卜”，就能得到一句真诚的「真棒」。
          </div>
          <div class="actions">
            <button class="btn primary" @click="startGame">再来一局</button>
            <button class="btn" @click="phase = PHASE.intro">回到设置</button>
          </div>
        </div>
      </section>
    </main>

    <transition name="toast">
      <div v-if="toast" class="toast" :data-type="toast.type" role="status" aria-live="polite">
        <div class="toastTitle">{{ toast.title }}</div>
        <div class="toastDetail">{{ toast.detail }}</div>
      </div>
    </transition>
  </div>
</template>

<style scoped>
.wrap {
  width: min(980px, 100%);
  display: grid;
  grid-template-rows: auto 1fr;
  gap: 14px;
}

.top {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 16px;
  flex-wrap: wrap;
}

.brand .title {
  font-size: 24px;
  font-weight: 800;
  letter-spacing: -0.02em;
}
.brand .sub {
  margin-top: 4px;
  color: var(--muted);
  font-size: 13px;
}

.stats {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  justify-content: flex-end;
}
.pill {
  display: inline-flex;
  gap: 8px;
  align-items: baseline;
  padding: 8px 10px;
  border: 1px solid var(--border);
  border-radius: 999px;
  background: color-mix(in srgb, var(--panel) 80%, transparent);
}
.pill span {
  color: var(--muted);
  font-size: 12px;
}
.pill b {
  font-variant-numeric: tabular-nums;
}

.panel {
  border: 1px solid var(--border);
  background: var(--panel);
  border-radius: 18px;
  padding: 18px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.08);
}

.intro .hero {
  display: flex;
  gap: 14px;
  align-items: center;
  padding: 14px;
  border: 1px dashed color-mix(in srgb, var(--border) 75%, transparent);
  border-radius: 14px;
  background: color-mix(in srgb, var(--panel) 70%, white);
}
.heroEmoji {
  font-size: 42px;
  width: 56px;
  height: 56px;
  display: grid;
  place-items: center;
  border-radius: 14px;
  background: radial-gradient(circle at 30% 30%, #fff, #f2f2f2);
  border: 1px solid var(--border);
}
.heroTitle {
  font-weight: 800;
  letter-spacing: -0.01em;
}
.heroDesc {
  margin-top: 6px;
  color: var(--muted);
  font-size: 13px;
  line-height: 1.45;
}

.settings {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px;
  margin-top: 14px;
}
.opt {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 12px;
  border: 1px solid var(--border);
  border-radius: 12px;
  background: color-mix(in srgb, var(--panel) 85%, transparent);
}
.opt span {
  color: var(--muted);
  font-size: 13px;
}
.opt select {
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 8px 10px;
  background: white;
}
.opt.toggle input {
  width: 42px;
  height: 22px;
}

.actions {
  margin-top: 14px;
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.btn {
  border: 1px solid var(--border);
  background: white;
  border-radius: 12px;
  padding: 10px 14px;
  font-weight: 650;
  cursor: pointer;
  transition: transform 0.08s ease, box-shadow 0.12s ease, border-color 0.12s ease;
}
.btn:hover {
  border-color: color-mix(in srgb, var(--accent) 60%, var(--border));
  box-shadow: 0 8px 18px rgba(0, 0, 0, 0.06);
}
.btn:active {
  transform: translateY(1px);
}
.btn.primary {
  background: linear-gradient(135deg, color-mix(in srgb, var(--accent) 85%, white), var(--accent2));
  border-color: color-mix(in srgb, var(--accent) 45%, var(--border));
  color: #0b1020;
}
.btn.ghost {
  background: transparent;
}

.commandRow {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  flex-wrap: wrap;
  margin-bottom: 12px;
}
.command {
  display: flex;
  gap: 10px;
  align-items: baseline;
}
.cmdLabel {
  color: var(--muted);
  font-size: 12px;
}
.cmdText {
  font-weight: 850;
  letter-spacing: -0.01em;
}

.grid {
  --cols: 3;
  display: grid;
  grid-template-columns: repeat(var(--cols), minmax(0, 1fr));
  gap: 10px;
}

.card {
  text-align: left;
  border-radius: 16px;
  padding: 14px;
  border: 1px solid var(--border);
  background: white;
  cursor: pointer;
  transition: transform 0.08s ease, box-shadow 0.12s ease, border-color 0.12s ease;
  position: relative;
  overflow: hidden;
}
.card:disabled {
  cursor: not-allowed;
  opacity: 0.95;
}
.card:hover:not(:disabled) {
  border-color: color-mix(in srgb, var(--accent) 55%, var(--border));
  box-shadow: 0 10px 22px rgba(0, 0, 0, 0.07);
}
.emoji {
  font-size: 30px;
  line-height: 1;
}
.name {
  margin-top: 10px;
  font-weight: 800;
}

.card[data-state='good'] {
  border-color: color-mix(in srgb, var(--good) 55%, var(--border));
  box-shadow: 0 10px 26px rgba(22, 163, 74, 0.18);
}
.card[data-state='bad'] {
  border-color: color-mix(in srgb, var(--bad) 55%, var(--border));
  box-shadow: 0 10px 26px rgba(220, 38, 38, 0.14);
  animation: shake 260ms ease-in-out 1;
}

.card[data-hint='on']::after {
  content: '';
  position: absolute;
  inset: -2px;
  border-radius: 18px;
  border: 2px solid color-mix(in srgb, var(--accent) 75%, white);
  box-shadow: 0 0 0 6px rgba(255, 214, 10, 0.2);
  pointer-events: none;
  animation: pulse 900ms ease-in-out infinite;
}

.hintline {
  margin-top: 12px;
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  align-items: center;
}
.muted {
  color: var(--muted);
  font-size: 12px;
}
.chip {
  font-size: 12px;
  padding: 6px 10px;
  border-radius: 999px;
  border: 1px solid var(--border);
  background: color-mix(in srgb, var(--panel) 80%, white);
}

.end .result {
  padding: 14px;
  border: 1px solid var(--border);
  border-radius: 16px;
  background: color-mix(in srgb, var(--panel) 70%, white);
}
.resultTitle {
  font-size: 18px;
  font-weight: 900;
  margin-bottom: 10px;
}
.resultNums {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 10px;
}
.big {
  border: 1px solid var(--border);
  border-radius: 14px;
  padding: 10px 12px;
  background: white;
  display: grid;
  gap: 4px;
}
.big span {
  color: var(--muted);
  font-size: 12px;
}
.big b {
  font-size: 20px;
  font-variant-numeric: tabular-nums;
}
.resultText {
  margin-top: 10px;
  color: var(--muted);
  font-size: 13px;
  line-height: 1.45;
}

.toast {
  position: fixed;
  left: 50%;
  bottom: 18px;
  transform: translateX(-50%);
  border-radius: 14px;
  padding: 12px 14px;
  border: 1px solid var(--border);
  background: white;
  box-shadow: 0 14px 30px rgba(0, 0, 0, 0.12);
  min-width: min(420px, calc(100vw - 32px));
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
  border-color: color-mix(in srgb, var(--good) 55%, var(--border));
}
.toast[data-type='bad'] {
  border-color: color-mix(in srgb, var(--bad) 55%, var(--border));
}
.toast[data-type='info'] {
  border-color: color-mix(in srgb, var(--accent) 55%, var(--border));
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

@keyframes shake {
  0% {
    transform: translateX(0);
  }
  25% {
    transform: translateX(-4px);
  }
  50% {
    transform: translateX(4px);
  }
  75% {
    transform: translateX(-3px);
  }
  100% {
    transform: translateX(0);
  }
}

@keyframes pulse {
  0%,
  100% {
    opacity: 0.25;
  }
  50% {
    opacity: 1;
  }
}

@media (max-width: 640px) {
  .settings {
    grid-template-columns: 1fr;
  }
  .resultNums {
    grid-template-columns: 1fr;
  }
  .grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}
</style>

