<script setup>
import { computed, ref } from 'vue'
import { useHrStore } from '@/store/hr'

const store = useHrStore()
const scoreView = ref('latest') // latest=策略最新结果, snapshot=投递时历史评分

// 按口径取应聘记录上的评分
const scoreOf = a => scoreView.value === 'snapshot' ? (a.snapshot ? a.snapshot.score : null) : (a.match ? a.match.score : null)

const funnel = computed(() => {
  const order = ['submitted', 'screening', 'interview', 'offer', 'hired']
  const m = {}
  store.applications.forEach(a => { if (a.stage !== 'rejected') m[a.stage] = (m[a.stage] || 0) + 1 })
  let prev = null
  return order.map(s => {
    const count = m[s] || 0
    const conv = prev != null && prev > 0 ? Math.round((count / prev) * 100) : null
    const row = { s, label: { submitted: '投递', screening: '筛选', interview: '面试', offer: 'Offer', hired: '录用' }[s], count, conv }
    prev = count
    return row
  })
})

const overallConv = computed(() => {
  const sub = (funnel.value[0]?.count) || 0
  const hire = funnel.value[4]?.count || 0
  return sub ? Math.round(hire / sub * 100) : 0
})

const channelCost = computed(() => {
  const map = {}
  store.candidates.forEach(c => map[c.channel] = (map[c.channel] || 0) + 1)
  return store.channels.map(ch => {
    const count = map[ch.name] || 0
    const hired = store.applications.filter(a => a.stage === 'hired' &&
      store.candidates.find(c => c.id === a.candidate_id)?.channel === ch.name).length
    return { name: ch.name, cost: ch.cost, count, cpc: count ? Math.round(ch.cost / count) : 0, hired }
  }).sort((a, b) => a.cpc - b.cpc)
})

const deptProgress = computed(() => {
  const map = {}
  store.applications.forEach(a => {
    const dept = a.dept
    map[dept] = map[dept] || { total: 0, hired: 0, interview: 0, offer: 0 }
    map[dept].total++
    if (a.stage === 'hired') map[dept].hired++
    if (a.stage === 'offer' || a.stage === 'hired') map[dept].offer++
    if (a.stage === 'interview' || a.stage === 'offer' || a.stage === 'hired') map[dept].interview++
  })
  return Object.entries(map).map(([k, v]) => ({ dept: k, ...v }))
})

const makeBuckets = () => ({
  sink: { label: '低匹配 0-59', count: 0, color: 'var(--red)' },
  mid: { label: '一般 60-79', count: 0, color: 'var(--accent2)' },
  hi: { label: '高匹配 80+', count: 0, color: 'var(--green)' }
})

// 匹配度分布按口径切换：最新结果 vs 投递时历史评分，两口径使用同一分桶标准
const matchDist = computed(() => {
  const buckets = makeBuckets()
  store.applications.forEach(a => {
    const s = scoreOf(a)
    if (s == null) return
    if (s >= 80) buckets.hi.count++
    else if (s >= 60) buckets.mid.count++
    else buckets.sink.count++
  })
  return Object.values(buckets)
})

// 最新 vs 历史：可同时取到两种评分的投递，用于观察策略调整后分数迁移
const bothScored = computed(() => store.applications.filter(a => a.match && a.snapshot))
const avgLatest = computed(() => bothScored.value.length ? Math.round(bothScored.value.reduce((s, a) => s + a.match.score, 0) / bothScored.value.length) : null)
const avgSnapshot = computed(() => bothScored.value.length ? Math.round(bothScored.value.reduce((s, a) => s + a.snapshot.score, 0) / bothScored.value.length) : null)
const upgraded = computed(() => bothScored.value.filter(a => a.match.score > a.snapshot.score).length)
const downgraded = computed(() => bothScored.value.filter(a => a.match.score < a.snapshot.score).length)

const matchTotal = computed(() => matchDist.value.reduce((s, b) => s + b.count, 0) || 1)

const donutSegs = computed(() => matchDist.value.map((b, i) => {
  const len = (b.count / matchTotal.value) * 389.4
  return { ...b, len, offset: -(matchDist.value.slice(0, i).reduce((s, x) => s + ((x.count / matchTotal.value) * 389.4), 0)) }
}))

const avgSalary = computed(() => {
  const offers = store.offers.filter(o => o.status === 'accepted' || o.status === 'joined')
  return offers.length ? Math.round(offers.reduce((s, o) => s + o.salary, 0) / offers.length) : 0
})
</script>

<template>
  <div class="reports" v-if="store.loaded">
    <div class="stat-grid">
      <div class="card stat"><span>🔻</span><b>{{ overallConv }}%</b><em>总体录用转化</em></div>
      <div class="card stat"><span>💵</span><b class="money">¥{{ avgSalary.toLocaleString() }}</b><em>平均 Offer 薪资</em></div>
      <div class="card stat"><span>🎉</span><b class="money">{{ funnel[4]?.count }}</b><em>已录用人数</em></div>
      <div class="card stat"><span>📡</span><b>{{ store.channels.length }}</b><em>使用渠道</em></div>
    </div>

    <div class="row">
      <div class="card">
        <h3>🔻 招聘漏斗转化率</h3>
        <div class="funnel">
          <div v-for="(f, i) in funnel" :key="f.s">
            <div class="fl">
              <span class="flabel">{{ f.label }}</span>
              <b>{{ f.count }}</b>
              <span class="conv" v-if="f.conv != null">{{ f.conv }}%</span>
              <i class="arrow" v-if="i < funnel.length - 1">↓</i>
            </div>
          </div>
        </div>
      </div>

      <div class="card">
        <h3>💴 渠道成本与效率</h3>
        <div class="clist">
          <div v-for="c in channelCost" :key="c.name" class="crow">
            <span class="cname">{{ c.name }}</span>
            <span class="muted">获取 {{ c.count }} 人</span>
            <b class="cpc">¥{{ c.cpc }}</b>
            <span class="hired-tag" v-if="c.hired">入职{{ c.hired }}</span>
          </div>
        </div>
        <div class="muted tip">单位成本 = 渠道费用 ÷ 获取候选人数，入职数标注为该渠道贡献的录用。</div>
      </div>

      <div class="card">
        <h3>🏢 部门招聘进度</h3>
        <div class="dlist">
          <div v-for="d in deptProgress" :key="d.dept" class="drow">
            <div class="dhead">
              <span>{{ d.dept }}</span>
              <span class="muted">录用 {{ d.hired }}/{{ d.total }}</span>
            </div>
            <div class="bar"><i :style="{ width: (d.hired / (d.total || 1)) * 100 + '%', background: 'var(--green)' }"></i></div>
          </div>
        </div>
      </div>
    </div>

    <div class="row2">
      <div class="card">
        <h3>
          🎯 候选人匹配度分布
          <div class="score-toggle">
            <button :class="{ on: scoreView === 'latest' }" @click="scoreView = 'latest'">最新评分</button>
            <button :class="{ on: scoreView === 'snapshot' }" @click="scoreView = 'snapshot'">投递时评分</button>
          </div>
        </h3>
        <div class="donut-wrap">
          <svg viewBox="0 0 160 160" class="donut">
            <circle cx="80" cy="80" r="62" fill="none" stroke="#222a4a" stroke-width="26"/>
            <circle v-for="(s, i) in donutSegs" :key="s.label" cx="80" cy="80" r="62" fill="none"
              :stroke="s.color" stroke-width="26"
              :stroke-dasharray="s.len" :stroke-dashoffset="s.offset" transform="rotate(-90 80 80)"/>
          </svg>
          <div class="center"><b>{{ matchTotal }}</b><em class="muted">{{ scoreView === 'snapshot' ? '历史评分应聘' : '最新评分应聘' }}</em></div>
        </div>
        <div class="leg">
          <div v-for="s in matchDist" :key="s.label"><span class="sw" :style="{background:s.color}"></span>{{ s.label }}<b>{{ s.count }}人</b></div>
        </div>
        <div class="muted tip">「最新评分」为策略发布与批量重算后的结果；「投递时评分」为候选人投递那一刻冻结的历史依据，两者独立统计、互不覆盖。</div>
      </div>

      <div class="card">
        <h3>📊 招聘效率概况</h3>
        <div class="kpis">
          <div><em class="muted">简历投递 → 筛选</em><b>{{ funnel[1]?.conv ?? 0 }}%</b></div>
          <div><em class="muted">筛选 → 面试</em><b>{{ funnel[2]?.conv ?? 0 }}%</b></div>
          <div><em class="muted">面试 → Offer</em><b>{{ funnel[3]?.conv ?? 0 }}%</b></div>
          <div><em class="muted">Offer → 录用</em><b>{{ funnel[4]?.conv ?? 0 }}%</b></div>
        </div>
        <div class="score-cmp" v-if="bothScored.length">
          <div class="cmp-title muted">最新评分 vs 投递时评分（{{ bothScored.length }} 份在库应聘）</div>
          <div class="cmp-row">
            <span>平均分</span>
            <b class="hist-score">{{ avgSnapshot }}</b>
            <span class="arrow">→</span>
            <b class="new-score">{{ avgLatest }}</b>
            <em :class="avgLatest - avgSnapshot > 0 ? 'up' : avgLatest - avgSnapshot < 0 ? 'down' : ''">
              {{ avgLatest - avgSnapshot > 0 ? '▲' : avgLatest - avgSnapshot < 0 ? '▼' : '＝' }}{{ Math.abs(avgLatest - avgSnapshot) }}
            </em>
          </div>
          <div class="cmp-row sub">
            <span>分数上升 <b class="up">{{ upgraded }}</b> 人 · 下降 <b class="down">{{ downgraded }}</b> 人 · 持平 <b>{{ bothScored.length - upgraded - downgraded }}</b> 人</span>
          </div>
        </div>
        <div class="muted tip">建议：若「筛选→面试」转化低，可优化职位JD与筛选标准；「Offer→录用」低则需复核薪酬竞争力与流程效率。</div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.reports { display: flex; flex-direction: column; gap: 16px; }
.stat-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(180px, 1fr)); gap: 14px; }
.stat { display: flex; flex-direction: column; gap: 4px; }
.stat span { font-size: 24px; }
.stat b { font-size: 24px; }
.stat em { font-style: normal; color: var(--muted); font-size: 13px; }
.row { display: grid; grid-template-columns: 1.2fr 1fr 1fr; gap: 16px; }
.row2 { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
@media (max-width: 1000px) { .row, .row2 { grid-template-columns: 1fr; } }
.funnel { display: flex; flex-direction: column; gap: 2px; }
.fl { display: flex; align-items: center; gap: 12px; padding: 10px 12px; background: var(--panel2); border-radius: 8px; }
.flabel { width: 90px; }
.fl b { font-size: 20px; }
.conv { margin-left: auto; background: rgba(87,214,160,.16); color: var(--green); padding: 2px 8px; border-radius: 10px; font-size: 12px; }
.arrow { color: var(--muted); text-align: center; display: block; padding: 2px 0 2px 100px; }
.clist, .dlist { display: flex; flex-direction: column; gap: 10px; }
.crow { display: flex; align-items: center; gap: 12px; padding: 9px 10px; background: var(--panel2); border-radius: 8px; font-size: 13px; }
.cname { width: 90px; font-weight: 600; }
.cpc { margin-left: auto; font-size: 16px; color: var(--accent2); }
.hired-tag { font-size: 11px; background: rgba(87,214,160,.15); color: var(--green); padding: 2px 8px; border-radius: 10px; }
.tip { margin-top: 10px; line-height: 1.6; }
.drow { display: flex; flex-direction: column; gap: 6px; }
.dhead { display: flex; justify-content: space-between; font-size: 13px; }
.bar { height: 9px; background: var(--panel2); border-radius: 5px; overflow: hidden; }
.bar i { display: block; height: 100%; transition: .4s; }
.donut-wrap { position: relative; width: 160px; margin: 6px auto; }
.donut { width: 160px; height: 160px; }
.center { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; }
.center b { font-size: 26px; }
.leg { display: flex; flex-direction: column; gap: 6px; margin-top: 10px; }
.leg div { display: flex; align-items: center; gap: 6px; font-size: 13px; }
.leg b { margin-left: auto; }
.sw { width: 10px; height: 10px; border-radius: 3px; }
.kpis { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
.kpis div { background: var(--panel2); border-radius: 10px; padding: 14px; text-align: center; }
.kpis em { display: block; font-style: normal; font-size: 12px; }
.kpis b { font-size: 24px; color: var(--accent); }
.score-toggle { display: inline-flex; margin-left: auto; border: 1px solid var(--border); border-radius: 8px; overflow: hidden; }
.score-toggle button { border: none; border-radius: 0; padding: 4px 10px; font-size: 11px; background: transparent; }
.score-toggle button.on { background: rgba(91,140,255,.18); color: var(--accent); }
.score-cmp { margin-top: 12px; background: var(--panel2); border-radius: 10px; padding: 12px 14px; }
.cmp-title { font-size: 12px; margin-bottom: 8px; }
.cmp-row { display: flex; align-items: center; gap: 10px; font-size: 14px; }
.cmp-row .arrow { color: var(--muted); }
.hist-score { color: var(--accent2); font-size: 20px; }
.new-score { color: var(--green); font-size: 20px; }
.cmp-row em { font-style: normal; font-size: 12px; }
.cmp-row em.up, .up { color: var(--green); }
.cmp-row em.down, .down { color: var(--red); }
.cmp-row.sub { margin-top: 6px; font-size: 12px; color: var(--muted); }
</style>