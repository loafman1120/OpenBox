<script lang="ts" setup>
import { computed, onMounted, ref, watch } from 'vue'

type SkillKey =
  | 'img'
  | 'helper'
  | 'translate'
  | 'code'
  | 'research'
  | 'aiRead'
  | 'more'

export interface SkillItem {
  key: SkillKey | string
  label: string
  icon?: string // daisyUI icon 或者 emoji
  badge?: string
  disabled?: boolean
}

const props = withDefaults(
  defineProps<{
    modelValue?: string
    placeholder?: string
    loading?: boolean
    skills?: SkillItem[]
    enableDeepThink?: boolean
    deepThink?: boolean
    maxLength?: number
    rows?: number
  }>(),
  {
    modelValue: '',
    placeholder: '发消息或输入 / 选择技能',
    loading: false,
    enableDeepThink: true,
    deepThink: false,
    maxLength: 0,
    rows: 3,
    skills: () => [
      { key: 'img', label: '图像生成', icon: '🎨' },
      { key: 'helper', label: '帮我写作', icon: '✍️' },
      { key: 'translate', label: '翻译', icon: '🌐' },
      { key: 'code', label: '编程', icon: '🧑‍💻' },
      { key: 'research', label: '深入研究', icon: '🔎' },
      { key: 'aiRead', label: 'AI 摘要', icon: '🗞️' },
      { key: 'more', label: '更多', icon: '➕' }
    ]
  }
)

const emit = defineEmits<{
  (e: 'update:modelValue', v: string): void
  (e: 'send', payload: { text: string; deepThink: boolean; skill?: SkillItem | null }): void
  (e: 'pick-skill', skill: SkillItem): void
  (e: 'mic'): void
  (e: 'paste', text: string): void
  (e: 'toggle-deep-think', v: boolean): void
}>()

const text = ref(props.modelValue)
watch(
  () => props.modelValue,
  v => (text.value = v)
)
watch(text, v => emit('update:modelValue', v))

const deepThink = ref(props.deepThink)
watch(
  () => props.deepThink,
  v => (deepThink.value = v)
)

const currentSkill = ref<SkillItem | null>(null)
const canSend = computed(() => text.value.trim().length > 0 && !props.loading)
const counter = computed(() =>
  props.maxLength ? `${text.value.length}/${props.maxLength}` : ''
)

function onEnterSend(e: KeyboardEvent) {
  if (e.key === 'Enter' && !e.shiftKey) {
    e.preventDefault()
    trySend()
  }
}
function trySend() {
  if (!canSend.value) return
  if (props.maxLength && text.value.length > props.maxLength) return
  emit('send', { text: text.value.trim(), deepThink: deepThink.value, skill: currentSkill.value })
  text.value = ''
}

function pickSkill(s: SkillItem) {
  if (s.disabled) return
  currentSkill.value = s
  emit('pick-skill', s)
}

function toggleDeepThink() {
  if (!props.enableDeepThink) return
  deepThink.value = !deepThink.value
  emit('toggle-deep-think', deepThink.value)
}

async function pasteFromClipboard() {
  try {
    const t = await navigator.clipboard.readText()
    if (t) {
      text.value += (text.value ? '\n' : '') + t
      emit('paste', t)
    }
  } catch {
    // ignore
  }
}

function onMic() {
  emit('mic')
}

const textareaRef = ref<HTMLTextAreaElement | null>(null)
function autoResize() {
  const el = textareaRef.value
  if (!el) return
  el.style.height = 'auto'
  el.style.height = Math.min(el.scrollHeight, 240) + 'px'
}
onMounted(autoResize)
watch(text, autoResize)
</script>

<template>
  <div class="w-full">
    <!-- 输入框卡片 -->
    <div class="card bg-base-100 shadow-sm border border-base-200">
      <div class="card-body p-3 gap-2">
        <div class="flex items-start gap-2">
          <!-- 左侧选择技能 Dropdown -->
          <div class="dropdown">
            <div tabindex="0" role="button" class="btn btn-ghost btn-sm">
              <span class="opacity-70">/</span>
              <span class="ml-1">选择技能</span>
              <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 ml-1 opacity-70" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="m19 9-7 7-7-7" />
              </svg>
            </div>
            <ul tabindex="0" class="dropdown-content menu bg-base-100 rounded-box z-[1] w-56 p-2 shadow">
              <li v-for="s in props.skills" :key="s.key">
                <a :class="{ disabled: s.disabled }" @click="pickSkill(s)">
                  <span class="mr-2">{{ s.icon ?? '✨' }}</span>{{ s.label }}
                  <span v-if="s.badge" class="badge badge-sm ml-auto">{{ s.badge }}</span>
                </a>
              </li>
            </ul>
          </div>

          <!-- 文本域 -->
          <div class="flex-1">
            <textarea
              ref="textareaRef"
              v-model="text"
              :placeholder="props.placeholder"
              :rows="props.rows"
              :maxlength="props.maxLength || undefined"
              class="textarea textarea-bordered w-full resize-none leading-6"
              @keydown="onEnterSend"
            />
            <div class="mt-1 flex items-center justify-between text-xs text-base-content/60">
              <div class="flex items-center gap-2">
                <button
                  v-if="props.enableDeepThink"
                  class="btn btn-xs"
                  :class="deepThink ? 'btn-primary' : 'btn-ghost'"
                  @click="toggleDeepThink"
                >
                  <span class="mr-1">🧠</span> 深度思考
                </button>
                <span v-if="currentSkill" class="badge badge-outline">
                  已选：{{ currentSkill.label }}
                </span>
              </div>
              <span v-if="counter">{{ counter }}</span>
            </div>
          </div>

          <!-- 右侧动作 -->
          <div class="flex flex-col items-center gap-2 pl-1 pt-1">
            <button class="btn btn-ghost btn-sm" @click="pasteFromClipboard" title="粘贴">
              <span class="i">📋</span>
            </button>
            <button class="btn btn-ghost btn-sm" @click="onMic" title="语音">
              <span>🎤</span>
            </button>
            <button
              class="btn btn-primary btn-sm"
              :class="{ 'btn-disabled': !canSend, loading: props.loading }"
              @click="trySend"
              title="发送"
            >
              <span v-if="!props.loading">⏎</span>
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- 下方技能胶囊按钮组 -->
    <div class="mt-3 flex gap-2 overflow-x-auto no-scrollbar">
      <button
        v-for="s in props.skills"
        :key="s.key + '-pill'"
        class="btn btn-outline btn-sm rounded-full whitespace-nowrap"
        :class="{ 'btn-active': currentSkill?.key === s.key, 'btn-disabled': s.disabled }"
        @click="pickSkill(s)"
      >
        <span class="mr-1">{{ s.icon ?? '✨' }}</span>{{ s.label }}
      </button>
    </div>
  </div>
</template>

<style scoped>
.no-scrollbar::-webkit-scrollbar {
  display: none;
}
.no-scrollbar {
  -ms-overflow-style: none;
  scrollbar-width: none;
}
</style>