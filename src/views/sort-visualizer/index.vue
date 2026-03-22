<script setup lang="ts">
import { ref, computed, watch, onMounted, onUnmounted, nextTick } from 'vue'
import { algorithms, algorithmMap } from './algorithms'
import type { SortStep, AlgorithmId, VisualizationMode, InputType, PivotStrategy } from './types'

// ── State ──────────────────────────────────────────────────────────────────
const isDark = ref(true)
const soundEnabled = ref(false)
const showLearn = ref(true)
const vizMode = ref<VisualizationMode>('bar')
const currentAlgoId = ref<AlgorithmId>('bubble')
const pivotStrategy = ref<PivotStrategy>('last')
const arraySize = ref(60)
const speed = ref(5)
const inputType = ref<InputType>('random')
const customInput = ref('')
const showCustomInput = ref(false)

const isPlaying = ref(false)
const currentStepIndex = ref(0)
const steps = ref<SortStep[]>([])
const startTime = ref(0)
const elapsedMs = ref(0)

// Race mode
const raceMode = ref(false)
const raceAlgo1 = ref<AlgorithmId>('bubble')
const raceAlgo2 = ref<AlgorithmId>('merge')
const raceSteps1 = ref<SortStep[]>([])
const raceSteps2 = ref<SortStep[]>([])
const raceIdx1 = ref(0)
const raceIdx2 = ref(0)
const raceFinished1 = ref(false)
const raceFinished2 = ref(false)
const raceWinner = ref<1 | 2 | null>(null)

const canvas = ref<HTMLCanvasElement | null>(null)
const raceCanvas1 = ref<HTMLCanvasElement | null>(null)
const raceCanvas2 = ref<HTMLCanvasElement | null>(null)
const vizContainer = ref<HTMLDivElement | null>(null)

let audioCtx: AudioContext | null = null
let animTimer: ReturnType<typeof setTimeout> | null = null
let sharedInitArray: number[] = []

// ── Computed ───────────────────────────────────────────────────────────────
const currentAlgo = computed(() => algorithmMap.get(currentAlgoId.value)!)
const currentStep = computed<SortStep>(
  () =>
    steps.value[currentStepIndex.value] ?? {
      array: [],
      comparing: [],
      swapping: [],
      sorted: [],
      description: 'Nhấn ▶ để bắt đầu',
      pseudoCodeLine: -1,
      comparisons: 0,
      swaps: 0,
    },
)
const progress = computed(() =>
  steps.value.length > 0 ? Math.round((currentStepIndex.value / (steps.value.length - 1)) * 100) : 0,
)
const speedDelay = computed(() => Math.max(1, Math.round(300 / speed.value)))
const isDone = computed(() => currentStepIndex.value >= steps.value.length - 1 && steps.value.length > 0)

// ── Array generation ───────────────────────────────────────────────────────
function generateArray(): number[] {
  const n = currentAlgoId.value === 'bogo' ? Math.min(arraySize.value, 8) : arraySize.value
  switch (inputType.value) {
    case 'nearly-sorted': {
      const a = Array.from({ length: n }, (_, i) => i + 1)
      const swaps = Math.max(1, Math.floor(n * 0.1))
      for (let i = 0; i < swaps; i++) {
        const x = Math.floor(Math.random() * n)
        const y = Math.floor(Math.random() * n)
        ;[a[x], a[y]] = [a[y], a[x]]
      }
      return a
    }
    case 'reversed':
      return Array.from({ length: n }, (_, i) => n - i)
    case 'duplicates':
      return Array.from({ length: n }, () => Math.floor(Math.random() * Math.max(5, Math.floor(n / 5))) + 1)
    default:
      return Array.from({ length: n }, () => Math.floor(Math.random() * 100) + 1)
  }
}

function parseCustomInput(): number[] | null {
  const nums = customInput.value
    .split(/[\s,]+/)
    .map((s) => parseInt(s.trim()))
    .filter((n) => !isNaN(n) && n > 0)
  return nums.length >= 2 ? nums.slice(0, 200) : null
}

// ── Step generation ────────────────────────────────────────────────────────
function generateSteps(arr: number[], algoId: AlgorithmId): SortStep[] {
  const def = algorithmMap.get(algoId)!
  const gen = def.generate(arr, pivotStrategy.value)
  const stepsArr: SortStep[] = []
  const MAX = 100000
  while (stepsArr.length < MAX) {
    const r = gen.next()
    if (r.done) break
    stepsArr.push(r.value)
  }
  return stepsArr
}

function initSort() {
  stop()

  let arr: number[]
  if (inputType.value === 'duplicates' && showCustomInput.value && customInput.value.trim()) {
    arr = parseCustomInput() ?? generateArray()
  } else if (showCustomInput.value && customInput.value.trim()) {
    arr = parseCustomInput() ?? generateArray()
  } else {
    arr = generateArray()
  }

  sharedInitArray = [...arr]
  steps.value = generateSteps(arr, currentAlgoId.value)
  currentStepIndex.value = 0
  elapsedMs.value = 0

  nextTick(drawFrame)
}

function initRace() {
  stop()
  const arr = generateArray()
  sharedInitArray = [...arr]
  raceSteps1.value = generateSteps([...arr], raceAlgo1.value)
  raceSteps2.value = generateSteps([...arr], raceAlgo2.value)
  raceIdx1.value = 0
  raceIdx2.value = 0
  raceFinished1.value = false
  raceFinished2.value = false
  raceWinner.value = null
  nextTick(() => {
    drawRaceFrame(raceCanvas1.value, raceSteps1.value, raceIdx1.value)
    drawRaceFrame(raceCanvas2.value, raceSteps2.value, raceIdx2.value)
  })
}

// ── Playback control ────────────────────────────────────────────────────────
function play() {
  if (isPlaying.value) return
  if (isDone.value) { currentStepIndex.value = 0 }
  isPlaying.value = true
  startTime.value = Date.now() - elapsedMs.value

  function tick() {
    if (!isPlaying.value) return

    if (raceMode.value) {
      let advanced = false
      if (raceIdx1.value < raceSteps1.value.length - 1) {
        raceIdx1.value++
        drawRaceFrame(raceCanvas1.value, raceSteps1.value, raceIdx1.value)
        advanced = true
        if (raceIdx1.value >= raceSteps1.value.length - 1 && !raceFinished1.value) {
          raceFinished1.value = true
          if (!raceWinner.value) raceWinner.value = 1
        }
      }
      if (raceIdx2.value < raceSteps2.value.length - 1) {
        raceIdx2.value++
        drawRaceFrame(raceCanvas2.value, raceSteps2.value, raceIdx2.value)
        advanced = true
        if (raceIdx2.value >= raceSteps2.value.length - 1 && !raceFinished2.value) {
          raceFinished2.value = true
          if (!raceWinner.value) raceWinner.value = 2
        }
      }
      if (!advanced) { stop(); return }
    } else {
      if (currentStepIndex.value >= steps.value.length - 1) { stop(); return }
      currentStepIndex.value++
      elapsedMs.value = Date.now() - startTime.value
      drawFrame()
      if (soundEnabled.value) playSound(currentStep.value)
    }

    animTimer = setTimeout(tick, speedDelay.value)
  }
  tick()
}

function stop() {
  isPlaying.value = false
  if (animTimer) { clearTimeout(animTimer); animTimer = null }
}

function togglePlay() {
  if (isPlaying.value) stop()
  else play()
}

function stepForward() {
  stop()
  if (raceMode.value) {
    if (raceIdx1.value < raceSteps1.value.length - 1) {
      raceIdx1.value++
      drawRaceFrame(raceCanvas1.value, raceSteps1.value, raceIdx1.value)
    }
    if (raceIdx2.value < raceSteps2.value.length - 1) {
      raceIdx2.value++
      drawRaceFrame(raceCanvas2.value, raceSteps2.value, raceIdx2.value)
    }
  } else if (currentStepIndex.value < steps.value.length - 1) {
    currentStepIndex.value++
    drawFrame()
  }
}

function stepBack() {
  stop()
  if (!raceMode.value && currentStepIndex.value > 0) {
    currentStepIndex.value--
    drawFrame()
  }
}

function shuffle() {
  stop()
  showCustomInput.value = false
  initSort()
}

// ── Canvas drawing ──────────────────────────────────────────────────────────
function getColors(dark: boolean) {
  return {
    default: dark ? '#60a5fa' : '#3b82f6',
    comparing: '#fbbf24',
    swapping: '#f87171',
    sorted: '#34d399',
    pivot: '#c084fc',
    bg: dark ? '#111827' : '#f3f4f6',
  }
}

function drawOnCanvas(
  c: HTMLCanvasElement,
  step: SortStep,
  mode: VisualizationMode = 'bar',
  dark = true,
) {
  const ctx = c.getContext('2d')
  if (!ctx) return

  const dpr = window.devicePixelRatio || 1
  const W = c.width
  const H = c.height
  const w = W / dpr
  const h = H / dpr

  ctx.clearRect(0, 0, W, H)

  const { array, comparing, swapping, sorted, pivot } = step
  const n = array.length
  if (n === 0) return
  const max = Math.max(...array) || 1

  const comparingSet = new Set(comparing)
  const swappingSet = new Set(swapping)
  const sortedSet = new Set(sorted)
  const colors = getColors(dark)

  function getColor(i: number): string {
    if (i === pivot) return colors.pivot
    if (swappingSet.has(i)) return colors.swapping
    if (comparingSet.has(i)) return colors.comparing
    if (sortedSet.has(i)) return colors.sorted
    return colors.default
  }

  ctx.save()
  ctx.scale(dpr, dpr)

  if (mode === 'bar') {
    const bw = w / n
    for (let i = 0; i < n; i++) {
      const bh = Math.max(1, (array[i] / max) * (h - 4))
      ctx.fillStyle = getColor(i)
      ctx.fillRect(i * bw, h - bh, Math.max(1, bw - (n > 80 ? 0 : 1)), bh)
    }
  } else if (mode === 'hue') {
    const bw = w / n
    for (let i = 0; i < n; i++) {
      const hue = (array[i] / max) * 300
      let lit = 50
      if (swappingSet.has(i)) lit = 75
      else if (comparingSet.has(i)) lit = 80
      ctx.fillStyle = `hsl(${hue}, 100%, ${lit}%)`
      ctx.fillRect(i * bw, 0, Math.max(1, bw - (n > 80 ? 0 : 0.5)), h)
    }
  } else if (mode === 'dot') {
    const r = Math.max(1.5, Math.min(6, (w / n) * 0.4))
    for (let i = 0; i < n; i++) {
      const x = (i + 0.5) * (w / n)
      const y = h - 4 - (array[i] / max) * (h - 12)
      ctx.beginPath()
      ctx.arc(x, y, r, 0, Math.PI * 2)
      ctx.fillStyle = getColor(i)
      ctx.fill()
    }
  }

  ctx.restore()
}

function resizeCanvas(c: HTMLCanvasElement | null, container: HTMLElement | null) {
  if (!c || !container) return
  const dpr = window.devicePixelRatio || 1
  const rect = container.getBoundingClientRect()
  c.width = rect.width * dpr
  c.height = rect.height * dpr
  c.style.width = rect.width + 'px'
  c.style.height = rect.height + 'px'
}

function drawFrame() {
  if (!canvas.value) return
  const step = steps.value[currentStepIndex.value]
  if (!step) return
  drawOnCanvas(canvas.value, step, vizMode.value, isDark.value)
}

function drawRaceFrame(c: HTMLCanvasElement | null, stepsArr: SortStep[], idx: number) {
  if (!c) return
  const step = stepsArr[idx]
  if (!step) return
  drawOnCanvas(c, step, 'bar', isDark.value)
}

function handleResize() {
  resizeCanvas(canvas.value, vizContainer.value)
  drawFrame()
  if (raceMode.value) {
    const rc1 = document.getElementById('race-canvas-1') as HTMLCanvasElement | null
    const rc2 = document.getElementById('race-canvas-2') as HTMLCanvasElement | null
    const rp1 = document.getElementById('race-panel-1')
    const rp2 = document.getElementById('race-panel-2')
    resizeCanvas(rc1, rp1)
    resizeCanvas(rc2, rp2)
    drawRaceFrame(rc1, raceSteps1.value, raceIdx1.value)
    drawRaceFrame(rc2, raceSteps2.value, raceIdx2.value)
  }
}

// ── Sound ──────────────────────────────────────────────────────────────────
function playSound(step: SortStep) {
  if (!step.comparing.length && !step.swapping.length) return
  try {
    if (!audioCtx) audioCtx = new AudioContext()
    const arr = step.array
    const max = Math.max(...arr) || 1
    const indices = step.swapping.length ? step.swapping : step.comparing
    const val = arr[indices[0]] ?? 1
    const freq = 180 + (val / max) * 880

    const osc = audioCtx.createOscillator()
    const gain = audioCtx.createGain()
    osc.connect(gain)
    gain.connect(audioCtx.destination)
    osc.type = step.swapping.length ? 'sawtooth' : 'sine'
    osc.frequency.value = freq
    gain.gain.setValueAtTime(0.08, audioCtx.currentTime)
    gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.08)
    osc.start()
    osc.stop(audioCtx.currentTime + 0.08)
  } catch {}
}

async function playCompletionSound() {
  if (!soundEnabled.value) return
  try {
    if (!audioCtx) audioCtx = new AudioContext()
    const notes = [261, 294, 330, 349, 392, 440, 494, 523]
    for (let i = 0; i < notes.length; i++) {
      await new Promise<void>((res) => setTimeout(res, 60))
      const osc = audioCtx!.createOscillator()
      const gain = audioCtx!.createGain()
      osc.connect(gain)
      gain.connect(audioCtx!.destination)
      osc.frequency.value = notes[i]
      gain.gain.setValueAtTime(0.1, audioCtx!.currentTime)
      gain.gain.exponentialRampToValueAtTime(0.001, audioCtx!.currentTime + 0.15)
      osc.start()
      osc.stop(audioCtx!.currentTime + 0.15)
    }
  } catch {}
}

// ── Watchers ────────────────────────────────────────────────────────────────
watch(currentStepIndex, () => {
  drawFrame()
  if (isDone.value) playCompletionSound()
})
watch(isDark, drawFrame)
watch(vizMode, drawFrame)
watch(currentAlgoId, () => { stop(); initSort() })
watch(arraySize, () => { stop(); initSort() })
watch(inputType, () => { stop(); initSort() })
watch(raceMode, async (val) => {
  stop()
  await nextTick()
  if (val) {
    await nextTick()
    handleResize()
    initRace()
  } else {
    await nextTick()
    resizeCanvas(canvas.value, vizContainer.value)
    initSort()
  }
})

// ── Lifecycle ───────────────────────────────────────────────────────────────
onMounted(async () => {
  await nextTick()
  resizeCanvas(canvas.value, vizContainer.value)
  initSort()
  window.addEventListener('resize', handleResize)
})
onUnmounted(() => {
  stop()
  window.removeEventListener('resize', handleResize)
  audioCtx?.close()
})

// ── Helpers ──────────────────────────────────────────────────────────────────
const speedLabel = computed(() => {
  if (speed.value <= 2) return 'Rất chậm'
  if (speed.value <= 4) return 'Chậm'
  if (speed.value <= 6) return 'Bình thường'
  if (speed.value <= 8) return 'Nhanh'
  return 'Rất nhanh'
})

function formatMs(ms: number) {
  if (ms < 1000) return `${ms}ms`
  return `${(ms / 1000).toFixed(1)}s`
}

const raceStep1 = computed(() => raceSteps1.value[raceIdx1.value])
const raceStep2 = computed(() => raceSteps2.value[raceIdx2.value])
const raceProg1 = computed(() =>
  raceSteps1.value.length > 0 ? Math.round((raceIdx1.value / (raceSteps1.value.length - 1)) * 100) : 0,
)
const raceProg2 = computed(() =>
  raceSteps2.value.length > 0 ? Math.round((raceIdx2.value / (raceSteps2.value.length - 1)) * 100) : 0,
)

const algoGroups = [
  { label: 'Cơ bản', ids: ['bubble', 'selection', 'insertion'] as AlgorithmId[] },
  { label: 'Nâng cao', ids: ['merge', 'quick', 'heap', 'shell'] as AlgorithmId[] },
  { label: 'Đặc biệt', ids: ['counting', 'radix', 'bogo'] as AlgorithmId[] },
]
</script>

<template>
  <div
    class="flex flex-col min-h-screen font-mono text-sm transition-colors duration-200"
    :class="isDark ? 'bg-gray-950 text-gray-100' : 'bg-gray-50 text-gray-900'"
  >
    <!-- Header -->
    <header
      class="sticky top-0 z-20 flex items-center justify-between px-4 py-2 border-b"
      :class="isDark ? 'bg-gray-900 border-gray-800' : 'bg-white border-gray-200'"
    >
      <div class="flex items-center gap-2">
        <a
          href="/"
          class="text-xs px-2 py-1 rounded transition-colors"
          :class="isDark ? 'text-gray-400 hover:text-white hover:bg-gray-800' : 'text-gray-500 hover:text-gray-900 hover:bg-gray-100'"
        >← Trang chủ</a>
        <span class="text-gray-600">|</span>
        <span class="font-bold text-base" :class="isDark ? 'text-blue-400' : 'text-blue-600'">Sort Visualizer</span>
      </div>
      <div class="flex items-center gap-1">
        <button
          @click="showLearn = !showLearn"
          class="px-2 py-1 rounded text-xs transition-colors"
          :class="[
            showLearn
              ? isDark ? 'bg-blue-600 text-white' : 'bg-blue-500 text-white'
              : isDark ? 'text-gray-400 hover:bg-gray-800' : 'text-gray-500 hover:bg-gray-100'
          ]"
          title="Chế độ học"
        >📖 Học</button>
        <button
          @click="raceMode = !raceMode"
          class="px-2 py-1 rounded text-xs transition-colors"
          :class="[
            raceMode
              ? 'bg-orange-500 text-white'
              : isDark ? 'text-gray-400 hover:bg-gray-800' : 'text-gray-500 hover:bg-gray-100'
          ]"
          title="Race Mode"
        >⚔️ Race</button>
        <button
          @click="soundEnabled = !soundEnabled"
          class="px-2 py-1 rounded text-xs transition-colors"
          :class="isDark ? 'text-gray-400 hover:bg-gray-800' : 'text-gray-500 hover:bg-gray-100'"
          :title="soundEnabled ? 'Tắt âm thanh' : 'Bật âm thanh'"
        >{{ soundEnabled ? '🔊' : '🔇' }}</button>
        <button
          @click="isDark = !isDark"
          class="px-2 py-1 rounded text-xs transition-colors"
          :class="isDark ? 'text-gray-400 hover:bg-gray-800' : 'text-gray-500 hover:bg-gray-100'"
        >{{ isDark ? '☀️' : '🌙' }}</button>
      </div>
    </header>

    <!-- Algorithm selector -->
    <div
      class="px-3 py-2 border-b flex flex-wrap gap-1 items-center"
      :class="isDark ? 'bg-gray-900/50 border-gray-800' : 'bg-white border-gray-200'"
    >
      <template v-if="!raceMode">
        <template v-for="group in algoGroups" :key="group.label">
          <span class="text-xs mr-1" :class="isDark ? 'text-gray-600' : 'text-gray-400'">{{ group.label }}:</span>
          <button
            v-for="id in group.ids"
            :key="id"
            @click="currentAlgoId = id"
            class="px-2 py-0.5 rounded text-xs transition-all"
            :class="[
              currentAlgoId === id
                ? isDark ? 'bg-blue-600 text-white' : 'bg-blue-500 text-white'
                : isDark ? 'bg-gray-800 text-gray-300 hover:bg-gray-700' : 'bg-gray-100 text-gray-600 hover:bg-gray-200'
            ]"
          >{{ algorithmMap.get(id)?.name.replace(' Sort', '') }}</button>
          <span class="mr-1" />
        </template>
      </template>
      <template v-else>
        <span class="text-xs font-bold text-orange-400">⚔️ Race Mode — Chọn 2 thuật toán:</span>
        <select
          v-model="raceAlgo1"
          class="ml-2 px-2 py-0.5 rounded text-xs"
          :class="isDark ? 'bg-gray-800 text-white border border-gray-700' : 'bg-white border border-gray-300'"
          @change="initRace"
        >
          <option v-for="a in algorithms" :key="a.id" :value="a.id">{{ a.name }}</option>
        </select>
        <span class="text-orange-400">vs</span>
        <select
          v-model="raceAlgo2"
          class="px-2 py-0.5 rounded text-xs"
          :class="isDark ? 'bg-gray-800 text-white border border-gray-700' : 'bg-white border border-gray-300'"
          @change="initRace"
        >
          <option v-for="a in algorithms" :key="a.id" :value="a.id">{{ a.name }}</option>
        </select>
        <button
          @click="initRace"
          class="px-2 py-0.5 rounded text-xs bg-orange-500 text-white hover:bg-orange-400 transition-colors"
        >🔀 Mảng mới</button>
        <span
          v-if="raceWinner"
          class="ml-2 font-bold"
          :class="raceWinner === 1 ? 'text-yellow-400' : 'text-blue-400'"
        >
          🏆 {{ raceWinner === 1 ? algorithmMap.get(raceAlgo1)?.name : algorithmMap.get(raceAlgo2)?.name }} thắng!
        </span>
      </template>
    </div>

    <!-- Main area -->
    <div class="flex-1 flex min-h-0">

      <!-- Left: Visualization + controls -->
      <div class="flex-1 flex flex-col p-3 gap-2 min-h-0">

        <!-- Viz mode + pivot + quick options (non-race) -->
        <div v-if="!raceMode" class="flex flex-wrap items-center gap-2">
          <div class="flex rounded overflow-hidden border" :class="isDark ? 'border-gray-700' : 'border-gray-300'">
            <button
              v-for="m in ([['bar','📊 Bar'],['hue','🌈 Hue'],['dot','⚫ Dot']] as const)"
              :key="m[0]"
              @click="vizMode = m[0]"
              class="px-2 py-0.5 text-xs transition-colors"
              :class="[
                vizMode === m[0]
                  ? isDark ? 'bg-blue-600 text-white' : 'bg-blue-500 text-white'
                  : isDark ? 'bg-gray-800 text-gray-400 hover:bg-gray-700' : 'bg-white text-gray-500 hover:bg-gray-100'
              ]"
            >{{ m[1] }}</button>
          </div>

          <template v-if="currentAlgoId === 'quick'">
            <span class="text-xs" :class="isDark ? 'text-gray-500' : 'text-gray-400'">Pivot:</span>
            <select
              v-model="pivotStrategy"
              class="px-2 py-0.5 rounded text-xs"
              :class="isDark ? 'bg-gray-800 text-white border border-gray-700' : 'bg-white border border-gray-300'"
              @change="initSort"
            >
              <option value="last">Last</option>
              <option value="first">First</option>
              <option value="random">Random</option>
              <option value="median">Median of 3</option>
            </select>
          </template>

          <div class="ml-auto flex items-center gap-2">
            <span class="text-xs" :class="isDark ? 'text-gray-500' : 'text-gray-400'">Input:</span>
            <select
              v-model="inputType"
              class="px-2 py-0.5 rounded text-xs"
              :class="isDark ? 'bg-gray-800 text-white border border-gray-700' : 'bg-white border border-gray-300'"
            >
              <option value="random">Ngẫu nhiên</option>
              <option value="nearly-sorted">Gần đã sort</option>
              <option value="reversed">Đảo ngược</option>
              <option value="duplicates">Nhiều trùng lặp</option>
            </select>
            <button
              @click="showCustomInput = !showCustomInput"
              class="px-2 py-0.5 rounded text-xs transition-colors"
              :class="isDark ? 'bg-gray-800 text-gray-300 hover:bg-gray-700' : 'bg-gray-100 text-gray-600 hover:bg-gray-200'"
            >✎ Custom</button>
          </div>
        </div>

        <!-- Custom input -->
        <div v-if="showCustomInput && !raceMode" class="flex gap-2 items-center">
          <input
            v-model="customInput"
            placeholder="Nhập số cách nhau bởi dấu phẩy hoặc space: 5, 3, 8, 1, 9..."
            class="flex-1 px-2 py-1 rounded text-xs border"
            :class="isDark ? 'bg-gray-800 text-white border-gray-700 placeholder-gray-600' : 'bg-white border-gray-300 placeholder-gray-400'"
          />
          <button
            @click="initSort"
            class="px-3 py-1 rounded text-xs bg-blue-600 text-white hover:bg-blue-500 transition-colors"
          >Sort!</button>
        </div>

        <!-- Canvas visualization -->
        <div
          ref="vizContainer"
          class="relative rounded-lg overflow-hidden border flex-shrink-0"
          :class="isDark ? 'bg-gray-900 border-gray-800' : 'bg-gray-100 border-gray-300'"
          :style="{ height: raceMode ? 'auto' : '280px' }"
        >
          <!-- Normal mode -->
          <canvas
            v-if="!raceMode"
            ref="canvas"
            class="w-full h-full"
            style="height: 280px"
          />

          <!-- Race mode: two canvases -->
          <div v-if="raceMode" class="flex gap-2 p-2" style="height: 300px">
            <div class="flex-1 flex flex-col gap-1">
              <div class="flex items-center justify-between">
                <span class="text-xs font-bold" :class="raceWinner === 1 ? 'text-yellow-400' : isDark ? 'text-blue-400' : 'text-blue-600'">
                  {{ raceWinner === 1 ? '🏆 ' : '' }}{{ algorithmMap.get(raceAlgo1)?.name }}
                </span>
                <span class="text-xs" :class="isDark ? 'text-gray-500' : 'text-gray-400'">
                  {{ raceStep1?.comparisons ?? 0 }} cmp / {{ raceStep1?.swaps ?? 0 }} swp
                </span>
              </div>
              <div
                id="race-panel-1"
                class="flex-1 rounded overflow-hidden border"
                :class="isDark ? 'bg-gray-950 border-gray-700' : 'bg-gray-50 border-gray-300'"
              >
                <canvas id="race-canvas-1" ref="raceCanvas1" class="w-full h-full" />
              </div>
              <div class="h-1 rounded-full overflow-hidden" :class="isDark ? 'bg-gray-800' : 'bg-gray-300'">
                <div class="h-full bg-blue-500 transition-all" :style="{ width: raceProg1 + '%' }" />
              </div>
            </div>
            <div class="flex-1 flex flex-col gap-1">
              <div class="flex items-center justify-between">
                <span class="text-xs font-bold" :class="raceWinner === 2 ? 'text-yellow-400' : isDark ? 'text-orange-400' : 'text-orange-600'">
                  {{ raceWinner === 2 ? '🏆 ' : '' }}{{ algorithmMap.get(raceAlgo2)?.name }}
                </span>
                <span class="text-xs" :class="isDark ? 'text-gray-500' : 'text-gray-400'">
                  {{ raceStep2?.comparisons ?? 0 }} cmp / {{ raceStep2?.swaps ?? 0 }} swp
                </span>
              </div>
              <div
                id="race-panel-2"
                class="flex-1 rounded overflow-hidden border"
                :class="isDark ? 'bg-gray-950 border-gray-700' : 'bg-gray-50 border-gray-300'"
              >
                <canvas id="race-canvas-2" ref="raceCanvas2" class="w-full h-full" />
              </div>
              <div class="h-1 rounded-full overflow-hidden" :class="isDark ? 'bg-gray-800' : 'bg-gray-300'">
                <div class="h-full bg-orange-500 transition-all" :style="{ width: raceProg2 + '%' }" />
              </div>
            </div>
          </div>
        </div>

        <!-- Controls -->
        <div class="flex flex-wrap items-center gap-2">
          <div class="flex gap-1">
            <button
              v-if="!raceMode"
              @click="stepBack"
              :disabled="currentStepIndex === 0"
              class="px-2 py-1 rounded text-base transition-colors disabled:opacity-30"
              :class="isDark ? 'bg-gray-800 hover:bg-gray-700' : 'bg-gray-200 hover:bg-gray-300'"
              title="Bước trước"
            >⏮</button>
            <button
              @click="togglePlay"
              class="px-4 py-1 rounded text-base font-bold transition-colors"
              :class="isPlaying
                ? 'bg-yellow-500 text-black hover:bg-yellow-400'
                : isDark ? 'bg-blue-600 text-white hover:bg-blue-500' : 'bg-blue-500 text-white hover:bg-blue-400'"
            >{{ isPlaying ? '⏸ Dừng' : isDone ? '↺ Chạy lại' : '▶ Chạy' }}</button>
            <button
              @click="stepForward"
              class="px-2 py-1 rounded text-base transition-colors"
              :class="isDark ? 'bg-gray-800 hover:bg-gray-700' : 'bg-gray-200 hover:bg-gray-300'"
              title="Bước tiếp"
            >⏭</button>
          </div>

          <button
            @click="raceMode ? initRace() : shuffle()"
            class="px-3 py-1 rounded text-xs transition-colors"
            :class="isDark ? 'bg-gray-800 text-gray-300 hover:bg-gray-700' : 'bg-gray-100 text-gray-600 hover:bg-gray-200'"
          >🔀 Mảng mới</button>

          <button
            v-if="!raceMode"
            @click="initSort"
            class="px-3 py-1 rounded text-xs transition-colors"
            :class="isDark ? 'bg-gray-800 text-gray-300 hover:bg-gray-700' : 'bg-gray-100 text-gray-600 hover:bg-gray-200'"
          >↺ Reset</button>

          <div class="flex items-center gap-2 ml-auto">
            <span class="text-xs" :class="isDark ? 'text-gray-500' : 'text-gray-400'">Tốc độ:</span>
            <input type="range" v-model.number="speed" min="1" max="10" class="w-20 accent-blue-500" />
            <span class="text-xs w-16" :class="isDark ? 'text-gray-400' : 'text-gray-600'">{{ speedLabel }}</span>
          </div>

          <div v-if="!raceMode" class="flex items-center gap-2">
            <span class="text-xs" :class="isDark ? 'text-gray-500' : 'text-gray-400'">
              {{ currentAlgoId === 'bogo' ? 'Phần tử (≤8):' : 'Số phần tử:' }}
            </span>
            <input
              type="range"
              v-model.number="arraySize"
              :min="5"
              :max="currentAlgoId === 'bogo' ? 8 : currentAlgoId === 'counting' || currentAlgoId === 'radix' ? 150 : 200"
              class="w-20 accent-blue-500"
            />
            <span class="text-xs w-8" :class="isDark ? 'text-gray-400' : 'text-gray-600'">
              {{ currentAlgoId === 'bogo' ? Math.min(arraySize, 8) : arraySize }}
            </span>
          </div>
        </div>

        <!-- Stats (non-race) -->
        <div v-if="!raceMode" class="grid grid-cols-2 sm:grid-cols-4 gap-2">
          <div
            class="rounded px-3 py-1.5 text-center"
            :class="isDark ? 'bg-gray-800' : 'bg-gray-200'"
          >
            <div class="text-xs" :class="isDark ? 'text-gray-500' : 'text-gray-500'">So sánh</div>
            <div class="text-base font-bold text-yellow-400">{{ currentStep.comparisons.toLocaleString() }}</div>
          </div>
          <div
            class="rounded px-3 py-1.5 text-center"
            :class="isDark ? 'bg-gray-800' : 'bg-gray-200'"
          >
            <div class="text-xs" :class="isDark ? 'text-gray-500' : 'text-gray-500'">Hoán đổi</div>
            <div class="text-base font-bold text-red-400">{{ currentStep.swaps.toLocaleString() }}</div>
          </div>
          <div
            class="rounded px-3 py-1.5 text-center"
            :class="isDark ? 'bg-gray-800' : 'bg-gray-200'"
          >
            <div class="text-xs" :class="isDark ? 'text-gray-500' : 'text-gray-500'">Thời gian</div>
            <div class="text-base font-bold text-green-400">{{ formatMs(elapsedMs) }}</div>
          </div>
          <div
            class="rounded px-3 py-1.5 text-center"
            :class="isDark ? 'bg-gray-800' : 'bg-gray-200'"
          >
            <div class="text-xs" :class="isDark ? 'text-gray-500' : 'text-gray-500'">Bước</div>
            <div class="text-base font-bold text-blue-400">{{ currentStepIndex }}/{{ steps.length }}</div>
          </div>
        </div>

        <!-- Progress bar (non-race) -->
        <div v-if="!raceMode" class="relative h-1.5 rounded-full overflow-hidden" :class="isDark ? 'bg-gray-800' : 'bg-gray-200'">
          <div
            class="h-full rounded-full transition-all duration-100"
            :class="isDone ? 'bg-green-500' : 'bg-blue-500'"
            :style="{ width: progress + '%' }"
          />
        </div>

        <!-- Color legend -->
        <div class="flex flex-wrap gap-3 text-xs" :class="isDark ? 'text-gray-500' : 'text-gray-500'">
          <span><span class="inline-block w-3 h-3 rounded-sm bg-blue-400 mr-1" />Chưa xét</span>
          <span><span class="inline-block w-3 h-3 rounded-sm bg-yellow-400 mr-1" />Đang so sánh</span>
          <span><span class="inline-block w-3 h-3 rounded-sm bg-red-400 mr-1" />Đang swap</span>
          <span><span class="inline-block w-3 h-3 rounded-sm bg-green-400 mr-1" />Đã sắp xếp</span>
          <span v-if="currentAlgoId === 'quick'"><span class="inline-block w-3 h-3 rounded-sm bg-purple-400 mr-1" />Pivot</span>
        </div>
      </div>

      <!-- Right: Learn panel -->
      <div
        v-if="showLearn && !raceMode"
        class="w-72 xl:w-80 border-l flex flex-col gap-3 p-3 overflow-y-auto hidden md:flex"
        :class="isDark ? 'border-gray-800 bg-gray-900/30' : 'border-gray-200 bg-gray-50'"
        style="max-height: calc(100vh - 100px)"
      >
        <!-- Step description -->
        <div
          class="rounded-lg p-3 border min-h-12"
          :class="isDark ? 'bg-gray-800 border-gray-700' : 'bg-white border-gray-200'"
        >
          <div class="text-xs font-bold mb-1 text-blue-400">💬 Bước hiện tại</div>
          <div class="text-xs leading-relaxed" :class="isDark ? 'text-gray-300' : 'text-gray-700'">
            {{ currentStep.description || 'Nhấn ▶ để bắt đầu' }}
          </div>
        </div>

        <!-- Pseudocode -->
        <div
          class="rounded-lg border overflow-hidden"
          :class="isDark ? 'border-gray-700' : 'border-gray-200'"
        >
          <div
            class="px-3 py-1.5 text-xs font-bold border-b"
            :class="isDark ? 'bg-gray-800 border-gray-700 text-gray-400' : 'bg-gray-100 border-gray-200 text-gray-600'"
          >📝 Pseudo-code</div>
          <div :class="isDark ? 'bg-gray-950' : 'bg-white'">
            <div
              v-for="(line, idx) in currentAlgo.pseudoCode"
              :key="idx"
              class="px-3 py-0.5 text-xs font-mono transition-colors"
              :class="[
                currentStep.pseudoCodeLine === idx
                  ? isDark ? 'bg-blue-900/60 text-blue-200 border-l-2 border-blue-400' : 'bg-blue-50 text-blue-800 border-l-2 border-blue-400'
                  : isDark ? 'text-gray-500' : 'text-gray-400',
                line === '' ? 'py-0' : ''
              ]"
            >{{ line || '\u00a0' }}</div>
          </div>
        </div>

        <!-- Algorithm info -->
        <div
          class="rounded-lg p-3 border"
          :class="isDark ? 'bg-gray-800 border-gray-700' : 'bg-white border-gray-200'"
        >
          <div class="text-xs font-bold mb-2 text-purple-400">ℹ️ {{ currentAlgo.name }}</div>
          <p class="text-xs leading-relaxed mb-2" :class="isDark ? 'text-gray-400' : 'text-gray-600'">
            {{ currentAlgo.description }}
          </p>
          <div class="grid grid-cols-2 gap-1 text-xs">
            <div :class="isDark ? 'text-gray-500' : 'text-gray-400'">Best:</div>
            <div class="text-green-400 font-mono">{{ currentAlgo.timeComplexity.best }}</div>
            <div :class="isDark ? 'text-gray-500' : 'text-gray-400'">Average:</div>
            <div class="text-yellow-400 font-mono">{{ currentAlgo.timeComplexity.average }}</div>
            <div :class="isDark ? 'text-gray-500' : 'text-gray-400'">Worst:</div>
            <div class="text-red-400 font-mono">{{ currentAlgo.timeComplexity.worst }}</div>
            <div :class="isDark ? 'text-gray-500' : 'text-gray-400'">Space:</div>
            <div class="text-blue-400 font-mono">{{ currentAlgo.spaceComplexity }}</div>
            <div :class="isDark ? 'text-gray-500' : 'text-gray-400'">Ổn định:</div>
            <div :class="currentAlgo.stable ? 'text-green-400' : 'text-red-400'">
              {{ currentAlgo.stable ? '✓ Có' : '✗ Không' }}
            </div>
          </div>
        </div>

        <!-- All algorithms complexity table -->
        <div
          class="rounded-lg border overflow-hidden"
          :class="isDark ? 'border-gray-700' : 'border-gray-200'"
        >
          <div
            class="px-3 py-1.5 text-xs font-bold border-b"
            :class="isDark ? 'bg-gray-800 border-gray-700 text-gray-400' : 'bg-gray-100 border-gray-200 text-gray-600'"
          >📊 So sánh độ phức tạp</div>
          <div class="overflow-x-auto">
            <table class="w-full text-xs">
              <thead>
                <tr :class="isDark ? 'bg-gray-800 text-gray-500' : 'bg-gray-50 text-gray-400'">
                  <th class="px-2 py-1 text-left font-normal">Thuật toán</th>
                  <th class="px-2 py-1 text-right font-normal">Avg</th>
                  <th class="px-2 py-1 text-right font-normal">Space</th>
                </tr>
              </thead>
              <tbody>
                <tr
                  v-for="algo in algorithms"
                  :key="algo.id"
                  class="border-t transition-colors cursor-pointer"
                  :class="[
                    isDark ? 'border-gray-800' : 'border-gray-100',
                    currentAlgoId === algo.id
                      ? isDark ? 'bg-blue-900/30 text-blue-300' : 'bg-blue-50 text-blue-700'
                      : isDark ? 'text-gray-400 hover:bg-gray-800' : 'text-gray-500 hover:bg-gray-50'
                  ]"
                  @click="currentAlgoId = algo.id"
                >
                  <td class="px-2 py-0.5">{{ algo.name }}</td>
                  <td class="px-2 py-0.5 text-right font-mono text-yellow-400/80">{{ algo.timeComplexity.average }}</td>
                  <td class="px-2 py-0.5 text-right font-mono text-blue-400/80">{{ algo.spaceComplexity }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>

    <!-- Mobile learn panel -->
    <div
      v-if="showLearn && !raceMode"
      class="md:hidden border-t p-3 space-y-3"
      :class="isDark ? 'border-gray-800 bg-gray-900/30' : 'border-gray-200'"
    >
      <div class="text-xs font-bold text-blue-400">💬 {{ currentStep.description || 'Nhấn ▶ để bắt đầu' }}</div>
      <div
        class="rounded p-2 border text-xs"
        :class="isDark ? 'bg-gray-950 border-gray-700' : 'bg-white border-gray-200'"
      >
        <div class="font-bold mb-1" :class="isDark ? 'text-gray-400' : 'text-gray-600'">{{ currentAlgo.name }}</div>
        <div class="grid grid-cols-2 gap-x-4 gap-y-0.5" :class="isDark ? 'text-gray-500' : 'text-gray-400'">
          <span>Avg: <span class="text-yellow-400">{{ currentAlgo.timeComplexity.average }}</span></span>
          <span>Space: <span class="text-blue-400">{{ currentAlgo.spaceComplexity }}</span></span>
          <span>Ổn định: <span :class="currentAlgo.stable ? 'text-green-400' : 'text-red-400'">{{ currentAlgo.stable ? 'Có' : 'Không' }}</span></span>
        </div>
      </div>
    </div>
  </div>
</template>
