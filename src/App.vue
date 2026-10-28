<script setup>
import { computed, nextTick, onBeforeUnmount, reactive, ref } from 'vue'
import '@fontsource/roboto-mono/latin-400.css'
import '@fontsource/roboto-mono/cyrillic-400.css'

const passages = [
  'Задача организации, в особенности же рамки и место обучения кадров в значительной степени обуславливает создание направлений прогрессивного развития. Таким образом постоянное информационное обеспечение нашей деятельности играет важную роль в формировании новых предложений. С другой стороны постоянный количественный рост позволяет оценить значение дальнейшей работы.',
  'Раннее утро наполняет комнату мягким светом. За окном просыпается город, а на столе уже остывает чашка чая. В такие минуты особенно приятно открыть книгу и провести немного времени в тишине. Каждый новый день приносит маленькие открытия и возможность попробовать что-нибудь интересное.',
  'Дорога уходила далеко за горизонт. По обе стороны тянулись зелёные поля, и лёгкий ветер шевелил высокую траву. Мы решили идти дальше, чтобы увидеть озеро среди старых деревьев. Иногда самое удивительное путешествие начинается с простого желания выйти из дома.',
  'Умение быстро печатать приходит с практикой. Не стоит торопиться: сначала найдите удобный ритм и старайтесь попадать по нужным клавишам. Постепенно движения станут увереннее, а ошибок будет меньше. Главное — заниматься регулярно и получать удовольствие от собственного прогресса.',
  'Вечером небольшой книжный магазин превращается в особенное место. Тёплые лампы освещают полки, тихо шелестят страницы, и каждый посетитель выбирает свою историю. Здесь можно найти рассказ о далёких странах, научиться новому делу или просто отдохнуть от повседневной суеты.',
  'Хорошие идеи часто появляются неожиданно. Во время прогулки, разговора с другом или работы над обычной задачей мы замечаем то, что раньше ускользало от внимания. Записывайте мысли, задавайте вопросы и оставляйте место для экспериментов. Любопытство помогает увидеть знакомый мир по-новому.',
]

const mode = ref('words')
const unit = ref('words')
const duration = ref(60)
const wordCount = ref(50)
const characterCount = ref(250)
const screen = ref('start')
const input = ref(null)
const field = ref(null)
const focused = ref(false)
const customOpen = ref(false)
const customValue = ref(50)
const customUnit = ref('words')

const session = reactive({
  words: [],
  passageIndex: -1,
  raw: '',
  startedAt: null,
  elapsed: 0,
  attempts: 0,
  correctAttempts: 0,
})
let timer = null

const amount = computed(() => {
  if (mode.value === 'time') return duration.value
  if (unit.value === 'characters') return characterCount.value
  return wordCount.value
})

const presets = computed(() => {
  if (mode.value === 'time') return [15, 30, 60, 120]
  if (unit.value === 'characters') return [100, 250, 500, 1000]
  return [10, 25, 50, 100]
})

const amountLabel = computed(() => {
  if (mode.value === 'time') return 'секунд'
  if (unit.value === 'characters') return 'символов'
  return 'слов'
})

const state = computed(() => evaluateInput(session.raw, session.words))
const remaining = computed(() =>
  Math.max(0, Math.ceil(duration.value - session.elapsed)),
)
const running = computed(
  () => session.startedAt !== null && screen.value === 'test',
)
const customLimit = computed(() => (mode.value === 'time' ? 3600 : 5000))
const customValid = computed(() => {
  const value = Number(customValue.value)
  return Number.isInteger(value) && value >= 1 && value <= customLimit.value
})
const progress = computed(() => {
  if (mode.value === 'time') return remaining.value
  if (unit.value === 'characters') {
    const active = state.value.active
    const completed = session.words
      .slice(0, active)
      .reduce((sum, word) => sum + word.length + 1, 0)
    const current = Math.min(
      state.value.entries[active]?.value.length || 0,
      session.words[active]?.length || 0,
    )
    return `${completed + current} / ${characterCount.value}`
  }
  return `${state.value.active + (state.value.complete ? 1 : 0)} / ${session.words.length}`
})
const result = computed(() => ({
  cpm: Math.round((state.value.correct / Math.max(session.elapsed, 1)) * 60),
  accuracy: session.attempts
    ? Math.round((session.correctAttempts / session.attempts) * 100)
    : 0,
  characters: session.raw.length,
  errors: state.value.errors,
}))

function generateWords(count, textUnit = 'words', passageIndex = 0) {
  let text = passages[passageIndex]
  let index = passageIndex

  while (
    (textUnit === 'characters' ? text.length : text.split(' ').length) < count
  ) {
    index = (index + 1) % passages.length
    text += ' ' + passages[index]
  }

  if (textUnit === 'characters') {
    // Сохраняем точное число символов без пустого последнего слова.
    return text.slice(0, count).replace(/ $/, '.').split(' ')
  }
  return text.split(' ').slice(0, count)
}

function evaluateInput(raw, words) {
  const entries = words.map(() => ({
    value: '',
    extraSpaces: 0,
    submitted: false,
  }))
  let active = 0
  let correct = 0
  let errors = 0

  for (const char of raw) {
    const entry = entries[active]
    if (!entry) break

    if (char === ' ') {
      if (!entry.value || active === words.length - 1) {
        entry.extraSpaces++
        errors++
      } else {
        errors += Math.max(0, words[active].length - entry.value.length)
        entry.submitted = true
        correct++
        active++
      }
    } else {
      if (char === words[active][entry.value.length]) correct++
      else errors++
      entry.value += char
    }
  }

  const complete =
    words.length > 0 &&
    active === words.length - 1 &&
    entries[active].value.length >= words[active].length
  return { entries, active, correct, errors, complete }
}

function setAmount(value) {
  if (mode.value === 'time') duration.value = value
  else if (unit.value === 'characters') characterCount.value = value
  else wordCount.value = value

  if (screen.value === 'test') start()
}

function changeMode(value) {
  mode.value = value
  if (screen.value === 'test') start()
}

function openCustom() {
  customValue.value = amount.value
  customUnit.value = unit.value
  customOpen.value = true
}

function applyCustom() {
  if (!customValid.value) return
  unit.value = customUnit.value
  setAmount(Number(customValue.value))
  customOpen.value = false
}

async function focusInput() {
  await nextTick()
  input.value?.focus({ preventScroll: true })
  input.value?.setSelectionRange(session.raw.length, session.raw.length)
}

function stopTimer() {
  clearInterval(timer)
  timer = null
}

function start(sameText = false) {
  stopTimer()

  if (!sameText || session.words.length === 0) {
    const offset = 1 + Math.floor(Math.random() * (passages.length - 1))
    session.passageIndex = (session.passageIndex + offset) % passages.length
    const count = mode.value === 'time' ? 180 : amount.value
    const textUnit = mode.value === 'time' ? 'words' : unit.value
    session.words = generateWords(count, textUnit, session.passageIndex)
  }

  session.raw = ''
  session.startedAt = null
  session.elapsed = 0
  session.attempts = 0
  session.correctAttempts = 0
  screen.value = 'test'
  nextTick(() => {
    if (field.value) field.value.scrollTop = 0
  })
  focusInput()
}

function finish() {
  stopTimer()
  screen.value = 'result'
}

function updateTimer() {
  if (session.startedAt === null || screen.value !== 'test') return
  session.elapsed = (Date.now() - session.startedAt) / 1000

  if (mode.value === 'time' && session.elapsed >= duration.value) {
    session.elapsed = duration.value
    finish()
  }
}

async function onInput(event) {
  if (event.isComposing) return
  updateTimer()
  if (screen.value !== 'test') return

  const value = event.target.value.replace(/[\n\r\t]/g, '')

  if (value && session.startedAt === null) {
    session.startedAt = Date.now()
    timer = setInterval(updateTimer, 100)
  }

  let common = 0
  while (
    common < session.raw.length &&
    common < value.length &&
    session.raw[common] === value[common]
  ) {
    common++
  }

  let before = evaluateInput(value.slice(0, common), session.words)
  for (let index = common; index < value.length; index++) {
    const after = evaluateInput(value.slice(0, index + 1), session.words)
    session.attempts++
    if (after.correct > before.correct) session.correctAttempts++
    if (after.active > before.active) {
      session.attempts += Math.max(
        0,
        session.words[before.active].length -
        before.entries[before.active].value.length,
      )
    }
    before = after
  }

  session.raw = value
  event.target.value = value

  if (mode.value === 'words' && state.value.complete) {
    updateTimer()
    finish()
  } else if (
    mode.value === 'time' &&
    state.value.active > session.words.length - 30
  ) {
    session.words = generateWords(
      session.words.length + 180,
      'words',
      session.passageIndex,
    )
  }

  await nextTick()
  const caret = field.value?.querySelector('.caret')
  if (caret && field.value) {
    const lineHeight = parseFloat(
      getComputedStyle(caret.closest('.words')).lineHeight,
    )
    const top =
      caret.getBoundingClientRect().top -
      field.value.getBoundingClientRect().top
    if (top < 0 || top > field.value.clientHeight - lineHeight * 2) {
      field.value.scrollTop = Math.max(
        0,
        field.value.scrollTop + top - lineHeight,
      )
    }
  }
}

function wordHasError(index) {
  const entry = state.value.entries[index]
  const word = session.words[index]
  return (
    entry.extraSpaces > 0 ||
    (entry.submitted && entry.value !== word) ||
    [...entry.value].some((char, letterIndex) => char !== word[letterIndex])
  )
}

function letterClass(word, index, letterIndex) {
  const entry = state.value.entries[index]
  if (letterIndex >= entry.value.length)
    return entry.submitted ? 'incorrect' : ''
  return entry.value[letterIndex] === word[letterIndex]
    ? 'correct'
    : 'incorrect'
}

onBeforeUnmount(stopTimer)
</script>

<template>
  <q-layout class="app-shell"><svg class="icon-definitions" aria-hidden="true">
      <defs>
        <path id="icon-clock" d="M12 8v4l3 2M21 12a9 9 0 1 1-18 0 9 9 0 0 1 18 0" />
        <path id="icon-redo" d="M20 7v5h-5M20 12a8 8 0 1 0-2 5M20 7l-3-2" />
        <path id="icon-arrow" d="m9 5 7 7-7 7" />
        <path id="icon-type" d="m5 20 7-16 7 16M8 14h8" />
        <path id="icon-adjust" d="M5 4v6m0 4v6M12 4v10m0 4v2M19 4v2m0 4v10M2 10h6m1 8h6m1-12h6" />
        <path id="icon-logo"
          d="M10 2h5m-2.5 0v28M10 30h5M8 9H5a4 4 0 0 0-4 4v6a4 4 0 0 0 4 4h3m9-14h12a5 5 0 0 1 5 5v4a5 5 0 0 1-5 5H17" />
      </defs>
    </svg>
    <q-page-container>
      <q-page class="app-page">
        <header class="brand" aria-label="TypingTest">
          <svg viewBox="0 0 36 32" class="icon" aria-hidden="true">
            <use href="#icon-logo" />
          </svg><span>TypingTest</span>
        </header>
        <main :class="['workspace', `screen-${screen}`]">
          <div v-if="screen !== 'result'" :class="['settings-wrap', { 'is-running': running }]">
            <nav class="settings" aria-label="Настройки теста">
              <div class="mode-options">
                <q-btn flat no-caps :class="{ selected: mode === 'time' }" :aria-pressed="mode === 'time'"
                  @click="changeMode('time')"><svg viewBox="0 0 24 24" class="icon" aria-hidden="true">
                    <use href="#icon-clock" />
                  </svg>Время</q-btn>
                <q-btn flat no-caps :class="{ selected: mode === 'words' }" :aria-pressed="mode === 'words'"
                  @click="changeMode('words')"><svg viewBox="0 0 24 24" class="icon" aria-hidden="true">
                    <use href="#icon-type" />
                  </svg>Слова</q-btn>
              </div>
              <span class="divider" />
              <div class="amount-options">
                <q-btn v-for="value in presets" :key="value" flat no-caps :class="{ selected: amount === value }"
                  :aria-pressed="amount === value" :aria-label="`${value} ${amountLabel}`" @click="setAmount(value)">{{
                    value }}</q-btn>
                <q-btn flat class="custom-button" aria-label="Своё значение"
                  :class="{ selected: !presets.includes(amount) }" @click="openCustom"><svg viewBox="0 0 24 24"
                    class="icon" aria-hidden="true">
                    <use href="#icon-adjust" />
                  </svg><q-tooltip>Своё значение · слова или символы</q-tooltip></q-btn>
              </div>
            </nav>
            <span v-if="unit === 'characters' && mode === 'words'" class="unit-caption">количество символов</span>
          </div>

          <section v-if="screen === 'start'" class="start-actions">
            <q-btn unelevated no-caps class="button-primary start-button" @click="start()">Начать тест<svg
                viewBox="0 0 24 24" class="icon" aria-hidden="true">
                <use href="#icon-arrow" />
              </svg></q-btn>
          </section>

          <section v-if="screen === 'test'" class="test" aria-label="Тест скорости печати">
            <div class="progress" aria-live="off">{{ progress }}</div>
            <div :class="['typing-area', { unfocused: !focused }]" @click="focusInput">
              <div ref="field" class="words-viewport" aria-hidden="true">
                <div class="words">
                  <span v-for="(word, index) in session.words" :key="index"
                    :class="['word', { 'word-error': wordHasError(index) }]">
                    <span v-if="state.entries[index].extraSpaces" class="extra">{{
                      '␣'.repeat(state.entries[index].extraSpaces) }}</span><span v-for="(letter, letterIndex) in word"
                      :key="letterIndex"
                      :class="['letter', letterClass(word, index, letterIndex), { caret: state.active === index && state.entries[index].value.length === letterIndex, },]">{{
                        letter }}</span>
                    <span v-for="(letter, extraIndex) in state.entries[index].value.slice(word.length)"
                      :key="`extra-${extraIndex}`" class="letter extra">{{ letter }}</span>
                    <span v-if="state.active === index && state.entries[index].value.length >= word.length"
                      class="caret end-caret" />
                  </span>
                </div>
              </div>
              <textarea ref="input" class="typing-input" aria-label="Введите текст теста" :value="session.raw"
                autocomplete="off" autocapitalize="off" :spellcheck="false" @input="onInput" @compositionend="onInput"
                @focus="focused = true" @blur="focused = false" @paste.prevent @drop.prevent @keydown.enter.prevent
                @keydown.end.prevent @keydown.home.prevent @keydown.left.prevent @keydown.right.prevent
                @keydown.up.prevent @keydown.down.prevent @keydown.esc.prevent="input.blur()" @select="
                  input.setSelectionRange(
                    session.raw.length,
                    session.raw.length,
                  )
                  " />
              <button v-if="!focused" class="focus-prompt" @click="focusInput">
                Нажмите здесь, чтобы продолжить
              </button>
            </div>
            <div class="test-actions">
              <q-btn unelevated no-caps class="button-secondary" @click="start(true)">Заново<svg viewBox="0 0 24 24"
                  class="icon" aria-hidden="true">
                  <use href="#icon-redo" />
                </svg></q-btn>
            </div>
            <p v-if="session.startedAt === null" class="typing-hint">
              Начните печатать, чтобы запустить тест
            </p>
          </section>

          <section v-if="screen === 'result'" class="results" aria-label="Результаты теста" aria-live="polite">
            <dl class="stats">
              <div>
                <dt>тип теста</dt>
                <dd>
                  {{
                    mode === 'time'
                      ? 'время'
                      : unit === 'words'
                        ? 'слова'
                        : 'символы'
                  }}
                  {{ amount }}
                </dd>
              </div>
              <div>
                <dt><abbr title="Символов в минуту">cpm</abbr></dt>
                <dd>{{ result.cpm }}</dd>
              </div>
              <div>
                <dt>точность</dt>
                <dd>{{ result.accuracy }}%</dd>
              </div>
              <div>
                <dt>символы</dt>
                <dd>{{ result.characters }}</dd>
              </div>
              <div>
                <dt>ошибки</dt>
                <dd>{{ result.errors }}</dd>
              </div>
            </dl>
            <div class="result-actions">
              <q-btn unelevated no-caps class="button-secondary" @click="start(true)">Заново<svg viewBox="0 0 24 24"
                  class="icon" aria-hidden="true">
                  <use href="#icon-redo" />
                </svg></q-btn><q-btn unelevated no-caps class="button-primary" @click="start()">Следующий тест<svg
                  viewBox="0 0 24 24" class="icon" aria-hidden="true">
                  <use href="#icon-arrow" />
                </svg></q-btn>
            </div>
            <q-btn flat no-caps class="change-settings" @click="screen = 'start'">Изменить настройки</q-btn>
          </section>

        </main>
      </q-page>
    </q-page-container>

    <q-dialog v-model="customOpen" @hide="screen === 'test' && focusInput()">
      <q-card class="custom-dialog">
        <form @submit.prevent="applyCustom">
          <h1>Настройки теста</h1>
          <label v-if="mode === 'words'" class="custom-label" for="count-unit">Объём текста</label>
          <select v-if="mode === 'words'" id="count-unit" v-model="customUnit">
            <option value="words">Количество слов</option>
            <option value="characters">Количество символов</option>
          </select>
          <label class="custom-label" for="custom-amount">{{
            mode === 'time'
              ? 'Время в секундах'
              : customUnit === 'words'
                ? 'Количество слов'
                : 'Количество символов'
          }}</label>
          <input id="custom-amount" v-model.number="customValue" type="number" min="1" :max="customLimit" autofocus
            required />
          <p class="custom-help">
            От 1 до {{ mode === 'time' ? '3600 секунд' : '5000' }}
          </p>
          <div class="dialog-actions">
            <q-btn flat no-caps label="Отмена" @click="customOpen = false" /><q-btn unelevated no-caps type="submit"
              class="button-primary" label="Применить" :disable="!customValid" />
          </div>
        </form>
      </q-card>
    </q-dialog>

  </q-layout>
</template>

<style lang="scss" src="./style.scss"></style>
