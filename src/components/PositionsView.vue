<script setup>
import { computed, ref } from 'vue'
import { useHrStore } from '@/store/hr'

const store = useHrStore()
const open = ref(false)
const editing = ref(null)
const form = ref({ name: '', dept: '技术部', city: '上海', level: 'P5', salary_min: 15000, salary_max: 30000, years: 2, slots: 1 })
const skillInput = ref('')
const skills = ref([])

// 匹配策略弹层
const stOpen = ref(false)
const stPos = ref(null)
const stForm = ref(null)
const stBusy = ref(false)
const stMsg = ref('')
const batchBusy = ref(false)

const DEFAULT_WEIGHTS = [
  { key: 'w_skill', label: '技能匹配' },
  { key: 'w_year', label: '经验年限' },
  { key: 'w_salary', label: '薪资匹配' },
  { key: 'w_edu', label: '学历' },
  { key: 'w_city', label: '城市地点' }
]

const depts = ['技术部', '产品部', '设计部', '数据部', '质量部', '市场部']
const cities = ['北京', '上海', '深圳', '杭州', '广州', '成都', '武汉', '南京']

const appCount = pid => store.applications.filter(a => a.position_id === pid).length

// 输入权重为相对值，保存时后端归一化闭合；这里展示归一化后的百分比预览
const weightPct = key => {
  if (!stForm.value) return 0
  const sum = DEFAULT_WEIGHTS.reduce((s, w) => s + (Number(stForm.value[w.key]) || 0), 0)
  return sum > 0 ? Math.round((Number(stForm.value[key]) || 0) / sum * 100) : 0
}
const rawWeightSum = computed(() => stForm.value ? DEFAULT_WEIGHTS.reduce((s, w) => s + (Number(stForm.value[w.key]) || 0), 0) : 0)

function openNew() {
  editing.value = null
  form.value = { name: '', dept: '技术部', city: '上海', level: 'P5', salary_min: 15000, salary_max: 30000, years: 2, slots: 1 }
  skills.value = []
  skillInput.value = ''
  open.value = true
}
function openEdit(p) {
  editing.value = p
  form.value = { name: p.name, dept: p.dept, city: p.city, level: p.level, salary_min: p.salary_min, salary_max: p.salary_max, years: p.years, slots: p.slots }
  skills.value = p.skills.map(s => ({ ...s }))
  skillInput.value = ''
  open.value = true
}
function addSkill() {
  const k = skillInput.value.trim()
  if (k) skills.value.push({ k, w: 5 })
  skillInput.value = ''
}
async function submit() {
  const payload = { ...form.value, skills: skills.value }
  if (editing.value) await store.updatePosition(editing.value.id, payload)
  else await store.addPosition(payload)
  open.value = false
}

async function openStrategy(p) {
  stPos.value = p
  stMsg.value = ''
  const r = await store.getStrategy(p.id)
  const s = r.strategy
  stForm.value = {
    w_skill: s.w_skill, w_year: s.w_year, w_salary: s.w_salary, w_edu: s.w_edu, w_city: s.w_city,
    keyword_bonus_cap: s.keyword_bonus_cap, rec_threshold: s.rec_threshold
  }
  stOpen.value = true
}

// 仅保存配置（暂存，不影响推荐结果）
async function saveOnly() {
  stBusy.value = true
  stMsg.value = ''
  const r = await store.saveStrategy(stPos.value.id, stForm.value, false)
  stBusy.value = false
  stMsg.value = r.ok ? '策略已保存：下次打开推荐页即按此策略计算；点击「发布并重算」可立即刷新全部已落库推荐。' : '保存失败'
}
// 发布策略并批量重算该职位
async function publish() {
  stBusy.value = true
  stMsg.value = ''
  const r = await store.publishStrategy(stPos.value.id, stForm.value)
  stBusy.value = false
  if (r.ok) {
    stMsg.value = `已发布并重算 ${r.matches} 条推荐（${r.positions} 个职位），投递历史评分不受影响。`
    stOpen.value = false
  }
}
async function resetStrategy() {
  stBusy.value = true
  const r = await store.resetStrategy(stPos.value.id)
  if (r.ok && r.strategy) {
    stForm.value = {
      w_skill: r.strategy.w_skill, w_year: r.strategy.w_year, w_salary: r.strategy.w_salary,
      w_edu: r.strategy.w_edu, w_city: r.strategy.w_city,
      keyword_bonus_cap: r.strategy.keyword_bonus_cap, rec_threshold: r.strategy.rec_threshold
    }
    stMsg.value = '已恢复平台默认策略（默认策略重算后生效）。'
  }
  stBusy.value = false
}
// 不改动策略，仅按当前已发布策略重算该职位
async function recalcOne(p, e) {
  e.stopPropagation()
  const btn = e.target
  btn.disabled = true
  await store.recalc({ position_id: p.id })
  btn.disabled = false
}
// 全部在招职位批量重算
async function recalcAll() {
  batchBusy.value = true
  const r = await store.recalc({ scope: 'all' })
  batchBusy.value = false
  window.alert?.(`已按各职位已发布策略批量重算完成：${r.positions} 个职位、${r.matches} 条推荐结果。投递时的历史评分保持不变。`)
}

const strategyOf = p => p.match_strategy || null
</script>

<template>
  <div class="positions">
    <div class="bar">
      <span class="muted">共 {{ store.positions.length }} 个职位 · 在招 {{ store.openPositions.length }}</span>
      <div class="acts">
        <button class="warn" :disabled="batchBusy || !store.openPositions.length" @click="recalcAll">
          🔄 {{ batchBusy ? '重算中…' : '批量重算全部推荐' }}
        </button>
        <button class="primary" @click="openNew">＋ 发布职位</button>
      </div>
    </div>

    <div class="cards">
      <div class="pcard card" v-for="p in store.positions" :key="p.id" :class="{ closed: p.status === 'closed' }">
        <div class="phead">
          <div>
            <b>{{ p.name }}</b>
            <span class="tag" :class="p.status">{{ p.status === 'open' ? '🔥 招聘中' : '⏸ 已关闭' }}</span>
            <span class="tag st" :class="{ custom: strategyOf(p) }">
              {{ strategyOf(p) ? '⚙️ 自定义策略' : '⚙️ 默认策略' }}
            </span>
          </div>
          <div class="meta muted">
            <span>🏢 {{ p.dept }}</span><span>📍 {{ p.city }}</span><span>{{ p.level }}</span>
          </div>
        </div>
        <div class="sal money">¥{{ p.salary_min.toLocaleString() }} - {{ p.salary_max.toLocaleString() }}</div>
        <div class="chips">
          <span class="skill-chip" v-for="s in p.skills" :key="s.k">{{ s.k }} <i>x{{ s.w }}</i></span>
        </div>
        <div class="pmeta muted">
          <span>经验 {{ p.years }} 年+</span><span>编制 {{ p.slots }}</span><span>应聘 {{ appCount(p.id) }}</span>
          <span v-if="p.strategy_published_at" class="pub-at">策略发布 {{ p.strategy_published_at }}</span>
        </div>
        <div class="acts">
          <button class="ghost" @click="openEdit(p)">编辑</button>
          <button class="ghost st-btn" @click="openStrategy(p)">⚙️ 匹配策略</button>
          <button class="ghost" @click="recalcOne(p, $event)">🔄 重算推荐</button>
          <button class="warn" v-if="p.status === 'open'" @click="store.updatePosition(p.id, { status: 'closed' })">关闭职位</button>
          <button class="succ" v-else @click="store.updatePosition(p.id, { status: 'open' })">重新开放</button>
        </div>
      </div>
    </div>

    <!-- 职位新建/编辑 -->
    <div class="modal" v-if="open">
      <div class="modal-box card">
        <h3>{{ editing ? '✏️ 编辑职位' : '📌 发布新职位' }}</h3>
        <div class="form">
          <div class="fg">
            <label>职位名称<input v-model="form.name" placeholder="如 前端开发工程师" /></label>
            <label>部门<select v-model="form.dept"><option v-for="d in depts" :key="d">{{ d }}</option></select></label>
          </div>
          <div class="fg">
            <label>城市<select v-model="form.city"><option v-for="c in cities" :key="c">{{ c }}</option></select></label>
            <label>职级<select v-model="form.level"><option v-for="l in ['P4','P5','P6','P7','P8']" :key="l">{{ l }}</option></select></label>
          </div>
          <div class="fg">
            <label>最低薪资<input type="number" v-model.number="form.salary_min" /></label>
            <label>最高薪资<input type="number" v-model.number="form.salary_max" /></label>
          </div>
          <div class="fg">
            <label>经验年限<input type="number" v-model.number="form.years" /></label>
            <label>编制人数<input type="number" v-model.number="form.slots" /></label>
          </div>
          <label>技能要求（权重5=必备）</label>
          <div class="skill-editor">
            <input v-model="skillInput" placeholder="输入技能回车添加" @keyup.enter="addSkill" />
            <button class="ghost" @click="addSkill">＋ 添加</button>
          </div>
          <div class="edit-chips">
            <span v-for="(s, i) in skills" :key="i" class="chipx">
              {{ s.k }}<select v-model.number="s.w"><option :value="1">1</option><option :value="2">2</option><option :value="3">3</option><option :value="4">4</option><option :value="5">5</option></select>
              <button class="x" @click="skills.splice(i, 1)">✕</button>
            </span>
          </div>
        </div>
        <div class="acts">
          <button class="primary" @click="submit">{{ editing ? '保存' : '发布' }}</button>
          <button class="ghost" @click="open = false">取消</button>
        </div>
      </div>
    </div>

    <!-- 匹配策略配置/发布 -->
    <div class="modal" v-if="stOpen">
      <div class="modal-box card st-box">
        <h3>⚙️ 匹配策略 · {{ stPos?.name }}
          <span class="tag" :class="{ custom: strategyOf(stPos) }">{{ strategyOf(stPos) ? '自定义已发布' : '平台默认' }}</span>
        </h3>
        <div class="muted st-tip">调整五个维度的相对权重，保存时自动归一化为 100%。「发布并重算」会立即按新策略批量重算该职位已落库的全部推荐；之后打开推荐页也始终使用新策略。候选人投递时冻结的历史评分永久保留、不受影响。</div>

        <div class="w-list">
          <div class="w-row" v-for="w in DEFAULT_WEIGHTS" :key="w.key">
            <span class="w-label">{{ w.label }}</span>
            <input class="w-range" type="range" min="0" max="100" step="5" v-model.number="stForm[w.key]" />
            <input class="w-num" type="number" min="0" max="1" step="0.05" v-model.number="stForm[w.key]" />
            <b class="w-pct">{{ weightPct(w.key) }}%</b>
          </div>
        </div>
        <div class="muted" v-if="rawWeightSum === 0">⚠️ 五项权重不能全为 0，保存时将回退默认权重。</div>

        <div class="fg st-fg">
          <label>简历关键词加分上限（分）
            <input type="number" min="0" max="20" step="0.5" v-model.number="stForm.keyword_bonus_cap" />
          </label>
          <label>推荐分数线（达到即标记推荐）
            <input type="number" min="0" max="100" v-model.number="stForm.rec_threshold" />
          </label>
        </div>

        <div class="st-preview" v-if="strategyOf(stPos)">
          <span class="muted">当前已发布权重：</span>
          <span class="skill-chip" v-for="w in DEFAULT_WEIGHTS" :key="w.key">
            {{ w.label }} {{ Math.round((strategyOf(stPos)[w.key] || 0) * 100) }}%
          </span>
        </div>

        <div class="muted st-msg" v-if="stMsg">{{ stMsg }}</div>
        <div class="acts">
          <button class="primary" :disabled="stBusy" @click="publish">🚀 发布并重算推荐</button>
          <button class="ghost" :disabled="stBusy" @click="saveOnly">仅暂存配置</button>
          <button class="warn" :disabled="stBusy" @click="resetStrategy">恢复默认策略</button>
          <button class="ghost" @click="stOpen = false">关闭</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.positions { display: flex; flex-direction: column; gap: 14px; }
.bar { display: flex; justify-content: space-between; align-items: center; }
.cards { display: grid; grid-template-columns: repeat(auto-fill, minmax(340px, 1fr)); gap: 14px; }
.pcard { display: flex; flex-direction: column; gap: 10px; }
.pcard.closed { opacity: .6; }
.phead { display: flex; justify-content: space-between; align-items: flex-start; }
.phead b { font-size: 16px; }
.meta { display: flex; gap: 8px; font-size: 12px; }
.sal { font-size: 15px; }
.chips { display: flex; flex-wrap: wrap; }
.chips i { font-style: normal; opacity: .7; font-size: 10px; }
.pmeta { display: flex; gap: 12px; font-size: 12px; flex-wrap: wrap; }
.pmeta .pub-at { color: var(--accent2); }
.tag.open { background: rgba(87,214,160,.15); color: var(--green); border-color: rgba(87,214,160,.4); }
.tag.closed { background: rgba(140,149,176,.1); }
.tag.st.custom { background: rgba(167,139,250,.16); color: var(--purple); border-color: rgba(167,139,250,.4); }
.acts { display: flex; gap: 6px; flex-wrap: wrap; }
.st-btn { color: var(--purple); }
.form { display: flex; flex-direction: column; gap: 10px; margin: 14px 0; }
.fg { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
.form label { display: flex; flex-direction: column; gap: 5px; font-size: 13px; color: var(--muted); }
.skill-editor { display: flex; gap: 8px; }
.skill-editor input { flex: 1; }
.edit-chips { display: flex; flex-wrap: wrap; gap: 6px; }
.chipx { display: inline-flex; align-items: center; gap: 4px; background: rgba(91,140,255,.14); border: 1px solid rgba(91,140,255,.35); padding: 2px 6px; border-radius: 10px; font-size: 12px; }
.chipx select { background: transparent; border: none; color: var(--accent2); width: 34px; }
.chipx .x { background: none; border: none; color: var(--muted); cursor: pointer; font-size: 11px; padding: 0 2px; }
.st-box { width: min(600px, 94vw); }
.st-tip { line-height: 1.6; margin-bottom: 12px; }
.w-list { display: flex; flex-direction: column; gap: 10px; margin: 6px 0 12px; }
.w-row { display: grid; grid-template-columns: 76px 1fr 74px 48px; align-items: center; gap: 10px; font-size: 13px; }
.w-range { padding: 0; accent-color: var(--accent); }
.w-num { padding: 5px 8px; }
.w-pct { color: var(--accent2); text-align: right; }
.st-fg { margin-bottom: 10px; }
.st-fg label { font-size: 12px; gap: 6px; }
.st-preview { display: flex; flex-wrap: wrap; align-items: center; gap: 4px; padding: 8px 10px; background: var(--panel2); border-radius: 8px; margin-bottom: 10px; }
.st-msg { line-height: 1.6; margin-bottom: 8px; color: var(--green); }
</style>
