<script setup>
import { computed, ref } from 'vue'
import { useHrStore } from '@/store/hr'

const store = useHrStore()

const stages = [
  { k: 'submitted', label: '投递', icon: '📥', color: '#5b8cff' },
  { k: 'screening', label: '筛选', icon: '🔍', color: '#a78bfa' },
  { k: 'interview', label: '面试', icon: '💬', color: '#4fc3f7' },
  { k: 'offer', label: 'Offer', icon: '📄', color: '#ffd166' },
  { k: 'hired', label: '录用', icon: '🎉', color: '#57d6a0' }
]

const filter = ref('all')
const stageFilter = ref('all')
const scoreView = ref('latest') // latest=最新结果, snapshot=投递时历史评分

const list = computed(() => store.applications.filter(a => a.stage !== 'rejected'))
const viewList = computed(() => list.value.filter(a =>
  (stageFilter.value === 'all' || a.stage === stageFilter.value) &&
  (filter.value === 'all' || a.position === filter.value)
))

// 按当前口径取理由/短板：看板支持在"最新结果"与"投递时历史评分"之间切换
const dispWeakness = a => scoreView.value === 'snapshot' ? (a.snapshot?.weakness || '') : (a.match?.weakness || a.snapshot?.weakness || '')
const dispReason = a => scoreView.value === 'snapshot' ? (a.snapshot?.reason || '') : (a.match?.reason || a.snapshot?.reason || '')

const nextStage = s => ({ submitted: 'screening', screening: 'interview', interview: 'offer', offer: 'hired' }[s])
const nextStageLabel = s => ({ screening: '筛选', interview: '面试', offer: 'Offer' }[s] || '')

// 评分对比：latest=策略重算后的最新结果；snapshot=投递时冻结的历史依据
const scoreDelta = a => {
  if (!a.match || !a.snapshot) return null
  return a.match.score - a.snapshot.score
}
const deltaClass = d => d > 0 ? 'up' : d < 0 ? 'down' : 'flat'
</script>

<template>
  <div class="pipeline">
    <div class="bar">
      <div class="filters">
        <select v-model="filter">
          <option value="all">全部职位</option>
          <option v-for="p in store.openPositions" :key="p.id" :value="p.name">{{ p.name }}</option>
        </select>
        <select v-model="stageFilter">
          <option value="all">全部阶段</option>
          <option v-for="s in stages" :key="s.k" :value="s.k">{{ s.label }}</option>
        </select>
        <div class="score-toggle" title="切换看板展示的评分口径">
          <button :class="{ on: scoreView === 'latest' }" @click="scoreView = 'latest'">最新评分</button>
          <button :class="{ on: scoreView === 'snapshot' }" @click="scoreView = 'snapshot'">投递时评分</button>
        </div>
      </div>
    </div>

    <!-- Kanban -->
    <div class="kanban">
      <div class="kcol" v-for="s in stages" :key="s.k">
        <div class="khead" :style="{ borderColor: s.color }">
          <span>{{ s.icon }}</span><b>{{ s.label }}</b>
          <em class="tag">{{ viewList.filter(a => a.stage === s.k).length }}</em>
        </div>
        <div class="kbody">
          <div class="kcard card" v-for="a in viewList.filter(c => c.stage === s.k)" :key="a.id">
            <div class="kname">{{ a.candidate }}</div>
            <div class="kpos muted">{{ a.position }} · {{ a.dept }}</div>
            <div class="kmatch" v-if="a.match || a.snapshot">
              <div class="scores-col">
                <div class="score-line" :class="{ dim: scoreView === 'snapshot' && a.match }">
                  <span class="msc" :class="a.match ? (a.match.score >= 75 ? 's-hi' : a.match.score >= 55 ? 's-mid' : 's-lo') : ''">{{ a.match ? a.match.score : '—' }}</span>
                  <span class="s-label">最新</span>
                </div>
                <div class="score-line hist" :class="{ dim: scoreView === 'latest' && a.snapshot }" v-if="a.snapshot">
                  <span class="msc mini" :class="a.snapshot.score >= 75 ? 's-hi' : a.snapshot.score >= 55 ? 's-mid' : 's-lo'">{{ a.snapshot.score }}</span>
                  <span class="s-label">投递时</span>
                  <span class="delta" v-if="scoreDelta(a) !== null" :class="deltaClass(scoreDelta(a))">
                    {{ scoreDelta(a) > 0 ? '▲' : scoreDelta(a) < 0 ? '▼' : '＝' }}{{ Math.abs(scoreDelta(a)) }}
                  </span>
                </div>
              </div>
              <div class="minfo">
                <div class="mweak" v-if="dispWeakness(a) && dispWeakness(a) !== '无显著短板'">短板：{{ dispWeakness(a) }}</div>
                <div class="mreason muted">{{ dispReason(a) }}</div>
                <div class="mview-tag" :class="scoreView">{{ scoreView === 'snapshot' ? '📌 投递时历史评分' : '🔄 策略最新评分' }}</div>
              </div>
            </div>
            <div class="kskills"><span class="skill-chip" v-for="sk in (a.candSkills||[]).slice(0,4)" :key="sk.k">{{ sk.k }}</span></div>
            <div class="kfoot">
              <span class="muted">{{ a.city }}</span>
              <div class="ka">
                <button v-if="nextStage(a.stage)" class="primary" @click="store.advance(a.id)">→ {{ nextStageLabel(a.stage) }}</button>
                <span v-else class="succ-chip">✅ 已录用</span>
                <button class="danger" v-if="a.stage !== 'hired'" @click="store.reject(a.id)">淘汰</button>
              </div>
            </div>
          </div>
          <div class="muted empty-mini" v-if="!viewList.filter(c => c.stage === s.k).length">暂无</div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.pipeline { display: flex; flex-direction: column; gap: 14px; }
.bar { display: flex; }
.filters { display: flex; gap: 8px; align-items: center; flex-wrap: wrap; }
.score-toggle { display: inline-flex; border: 1px solid var(--border); border-radius: 8px; overflow: hidden; }
.score-toggle button { border: none; border-radius: 0; padding: 6px 12px; font-size: 12px; background: transparent; }
.score-toggle button.on { background: rgba(91,140,255,.18); color: var(--accent); }
.kanban { display: grid; grid-template-columns: repeat(5, 1fr); gap: 12px; }
@media (max-width: 1100px) { .kanban { grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); } }
.kcol { background: rgba(19,25,44,.6); border-radius: 12px; padding: 10px; border: 1px solid var(--border); }
.khead { display: flex; align-items: center; gap: 8px; padding-bottom: 8px; border-bottom: 2px solid; margin-bottom: 10px; }
.khead b { font-size: 14px; }
.khead .tag { margin-left: auto; }
.kbody { display: flex; flex-direction: column; gap: 10px; min-height: 60px; }
.kcard { padding: 12px; display: flex; flex-direction: column; gap: 8px; }
.kname { font-weight: 700; font-size: 14px; }
.kpos { font-size: 12px; }
.kskills { display: flex; flex-wrap: wrap; }
.kmatch { display: flex; gap: 8px; align-items: flex-start; background: var(--panel2); border-radius: 8px; padding: 6px 8px; }
.scores-col { display: flex; flex-direction: column; gap: 4px; flex-shrink: 0; }
.score-line { display: flex; align-items: center; gap: 4px; }
.score-line.dim { opacity: .4; }
.s-label { font-size: 10px; color: var(--muted); }
.msc.mini { min-width: 24px; height: 24px; font-size: 12px; border-radius: 6px; }
.delta { font-size: 10px; font-weight: 700; }
.delta.up { color: var(--green); }
.delta.down { color: var(--red); }
.delta.flat { color: var(--muted); }
.kmatch .msc { flex-shrink: 0; min-width: 32px; height: 32px; border-radius: 8px; display: flex; align-items: center; justify-content: center; font-weight: 800; font-size: 14px; }
.minfo { min-width: 0; flex: 1; display: flex; flex-direction: column; gap: 2px; }
.mweak { font-size: 11px; color: var(--red); }
.mreason { font-size: 10.5px; line-height: 1.4; word-break: break-all; }
.mview-tag { font-size: 10px; border-radius: 8px; padding: 1px 6px; align-self: flex-start; margin-top: 2px; }
.mview-tag.latest { background: rgba(91,140,255,.16); color: var(--accent); }
.mview-tag.snapshot { background: rgba(255,209,102,.14); color: var(--accent2); }
.kfoot { display: flex; flex-direction: column; gap: 8px; }
.ka { display: flex; gap: 6px; flex-wrap: wrap; }
.ka button { font-size: 11px; padding: 4px 8px; }
.succ-chip { color: var(--green); font-size: 12px; }
.empty-mini { font-size: 12px; padding: 10px; text-align: center; }
</style>