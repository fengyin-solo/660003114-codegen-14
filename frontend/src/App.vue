<template>
  <div class="min-h-screen bg-slate-900 text-slate-200">
    <header class="border-b border-slate-700 px-6 py-4 flex items-center justify-between flex-wrap gap-3">
      <div>
        <h1 class="text-2xl font-bold text-cyan-400">正则表达式可视化调试器</h1>
        <p class="text-sm text-slate-500 mt-1">NFA 状态机可视化 · 逐步匹配高亮 · 分组捕获 · 回溯追踪</p>
      </div>
      <div class="flex items-center gap-2 flex-wrap">
        <button @click="showShareModal = true" class="px-4 py-2 bg-purple-600 hover:bg-purple-500 rounded-lg text-white font-bold text-sm flex items-center gap-1">
          🔗 分享快照
        </button>
        <button @click="saveCurrentSnapshot" class="px-4 py-2 bg-emerald-600 hover:bg-emerald-500 rounded-lg text-white font-bold text-sm flex items-center gap-1">
          💾 保存快照
        </button>
      </div>
    </header>

    <div class="flex flex-col lg:flex-row gap-4 p-4">
      <div class="lg:w-1/4 space-y-4">
        <RegexEditor />
        <TemplateLibrary />
      </div>

      <div class="lg:w-1/2 space-y-4">
        <NfaVisualizer />
        <MatchHighlight />
      </div>

      <div class="lg:w-1/4 space-y-4">
        <div class="bg-slate-800 rounded-lg p-4 border border-slate-700">
          <h3 class="text-sm font-bold text-slate-400 mb-3">匹配统计</h3>
          <div v-if="store.matchResult" class="space-y-2 text-sm">
            <div class="flex justify-between"><span class="text-slate-500">匹配状态</span><span :class="store.matchResult.matched ? 'text-green-400' : 'text-red-400'">{{ store.matchResult.matched ? '✓ 匹配成功' : '✗ 未匹配' }}</span></div>
            <div class="flex justify-between"><span class="text-slate-500">匹配文本</span><span class="text-cyan-400 font-mono truncate ml-2">{{ store.matchResult.matchText || '—' }}</span></div>
            <div class="flex justify-between"><span class="text-slate-500">总步数</span><span class="text-slate-300">{{ store.matchResult.totalSteps }}</span></div>
            <div class="flex justify-between"><span class="text-slate-500">回溯次数</span><span :class="store.matchResult.backtracks > 0 ? 'text-orange-400 font-bold' : 'text-slate-300'">{{ store.matchResult.backtracks }}</span></div>
            <div class="flex justify-between"><span class="text-slate-500">耗时(ms)</span><span class="text-slate-300">{{ store.matchResult.duration }}</span></div>
          </div>
          <div v-else class="text-slate-500 text-sm">点击"执行匹配"开始</div>
        </div>

        <div class="bg-slate-800 rounded-lg p-4 border border-slate-700">
          <h3 class="text-sm font-bold text-slate-400 mb-3">逐步控制</h3>
          <div class="flex flex-wrap items-center gap-2 mb-3">
            <button @click="store.stepBackward" :disabled="store.currentStep === 0" class="px-3 py-1 bg-slate-700 hover:bg-slate-600 disabled:opacity-30 rounded text-sm">⏮ 上一步</button>
            <button v-if="!store.isPlaying" @click="store.play" class="px-3 py-1 bg-cyan-600 hover:bg-cyan-500 rounded text-sm">▶ 播放</button>
            <button v-else @click="store.stop" class="px-3 py-1 bg-red-600 hover:bg-red-500 rounded text-sm">⏸ 停止</button>
            <button @click="store.stepForward" :disabled="!store.matchResult || store.currentStep >= store.matchResult.steps.length - 1" class="px-3 py-1 bg-slate-700 hover:bg-slate-600 disabled:opacity-30 rounded text-sm">下一步 ⏭</button>
            <button @click="store.resetStep" class="px-3 py-1 bg-slate-700 hover:bg-slate-600 rounded text-sm">⟲ 重置</button>
          </div>
          <div class="text-sm text-slate-400">步骤: {{ store.currentStep }} / {{ store.matchResult?.steps.length || 0 }}</div>
        </div>

        <div class="bg-slate-800 rounded-lg p-4 border border-slate-700">
          <h3 class="text-sm font-bold text-slate-400 mb-3">当前步骤详情</h3>
          <div v-if="store.matchResult && store.matchResult.steps[store.currentStep]" class="space-y-1 text-sm">
            <div>字符索引: <span class="text-cyan-400">{{ store.matchResult.steps[store.currentStep].charIndex }}</span></div>
            <div>当前字符: <span class="text-yellow-400 font-mono">'{{ store.matchResult.steps[store.currentStep].char }}'</span></div>
            <div>状态转换: <span class="text-green-400">{{ store.matchResult.steps[store.currentStep].currentState }}</span> → <span class="text-blue-400">{{ store.matchResult.steps[store.currentStep].nextState }}</span></div>
            <div>转移符号: <span class="text-purple-400 font-mono">{{ store.matchResult.steps[store.currentStep].transition }}</span></div>
            <div v-if="store.matchResult.steps[store.currentStep].isBacktrack" class="text-orange-400 font-bold">⚠ 回溯发生</div>
          </div>
          <div v-else class="text-slate-500 text-sm">无步骤数据</div>
        </div>

        <div class="bg-slate-800 rounded-lg p-4 border border-slate-700">
          <h3 class="text-sm font-bold text-slate-400 mb-3 flex items-center justify-between">
            <span>📋 已保存快照</span>
            <span class="text-xs text-slate-500">({{ store.snapshots.length }}/20)</span>
          </h3>
          <div v-if="store.snapshots.length === 0" class="text-slate-500 text-sm">暂无快照，点击"保存快照"创建</div>
          <div v-else class="space-y-2 max-h-64 overflow-y-auto">
            <div v-for="(snap, idx) in store.snapshots" :key="idx" class="bg-slate-900 rounded p-2 text-xs border border-slate-700">
              <div class="flex items-center justify-between mb-1">
                <span class="text-cyan-400 font-bold truncate">{{ snap.title || '未命名快照' }}</span>
                <span class="text-slate-500">{{ formatTime(snap.timestamp) }}</span>
              </div>
              <div class="text-slate-400 font-mono truncate mb-1">/{{ snap.pattern }}/</div>
              <div class="text-slate-500 truncate mb-2">测试: {{ snap.testString }}</div>
              <div class="flex gap-1">
                <button @click="store.restoreSnapshot(snap)" class="px-2 py-0.5 bg-cyan-600 hover:bg-cyan-500 rounded text-white text-xs">恢复</button>
                <button @click="copySnapshotUrl(snap)" class="px-2 py-0.5 bg-purple-600 hover:bg-purple-500 rounded text-white text-xs">分享</button>
                <button @click="store.deleteSnapshot(idx)" class="px-2 py-0.5 bg-red-600 hover:bg-red-500 rounded text-white text-xs">删除</button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div v-if="showShareModal" class="fixed inset-0 bg-black/60 flex items-center justify-center z-50 p-4" @click.self="showShareModal = false">
      <div class="bg-slate-800 rounded-xl border border-slate-600 p-6 max-w-lg w-full shadow-2xl">
        <div class="flex items-center justify-between mb-4">
          <h2 class="text-xl font-bold text-cyan-400">🔗 分享调试快照</h2>
          <button @click="showShareModal = false" class="text-slate-400 hover:text-white text-2xl leading-none">&times;</button>
        </div>

        <div class="mb-4">
          <label class="block text-sm text-slate-400 mb-1">快照标题（可选）</label>
          <input v-model="snapshotTitle" type="text" placeholder="为这个快照起个名字..." class="w-full bg-slate-900 border border-slate-600 rounded-lg px-3 py-2 text-slate-200 text-sm focus:outline-none focus:border-cyan-500" />
        </div>

        <div class="space-y-3">
          <div>
            <label class="block text-sm text-slate-400 mb-1">分享链接</label>
            <div class="flex gap-2">
              <input readonly :value="store.shareUrl || generatedPreviewUrl" type="text" class="flex-1 bg-slate-900 border border-slate-600 rounded-lg px-3 py-2 text-slate-300 text-xs font-mono focus:outline-none" />
              <button @click="copyCurrentShareUrl" class="px-4 py-2 bg-purple-600 hover:bg-purple-500 rounded-lg text-white text-sm font-bold whitespace-nowrap">
                {{ copySuccess ? '✓ 已复制' : '复制链接' }}
              </button>
            </div>
            <p class="text-xs text-slate-500 mt-1">链接包含完整的表达式、测试字符串和当前步骤，打开即可恢复。</p>
          </div>

          <div>
            <div class="flex items-center justify-between mb-1">
              <label class="block text-sm text-slate-400">JSON 快照数据</label>
              <button @click="copyJsonSnapshot" class="text-xs text-cyan-400 hover:text-cyan-300">复制JSON</button>
            </div>
            <textarea readonly :value="jsonSnapshotPreview" rows="6" class="w-full bg-slate-900 border border-slate-600 rounded-lg px-3 py-2 text-slate-300 text-xs font-mono focus:outline-none resize-none"></textarea>
          </div>

          <div>
            <label class="block text-sm text-slate-400 mb-1">导入 JSON 快照</label>
            <textarea v-model="importJsonText" rows="3" placeholder="粘贴 JSON 快照数据..." class="w-full bg-slate-900 border border-slate-600 rounded-lg px-3 py-2 text-slate-200 text-xs font-mono focus:outline-none focus:border-cyan-500 resize-none"></textarea>
            <div class="flex gap-2 mt-2">
              <button @click="importSnapshot" :disabled="!importJsonText.trim()" class="px-4 py-2 bg-emerald-600 hover:bg-emerald-500 disabled:opacity-40 rounded-lg text-white text-sm font-bold">导入并恢复</button>
              <button v-if="importError" class="text-xs text-red-400 self-center">{{ importError }}</button>
            </div>
          </div>
        </div>

        <div class="mt-5 pt-4 border-t border-slate-700 flex justify-end gap-2">
          <button @click="showShareModal = false" class="px-4 py-2 bg-slate-700 hover:bg-slate-600 rounded-lg text-white text-sm">关闭</button>
        </div>
      </div>
    </div>

    <div v-if="toast" class="fixed top-20 left-1/2 -translate-x-1/2 bg-slate-800 border border-slate-600 text-white px-4 py-2 rounded-lg shadow-lg z-50 text-sm">
      {{ toast }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref, computed, watch } from 'vue'
import { useRegexStore } from './store/regex'
import RegexEditor from './components/RegexEditor.vue'
import NfaVisualizer from './components/NfaVisualizer.vue'
import MatchHighlight from './components/MatchHighlight.vue'
import TemplateLibrary from './components/TemplateLibrary.vue'
import type { DebugSnapshot } from './types'

const store = useRegexStore()

const showShareModal = ref(false)
const snapshotTitle = ref('')
const copySuccess = ref(false)
const toast = ref('')
const importJsonText = ref('')
const importError = ref('')

const generatedPreviewUrl = computed(() => store.encodeSnapshotToUrl())
const jsonSnapshotPreview = computed(() => store.exportSnapshotAsJson())

let toastTimer: ReturnType<typeof setTimeout>
function showToast(msg: string) {
  toast.value = msg
  clearTimeout(toastTimer)
  toastTimer = setTimeout(() => { toast.value = '' }, 2000)
}

function formatTime(ts: number): string {
  const d = new Date(ts)
  const pad = (n: number) => String(n).padStart(2, '0')
  return `${pad(d.getMonth() + 1)}/${pad(d.getDate())} ${pad(d.getHours())}:${pad(d.getMinutes())}`
}

function saveCurrentSnapshot() {
  store.saveSnapshot(snapshotTitle.value || undefined)
  snapshotTitle.value = ''
  showToast('✓ 快照已保存')
}

async function copyCurrentShareUrl() {
  const ok = await store.copyShareUrl()
  copySuccess.value = ok
  showToast(ok ? '✓ 链接已复制到剪贴板' : '✗ 复制失败')
  if (ok) setTimeout(() => { copySuccess.value = false }, 2000)
}

async function copySnapshotUrl(snap: DebugSnapshot) {
  const ok = await store.copyShareUrl(snap)
  showToast(ok ? '✓ 分享链接已复制' : '✗ 复制失败')
}

async function copyJsonSnapshot() {
  try {
    await navigator.clipboard.writeText(jsonSnapshotPreview.value)
    showToast('✓ JSON 已复制')
  } catch (e) {
    showToast('✗ 复制失败')
  }
}

function importSnapshot() {
  importError.value = ''
  const snap = store.importSnapshotFromJson(importJsonText.value)
  if (snap) {
    store.restoreSnapshot(snap)
    importJsonText.value = ''
    showShareModal.value = false
    showToast('✓ 快照已恢复')
  } else {
    importError.value = '无效的快照数据'
  }
}

watch(showShareModal, (v) => {
  if (!v) {
    snapshotTitle.value = ''
    importJsonText.value = ''
    importError.value = ''
    copySuccess.value = false
  }
})

onMounted(() => {
  const urlSnap = store.decodeSnapshotFromUrl()
  if (urlSnap) {
    store.restoreSnapshot(urlSnap)
    showToast('✓ 已从链接恢复快照')
  } else {
    store.execute()
  }
})
</script>
