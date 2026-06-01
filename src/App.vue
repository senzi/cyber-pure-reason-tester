<script setup>
import { computed, onMounted, ref } from 'vue'

const HISTORY_KEY = 'cyber_reason_history_records'
const ANSWER_LETTERS = ['A', 'B', 'C', 'D']
const MIN_QUESTION_COUNT = 10

const view = ref('home')
const loading = ref(true)
const loadError = ref('')
const starting = ref(false)
const questionManifest = ref([])
const requestedCount = ref(10)
const activeQuestions = ref([])
const currentIndex = ref(0)
const selectedAnswer = ref('')
const answers = ref([])
const startTime = ref(null)
const endTime = ref(null)
const resultRecord = ref(null)
const reviewOpen = ref(false)
const historyOpen = ref(false)
const historyRecords = ref([])

onMounted(async () => {
  try {
    const response = await fetch('/questions/manifest.json')
    if (!response.ok) {
      throw new Error(`题库清单加载失败：${response.status}`)
    }
    questionManifest.value = await response.json()
    requestedCount.value = Math.min(10, questionManifest.value.length)
  } catch (error) {
    loadError.value = error.message || '题库清单加载失败'
  } finally {
    loading.value = false
  }
})

const totalAvailable = computed(() => questionManifest.value.length)
const maxQuestionCount = computed(() => Math.min(172, totalAvailable.value || 172))
const currentQuestion = computed(() => activeQuestions.value[currentIndex.value])
const progressPercent = computed(() => {
  if (!activeQuestions.value.length) return 0
  return ((currentIndex.value + 1) / activeQuestions.value.length) * 100
})
const canProceed = computed(() => Boolean(selectedAnswer.value))
const correctCount = computed(() => answers.value.filter((answer) => answer.isCorrect).length)
const wrongAnswers = computed(() => answers.value.filter((answer) => !answer.isCorrect))
const questionPresets = computed(() => {
  const presets = [5, 10, 20, 50, maxQuestionCount.value]
  return [...new Set(presets.filter((value) => value >= MIN_QUESTION_COUNT && value <= maxQuestionCount.value))]
})

function clampQuestionCount() {
  const parsed = Number(requestedCount.value)
  if (!Number.isFinite(parsed)) {
    requestedCount.value = 10
    return
  }
  requestedCount.value = Math.max(
    MIN_QUESTION_COUNT,
    Math.min(maxQuestionCount.value, Math.round(parsed)),
  )
}

function adjustQuestionCount(delta) {
  requestedCount.value = Number(requestedCount.value) + delta
  clampQuestionCount()
}

function setQuestionCount(count) {
  requestedCount.value = count
  clampQuestionCount()
}

function shuffle(items) {
  const cloned = [...items]
  for (let index = cloned.length - 1; index > 0; index -= 1) {
    const swapIndex = Math.floor(Math.random() * (index + 1))
    ;[cloned[index], cloned[swapIndex]] = [cloned[swapIndex], cloned[index]]
  }
  return cloned
}

function normalizeQuestion(question) {
  return {
    id: question.id,
    prompt: question.question_text,
    type: question.question_type === '判断题' ? 'boolean' : 'choice',
    options: Object.entries(question.options).map(([key, text]) => ({ key, text })),
    secret_h: question.secret_h,
    secret_l: question.secret_l,
  }
}

async function fetchQuestion(entry) {
  const response = await fetch(`/questions/${entry.file}`)
  if (!response.ok) {
    throw new Error(`题目 ${entry.id} 加载失败：${response.status}`)
  }
  return normalizeQuestion(await response.json())
}

function xorBitFromHex(hexString) {
  return hexString
    .toLowerCase()
    .split('')
    .reduce((bit, char) => bit ^ (parseInt(char, 16).toString(2).replaceAll('0', '').length % 2), 0)
}

function decryptAnswer(question) {
  const bitH = xorBitFromHex(question.secret_h)
  const bitL = xorBitFromHex(question.secret_l)
  return ANSWER_LETTERS[(bitH << 1) | bitL]
}

function estimateBValue(scoreRate) {
  return 22 * Math.pow(scoreRate, 2) + 5 * scoreRate
}

function formatBValue(value) {
  return Number(value.toFixed(8)).toString()
}

function formatDateTime(date) {
  const pad = (value) => String(value).padStart(2, '0')
  return [
    date.getFullYear(),
    pad(date.getMonth() + 1),
    pad(date.getDate()),
  ].join('-') + ` ${pad(date.getHours())}:${pad(date.getMinutes())}:${pad(date.getSeconds())}`
}

function getPortrait(scoreRate) {
  const percent = scoreRate * 100
  if (percent <= 15) return '🤯 逻辑重度反杀（完美的反向指标，0.5B以下水准）'
  if (percent <= 30) return '🪵 盲猜混沌层（思维基本被德味古典黑话完全冲散）'
  if (percent <= 50) return '⚠️ 直觉型答题（容易陷入否定前件等二义性漏洞）'
  if (percent <= 75) return '🥉 边缘轻量智能（具备基本的基础符号拆解力）'
  if (percent <= 88) return '🥈 资深严谨学者（逼近中小型端侧模型天花板）'
  if (percent <= 96) return '🥇 纯粹理性批判（具备极高密度的逻辑防御直觉）'
  return '👁️ 终极逻辑先知（全对通关，27B级无情纯符号数理推由机器）'
}

function persistResult(record) {
  const previous = JSON.parse(localStorage.getItem(HISTORY_KEY) || '[]')
  localStorage.setItem(HISTORY_KEY, JSON.stringify([...previous, record]))
}

function readHistoryRecords() {
  try {
    const records = JSON.parse(localStorage.getItem(HISTORY_KEY) || '[]')
    historyRecords.value = Array.isArray(records) ? records.slice().reverse() : []
  } catch {
    historyRecords.value = []
  }
}

function openHistory() {
  readHistoryRecords()
  historyOpen.value = true
}

async function startQuiz() {
  clampQuestionCount()
  const total = requestedCount.value
  const selectedEntries = shuffle(questionManifest.value).slice(0, total)
  starting.value = true
  loadError.value = ''
  try {
    activeQuestions.value = await Promise.all(selectedEntries.map(fetchQuestion))
  } catch (error) {
    loadError.value = error.message || '抽题加载失败'
    activeQuestions.value = []
    starting.value = false
    return
  }
  currentIndex.value = 0
  selectedAnswer.value = ''
  answers.value = []
  resultRecord.value = null
  reviewOpen.value = false
  historyOpen.value = false
  startTime.value = new Date()
  endTime.value = null
  view.value = 'quiz'
  starting.value = false
}

function submitCurrentAnswer() {
  if (!selectedAnswer.value || !currentQuestion.value) return

  const correctLetter = decryptAnswer(currentQuestion.value)
  const selectedOption = currentQuestion.value.options.find((option) => option.key === selectedAnswer.value)
  const correctOption = currentQuestion.value.options.find((option) => option.key === correctLetter)

  answers.value.push({
    question: currentQuestion.value,
    selectedLetter: selectedAnswer.value,
    selectedText: selectedOption?.text || '',
    correctLetter,
    correctText: correctOption?.text || '',
    isCorrect: selectedAnswer.value === correctLetter,
  })

  if (currentIndex.value === activeQuestions.value.length - 1) {
    finishQuiz()
    return
  }

  currentIndex.value += 1
  selectedAnswer.value = ''
}

function finishQuiz() {
  endTime.value = new Date()
  const totalQuestions = activeQuestions.value.length
  const scoreRate = correctCount.value / totalQuestions
  const equivalentBValue = Number(estimateBValue(scoreRate).toFixed(8))
  const durationSeconds = Math.max(0, Math.round((endTime.value.getTime() - startTime.value.getTime()) / 1000))

  resultRecord.value = {
    start_time: formatDateTime(startTime.value),
    end_time: formatDateTime(endTime.value),
    duration_seconds: durationSeconds,
    total_questions: totalQuestions,
    correct_count: correctCount.value,
    score_rate: Number(scoreRate.toFixed(8)),
    equivalent_b_value: equivalentBValue,
  }

  persistResult(resultRecord.value)
  view.value = 'result'
}

function resetToHome() {
  view.value = 'home'
  activeQuestions.value = []
  currentIndex.value = 0
  selectedAnswer.value = ''
  answers.value = []
  resultRecord.value = null
  reviewOpen.value = false
  historyOpen.value = false
}
</script>

<template>
  <main class="app-shell">
    <section v-if="view === 'home'" class="home-view">
      <div class="brand-strip">
        <span>Local-first</span>
        <span>XOR sealed</span>
        <span>Scaling law</span>
      </div>

      <h1>逻辑抗性评测</h1>
      <p class="intro">
        此装置不召唤经验世界的偶然常识，也不宽恕语词缝隙中的二义性遁逃。它将题干化为范畴的压力场，
        令每一次判断都暴露为代数推导的可测痕迹，并在最后把你的思维抗性投影到冷峻的大模型尺度之上。
      </p>

      <div class="control-panel">
        <label for="question-count">本轮拷问题数</label>
        <div class="count-picker">
          <button
            class="step-button"
            :disabled="loading || starting || Boolean(loadError) || requestedCount <= MIN_QUESTION_COUNT"
            aria-label="减少题数"
            @click="adjustQuestionCount(-1)"
          >
            −
          </button>
          <input
            id="question-count"
            v-model.number="requestedCount"
            type="number"
            :min="MIN_QUESTION_COUNT"
            :max="maxQuestionCount"
            :disabled="loading || starting || Boolean(loadError)"
            @blur="clampQuestionCount"
          />
          <button
            class="step-button"
            :disabled="loading || starting || Boolean(loadError) || requestedCount >= maxQuestionCount"
            aria-label="增加题数"
            @click="adjustQuestionCount(1)"
          >
            +
          </button>
        </div>
        <div class="preset-row" aria-label="题数预设">
          <button
            v-for="preset in questionPresets"
            :key="preset"
            class="preset-button"
            :class="{ active: requestedCount === preset }"
            :disabled="loading || starting || Boolean(loadError)"
            @click="setQuestionCount(preset)"
          >
            {{ preset === maxQuestionCount ? '全量' : preset }}
          </button>
        </div>
        <div class="count-slider">
          <input
            v-model.number="requestedCount"
            type="range"
            :min="MIN_QUESTION_COUNT"
            :max="maxQuestionCount"
            :disabled="loading || starting || Boolean(loadError)"
            @change="clampQuestionCount"
          />
        </div>
        <p class="capacity">题库上限：{{ totalAvailable || 172 }} 题，最少 10 题，默认初始值：10 题</p>
      </div>

      <p v-if="loading" class="status-text">题库索引载入中...</p>
      <p v-else-if="starting" class="status-text">正在随机抽取并加载本轮题目...</p>
      <p v-else-if="loadError" class="status-text error">{{ loadError }}</p>
      <div class="home-actions">
        <button class="primary-action" :disabled="loading || starting || Boolean(loadError)" @click="startQuiz">
          {{ starting ? '抽取范畴中...' : '激活纯粹理性批判机制' }}
        </button>
        <button class="secondary-action" @click="openHistory">查看历史记录</button>
      </div>
    </section>

    <section v-else-if="view === 'quiz' && currentQuestion" class="quiz-view">
      <header class="quiz-header">
        <p>当前拷问进度：{{ currentIndex + 1 }} / {{ activeQuestions.length }} 题</p>
        <div class="progress-track" aria-hidden="true">
          <div class="progress-fill" :style="{ width: `${progressPercent}%` }"></div>
        </div>
      </header>

      <article class="question-stage">
        <p class="question-type">{{ currentQuestion.type === 'boolean' ? '二值判断范畴' : '四项单选范畴' }}</p>
        <h2>{{ currentQuestion.prompt }}</h2>
      </article>

      <div class="option-grid" :class="{ compact: currentQuestion.options.length === 2 }">
        <button
          v-for="option in currentQuestion.options"
          :key="option.key"
          class="option-card"
          :class="{ selected: selectedAnswer === option.key }"
          @click="selectedAnswer = option.key"
        >
          <span>{{ option.key }}</span>
          <strong>{{ option.text }}</strong>
        </button>
      </div>

      <button class="primary-action next-action" :disabled="!canProceed" @click="submitCurrentAnswer">
        {{ currentIndex === activeQuestions.length - 1 ? '进入绝对理性清算' : '挺进下一思辨范畴' }}
      </button>
    </section>

    <section v-else-if="view === 'result' && resultRecord" class="result-view">
      <p class="result-kicker">绝对理性清算页</p>
      <h2>逻辑平均得分率: {{ (resultRecord.score_rate * 100).toFixed(2) }}%</h2>
      <p class="b-value">
        你的大脑逻辑抗性相当于 {{ formatBValue(resultRecord.equivalent_b_value) }} b 的大模型
      </p>
      <p class="portrait">{{ getPortrait(resultRecord.score_rate) }}</p>

      <dl class="ledger">
        <div>
          <dt>审计耗时</dt>
          <dd>{{ resultRecord.duration_seconds }} 秒</dd>
        </div>
        <div>
          <dt>正确题数</dt>
          <dd>{{ resultRecord.correct_count }} / {{ resultRecord.total_questions }}</dd>
        </div>
      </dl>

      <div class="result-actions">
        <button class="secondary-action" @click="resetToHome">重返先验首页</button>
        <button class="secondary-action" @click="openHistory">查看历史记录</button>
        <button class="primary-action" @click="reviewOpen = true">审视坏病例（查看错题）</button>
      </div>
    </section>

    <div v-if="historyOpen" class="modal-backdrop" @click.self="historyOpen = false">
      <section class="history-modal" role="dialog" aria-modal="true" aria-label="历史记录">
        <header>
          <p>本地历史账本</p>
          <h2>历史记录：{{ historyRecords.length }} 次</h2>
        </header>

        <div v-if="historyRecords.length" class="history-list">
          <article v-for="(record, index) in historyRecords" :key="`${record.start_time}-${index}`" class="history-card">
            <div class="history-card-head">
              <strong>{{ record.start_time }}</strong>
              <span>{{ (record.score_rate * 100).toFixed(2) }}%</span>
            </div>
            <dl>
              <div>
                <dt>完成时间</dt>
                <dd>{{ record.end_time }}</dd>
              </div>
              <div>
                <dt>题目</dt>
                <dd>{{ record.correct_count }} / {{ record.total_questions }}</dd>
              </div>
              <div>
                <dt>耗时</dt>
                <dd>{{ record.duration_seconds }} 秒</dd>
              </div>
              <div>
                <dt>等效 B 数</dt>
                <dd>{{ formatBValue(record.equivalent_b_value) }} b</dd>
              </div>
            </dl>
          </article>
        </div>
        <p v-else class="perfect-text">尚无历史记录。完成一次评测后，本地账本会自动写入。</p>

        <button class="secondary-action close-action" @click="historyOpen = false">关闭历史</button>
      </section>
    </div>

    <div v-if="reviewOpen" class="modal-backdrop" @click.self="reviewOpen = false">
      <section class="review-modal" role="dialog" aria-modal="true" aria-label="错题审视">
        <header>
          <p>坏病例审视</p>
          <h2>本轮错误样本：{{ wrongAnswers.length }} 题</h2>
        </header>

        <div v-if="wrongAnswers.length" class="wrong-list">
          <article v-for="answer in wrongAnswers" :key="answer.question.id" class="wrong-card">
            <h3>【原问题干】</h3>
            <p>{{ answer.question.prompt }}</p>
            <h3>【你选择的偏见】</h3>
            <p>{{ answer.selectedLetter }}. {{ answer.selectedText }}</p>
            <h3>【绝对正确理性】</h3>
            <p>{{ answer.correctLetter }}. {{ answer.correctText }}</p>
          </article>
        </div>
        <p v-else class="perfect-text">没有坏病例。本轮推导未留下可审视的裂隙。</p>

        <button class="secondary-action close-action" @click="reviewOpen = false">关闭审视</button>
      </section>
    </div>
  </main>
</template>
