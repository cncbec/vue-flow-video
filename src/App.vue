<script setup>
import { computed, ref } from 'vue'
import { VueFlow } from '@vue-flow/core'
import { Background } from '@vue-flow/background'
import { Controls } from '@vue-flow/controls'
import { MiniMap } from '@vue-flow/minimap'

const llmName = ref('剧本导演 LLM')
const selectedTone = ref('国风赛博 / 悬疑热血')
const imageModel = ref('Seedream')
const videoModel = ref('Seedance')
const prompt = ref(
  '一个失忆少年在雨夜霓虹城醒来，靠一只会发光的纸鹤寻找身世，并在最终揭开守城 AI 的秘密。',
)
const currentStageIndex = ref(1)
const messages = ref([
  {
    id: 1,
    role: 'assistant',
    text: '你好，我会以对话方式陪你生成视频漫剧。先告诉我故事梗概、风格、时长和角色要求，我会把进度同步到右侧工作流画布。',
  },
  {
    id: 2,
    role: 'user',
    text: '我要生成一部 60 秒竖屏漫剧，风格要有雨夜、霓虹、东方幻想。',
  },
  {
    id: 3,
    role: 'assistant',
    text: '已拆解为「脚本 → 资产 → 分镜 → 分镜图 → 分镜视频 → 合成」流水线。下一步我将先生成脚本大纲和角色设定。',
  },
])

const stages = [
  {
    id: 'script',
    title: '脚本生成',
    description: 'LLM 通过多轮对话完善故事梗概、节奏点、台词和旁白。',
    model: 'LLM 编剧 Agent',
    output: '60 秒三幕式脚本、旁白、对白、情绪曲线',
    position: { x: 0, y: 60 },
  },
  {
    id: 'assets',
    title: '角色 / 场景 / 道具图片',
    description: '抽取视觉资产清单，并用 Seedream 生成统一风格设定图。',
    model: 'Seedream',
    output: '主角、纸鹤、霓虹雨巷、守城 AI 核心图',
    position: { x: 320, y: 60 },
  },
  {
    id: 'storyboard',
    title: '分镜规划',
    description: 'LLM 将脚本拆成镜头，输出景别、机位、运动和时长。',
    model: 'LLM 分镜 Agent',
    output: '8 个镜头，含景别、动作、字幕、音效提示',
    position: { x: 640, y: 60 },
  },
  {
    id: 'frames',
    title: '分镜图生成',
    description: '将每个镜头的视觉提示词交给 Seedream 生成关键帧。',
    model: 'Seedream',
    output: '8 张 9:16 分镜关键帧，角色一致性锁定',
    position: { x: 960, y: 60 },
  },
  {
    id: 'clips',
    title: '分镜视频生成',
    description: '使用 Seedance 将关键帧转成动态镜头，补充镜头运动。',
    model: 'Seedance',
    output: '8 段 4~8 秒镜头视频',
    position: { x: 1280, y: 60 },
  },
  {
    id: 'compose',
    title: '视频合成',
    description: '合成镜头、字幕、配音、音效、转场并导出成片。',
    model: '合成编排器',
    output: '60 秒竖屏 MP4 + 可回溯工程文件',
    position: { x: 1600, y: 60 },
  },
]

const storyboardShots = [
  { no: '01', name: '雨夜苏醒', time: '0-7s', detail: '少年在霓虹积水中睁眼，纸鹤发光。' },
  { no: '02', name: '追逐纸鹤', time: '7-16s', detail: '跟拍穿过广告牌与灯笼交错的小巷。' },
  { no: '03', name: '记忆碎片', time: '16-28s', detail: '道具触发童年与守城 AI 的闪回。' },
  { no: '04', name: '核心对峙', time: '28-45s', detail: '主角进入塔顶核心室，发现 AI 在守护城市。' },
  { no: '05', name: '选择与和解', time: '45-60s', detail: '纸鹤化作钥匙，城市天幕重新点亮。' },
]

const nodes = computed(() =>
  stages.map((stage, index) => ({
    id: stage.id,
    type: 'default',
    position: stage.position,
    data: {
      label: `${index + 1}. ${stage.title}\n${stage.model}\n${stage.output}`,
    },
    class: {
      'stage-node': true,
      active: index === currentStageIndex.value,
      done: index < currentStageIndex.value,
      waiting: index > currentStageIndex.value,
    },
  })),
)

const edges = computed(() =>
  stages.slice(0, -1).map((stage, index) => ({
    id: `${stage.id}-${stages[index + 1].id}`,
    source: stage.id,
    target: stages[index + 1].id,
    animated: index < currentStageIndex.value,
    class: index < currentStageIndex.value ? 'edge-done' : 'edge-waiting',
  })),
)

const progress = computed(() => Math.round(((currentStageIndex.value + 1) / stages.length) * 100))
const activeStage = computed(() => stages[currentStageIndex.value])

function sendPrompt() {
  const text = prompt.value.trim()
  if (!text) return

  messages.value.push({ id: Date.now(), role: 'user', text })
  messages.value.push({
    id: Date.now() + 1,
    role: 'assistant',
    text: `收到，我会用「${selectedTone.value}」风格继续拆解，并在 ${activeStage.value.title} 阶段调用 ${activeStage.value.model}。`,
  })
  prompt.value = ''
}

function advanceStage() {
  if (currentStageIndex.value < stages.length - 1) {
    currentStageIndex.value += 1
    messages.value.push({
      id: Date.now(),
      role: 'assistant',
      text: `阶段推进：${activeStage.value.title}。当前产物：${activeStage.value.output}。`,
    })
  }
}

function resetDemo() {
  currentStageIndex.value = 0
  messages.value.push({
    id: Date.now(),
    role: 'assistant',
    text: '已重置演示流程。请从脚本生成开始，我会逐步调用 LLM、Seedream 与 Seedance。',
  })
}
</script>

<template>
  <main class="app-shell">
    <section class="hero-card">
      <div>
        <p class="eyebrow">AI Video Comic Studio</p>
        <h1>大模型对话式视频漫剧生成工作台</h1>
        <p class="hero-copy">
          前端基于 Vue 3 与 Vue Flow，将脚本、资产、分镜、分镜图、分镜视频和合成链路实时可视化，方便用户在生成过程中逐步确认与回溯。
        </p>
      </div>
      <div class="hero-metrics">
        <div>
          <strong>{{ progress }}%</strong>
          <span>生成进度</span>
        </div>
        <div>
          <strong>{{ imageModel }}</strong>
          <span>图片模型</span>
        </div>
        <div>
          <strong>{{ videoModel }}</strong>
          <span>图生视频模型</span>
        </div>
      </div>
    </section>

    <section class="workspace-grid">
      <aside class="chat-panel glass-card">
        <div class="panel-header">
          <div>
            <p class="eyebrow">LLM Chat</p>
            <h2>{{ llmName }}</h2>
          </div>
          <button class="ghost-button" type="button" @click="resetDemo">重置</button>
        </div>

        <div class="config-grid">
          <label>
            漫剧风格
            <select v-model="selectedTone">
              <option>国风赛博 / 悬疑热血</option>
              <option>都市奇幻 / 温暖治愈</option>
              <option>末日废土 / 高燃冒险</option>
            </select>
          </label>
          <label>
            图片模型
            <input v-model="imageModel" />
          </label>
          <label>
            图生视频模型
            <input v-model="videoModel" />
          </label>
        </div>

        <div class="messages">
          <article v-for="message in messages" :key="message.id" :class="['message', message.role]">
            <span>{{ message.role === 'assistant' ? 'AI' : '你' }}</span>
            <p>{{ message.text }}</p>
          </article>
        </div>

        <form class="prompt-box" @submit.prevent="sendPrompt">
          <textarea v-model="prompt" placeholder="输入剧情、角色、画风、时长、镜头偏好……" />
          <button type="submit">发送给 LLM</button>
        </form>
      </aside>

      <section class="flow-panel glass-card">
        <div class="panel-header">
          <div>
            <p class="eyebrow">Workflow Canvas</p>
            <h2>{{ activeStage.title }}</h2>
          </div>
          <button class="primary-button" type="button" @click="advanceStage">推进下一阶段</button>
        </div>

        <div class="stage-summary">
          <p>{{ activeStage.description }}</p>
          <span>模型 / 编排器：{{ activeStage.model }}</span>
        </div>

        <div class="flow-canvas">
          <VueFlow :nodes="nodes" :edges="edges" fit-view-on-init :min-zoom="0.45" :max-zoom="1.2">
            <Background pattern-color="#334155" :gap="22" />
            <MiniMap pannable zoomable />
            <Controls />
          </VueFlow>
        </div>
      </section>
    </section>

    <section class="detail-grid">
      <article class="glass-card pipeline-card">
        <p class="eyebrow">Pipeline</p>
        <h2>端到端生成链路</h2>
        <ol>
          <li v-for="(stage, index) in stages" :key="stage.id" :class="{ current: index === currentStageIndex, done: index < currentStageIndex }">
            <strong>{{ stage.title }}</strong>
            <span>{{ stage.description }}</span>
          </li>
        </ol>
      </article>

      <article class="glass-card storyboard-card">
        <p class="eyebrow">Storyboard Preview</p>
        <h2>分镜样例</h2>
        <div class="shot-list">
          <div v-for="shot in storyboardShots" :key="shot.no" class="shot-card">
            <b>{{ shot.no }}</b>
            <div>
              <strong>{{ shot.name }} · {{ shot.time }}</strong>
              <p>{{ shot.detail }}</p>
            </div>
          </div>
        </div>
      </article>
    </section>
  </main>
</template>
