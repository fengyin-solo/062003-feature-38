<template>
  <div class="trainee-card card" :class="statusClass">
    <div class="card-top">
      <div class="name-area">
        <h4>
          {{ trainee.name }}
          <span v-if="trainee.isFocused" class="focus-star" title="重点培养">⭐</span>
        </h4>
        <div class="plan-target" v-if="trainee.personalPlan && !editingPlan">
          <span class="target-label">{{ statLabels[trainee.personalPlan.targetStat] }}</span>
          <span class="target-value">{{ trainee.stats[trainee.personalPlan.targetStat] }}/{{ trainee.personalPlan.targetValue }}</span>
        </div>
      </div>
      <div class="badges">
        <span class="badge" :class="trainee.status">{{ statusLabel }}</span>
        <button
          v-if="trainee.status === 'trainee'"
          class="focus-btn"
          :class="{ focused: trainee.isFocused }"
          :title="trainee.isFocused ? '取消重点培养' : '设为重点培养'"
          @click="$emit('toggle-focus')"
        >
          {{ trainee.isFocused ? '★' : '☆' }}
        </button>
      </div>
    </div>

    <div v-if="editingPlan" class="plan-editor">
      <div class="plan-row">
        <select v-model="newTargetStat">
          <option v-for="key in statKeys" :key="key" :value="key">
            {{ statLabels[key] }}
          </option>
        </select>
        <input
          type="number"
          v-model.number="newTargetValue"
          min="30"
          max="100"
          class="target-input"
        />
        <button class="btn tiny primary" @click="savePlan">保存</button>
        <button class="btn tiny ghost" @click="cancelEdit">取消</button>
      </div>
    </div>

    <div v-else-if="trainee.personalPlan && trainee.status !== 'left'" class="plan-bar">
      <div class="plan-bar-label">
        <span>专属计划</span>
        <button
          v-if="trainee.status === 'trainee'"
          class="edit-link"
          @click="startEdit"
        >修改</button>
      </div>
      <div class="plan-bar-track">
        <div
          class="plan-bar-fill"
          :style="{ width: planProgressPercent + '%' }"
        ></div>
        <div
          v-for="ms in milestones"
          :key="ms.label"
          class="milestone-dot"
          :class="{ achieved: trainee.personalPlan.achievedMilestones.includes(ms.label) }"
          :style="{ left: (ms.statTarget / 100 * 100) + '%' }"
          :title="`${ms.label} (${ms.statTarget}) - ${ms.reward}`"
        ></div>
      </div>
      <div class="milestone-tags">
        <span
          v-for="ms in milestones"
          :key="ms.label"
          class="ms-tag"
          :class="{ achieved: trainee.personalPlan.achievedMilestones.includes(ms.label) }"
          :title="ms.reward"
        >
          {{ trainee.personalPlan.achievedMilestones.includes(ms.label) ? '✓' : '○' }} {{ ms.label }}
        </span>
      </div>
    </div>

    <div class="bars">
      <div class="bar-row">
        <span>疲劳</span>
        <div class="bar"><div class="fill fatigue" :style="{ width: trainee.fatigue + '%' }"></div></div>
        <span>{{ trainee.fatigue }}</span>
      </div>
      <div class="bar-row">
        <span>压力</span>
        <div class="bar"><div class="fill stress" :style="{ width: trainee.stress + '%' }"></div></div>
        <span>{{ trainee.stress }}</span>
      </div>
    </div>

    <div class="stats-grid">
      <div
        v-for="key in statKeys"
        :key="key"
        class="stat-cell"
        :class="{ 'target-stat': trainee.personalPlan?.targetStat === key }"
      >
        <span class="stat-label">{{ statLabels[key] }}</span>
        <span class="stat-val">{{ trainee.stats[key] }}</span>
      </div>
    </div>

    <div v-if="score !== null" class="score">
      综合评分 <strong>{{ score }}</strong>
      <span v-if="trainee.status === 'trainee'" class="debut-hint">
        {{ score >= debutThreshold ? '✓ 可出道' : `需 ${debutThreshold}` }}
      </span>
    </div>

    <div v-if="trainee.illnessDays > 0" class="illness">🤒 休养中 ({{ trainee.illnessDays }}天)</div>
    <div v-if="trainee.fans > 0" class="fans">个人粉丝 {{ trainee.fans.toLocaleString() }}</div>
    <div v-if="trainee.isFocused" class="focus-tip">💫 重点培养：训练 +35%，疲劳/压力略增</div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { GAME_CONFIG } from '../config/gameConfig'

const props = defineProps({
  trainee: Object,
  score: { type: Number, default: null },
})

defineEmits(['toggle-focus', 'update-plan'])

const statKeys = GAME_CONFIG.stats
const statLabels = GAME_CONFIG.statLabels
const debutThreshold = GAME_CONFIG.rating.debutScoreThreshold
const milestones = GAME_CONFIG.personalPlan.milestones

const editingPlan = ref(false)
const newTargetStat = ref('')
const newTargetValue = ref(70)

const statusLabel = computed(() => {
  const map = { trainee: '练习生', debuted: '已出道', left: '已离开' }
  return map[props.trainee.status] || props.trainee.status
})

const statusClass = computed(() => ({
  debuted: props.trainee.status === 'debuted',
  left: props.trainee.status === 'left',
  ill: props.trainee.illnessDays > 0,
  focused: props.trainee.isFocused,
}))

const planProgressPercent = computed(() => {
  if (!props.trainee.personalPlan) return 0
  const { targetStat, targetValue } = props.trainee.personalPlan
  const cur = props.trainee.stats[targetStat]
  return Math.min(100, Math.round(cur / targetValue * 100))
})

function startEdit() {
  newTargetStat.value = props.trainee.personalPlan.targetStat
  newTargetValue.value = props.trainee.personalPlan.targetValue
  editingPlan.value = true
}

function cancelEdit() {
  editingPlan.value = false
}

function savePlan() {
  if (newTargetValue.value < 30) newTargetValue.value = 30
  if (newTargetValue.value > 100) newTargetValue.value = 100
  props.$emit('update-plan', newTargetStat.value, newTargetValue.value)
  editingPlan.value = false
}
</script>

<style scoped>
.trainee-card {
  padding: 1rem;
  transition: border-color 0.2s, box-shadow 0.2s;
}

.trainee-card.debuted { border-color: var(--accent); }
.trainee-card.left { opacity: 0.5; }
.trainee-card.ill { border-color: var(--warning); }
.trainee-card.focused {
  border-color: gold;
  box-shadow: 0 0 0 1px rgba(255, 215, 0, 0.3), 0 2px 12px rgba(255, 215, 0, 0.15);
}

.card-top {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 0.5rem;
}

.name-area h4 {
  font-size: 1rem;
  margin: 0;
}

.focus-star {
  color: gold;
  font-size: 0.9rem;
  margin-left: 0.2rem;
}

.plan-target {
  display: flex;
  gap: 0.35rem;
  font-size: 0.72rem;
  color: var(--text-muted);
  margin-top: 0.15rem;
}

.target-label {
  background: var(--accent-soft);
  color: var(--accent);
  padding: 0.05rem 0.4rem;
  border-radius: 4px;
  font-weight: 600;
}

.target-value {
  font-family: monospace;
  color: var(--text-secondary);
}

.badges {
  display: flex;
  align-items: center;
  gap: 0.4rem;
}

.badge {
  font-size: 0.7rem;
  padding: 0.15rem 0.5rem;
  border-radius: 999px;
  background: var(--bg-secondary);
}

.badge.debuted { background: var(--accent-soft); color: var(--accent); }
.badge.left { background: var(--danger-soft); color: var(--danger); }

.focus-btn {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  border: 1px solid var(--border);
  background: var(--bg-secondary);
  cursor: pointer;
  font-size: 0.9rem;
  transition: all 0.15s;
  color: var(--text-muted);
  display: flex;
  align-items: center;
  justify-content: center;
  line-height: 1;
}

.focus-btn:hover { border-color: gold; color: gold; }
.focus-btn.focused {
  background: gold;
  border-color: gold;
  color: #fff;
}

.plan-editor {
  background: var(--bg-secondary);
  border-radius: 8px;
  padding: 0.6rem;
  margin-bottom: 0.75rem;
}

.plan-row {
  display: flex;
  gap: 0.4rem;
  align-items: center;
  flex-wrap: wrap;
}

.plan-row select,
.target-input {
  padding: 0.3rem 0.45rem;
  border: 1px solid var(--border);
  border-radius: 6px;
  background: var(--bg-card);
  color: var(--text);
  font-size: 0.8rem;
}

.target-input {
  width: 68px;
}

.btn.tiny {
  padding: 0.25rem 0.55rem;
  font-size: 0.75rem;
}

.btn.primary {
  background: var(--accent);
  color: #fff;
  border: 1px solid var(--accent);
  border-radius: 6px;
  cursor: pointer;
}

.btn.ghost {
  background: transparent;
  color: var(--text-secondary);
  border: 1px solid var(--border);
  border-radius: 6px;
  cursor: pointer;
}

.plan-bar {
  margin-bottom: 0.75rem;
}

.plan-bar-label {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.72rem;
  color: var(--text-muted);
  margin-bottom: 0.25rem;
}

.edit-link {
  background: none;
  border: none;
  color: var(--accent);
  font-size: 0.7rem;
  cursor: pointer;
  padding: 0;
}

.edit-link:hover { text-decoration: underline; }

.plan-bar-track {
  position: relative;
  height: 8px;
  background: var(--bg-secondary);
  border-radius: 4px;
  overflow: visible;
  margin-bottom: 0.4rem;
}

.plan-bar-fill {
  position: absolute;
  left: 0;
  top: 0;
  height: 100%;
  background: linear-gradient(90deg, var(--accent), gold);
  border-radius: 4px;
  transition: width 0.3s;
}

.milestone-dot {
  position: absolute;
  top: 50%;
  transform: translate(-50%, -50%);
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background: var(--bg-card);
  border: 2px solid var(--border);
  z-index: 1;
  transition: all 0.2s;
}

.milestone-dot.achieved {
  background: gold;
  border-color: gold;
  box-shadow: 0 0 0 2px var(--bg-card);
}

.milestone-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.3rem;
}

.ms-tag {
  font-size: 0.62rem;
  padding: 0.1rem 0.35rem;
  border-radius: 4px;
  background: var(--bg-secondary);
  color: var(--text-muted);
}

.ms-tag.achieved {
  background: var(--accent-soft);
  color: var(--accent);
  font-weight: 600;
}

.bars { margin-bottom: 0.75rem; }

.bar-row {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.75rem;
  margin-bottom: 0.25rem;
}

.bar-row span:first-child { width: 28px; color: var(--text-muted); }
.bar-row span:last-child { width: 24px; text-align: right; }

.bar {
  flex: 1;
  height: 6px;
  background: var(--bg-secondary);
  border-radius: 3px;
  overflow: hidden;
}

.fill { height: 100%; border-radius: 3px; transition: width 0.3s; }
.fill.fatigue { background: var(--warning); }
.fill.stress { background: var(--danger); }

.stats-grid {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 0.25rem;
  text-align: center;
}

.stat-cell {
  background: var(--bg-secondary);
  border-radius: 6px;
  padding: 0.3rem 0.1rem;
  transition: all 0.2s;
  border: 1px solid transparent;
}

.stat-cell.target-stat {
  background: var(--accent-soft);
  border-color: var(--accent);
}

.stat-label { display: block; font-size: 0.65rem; color: var(--text-muted); }
.stat-cell.target-stat .stat-label { color: var(--accent); }
.stat-val { font-weight: 700; font-size: 0.85rem; }

.score {
  margin-top: 0.5rem;
  font-size: 0.85rem;
  color: var(--text-secondary);
}

.debut-hint {
  margin-left: 0.5rem;
  font-size: 0.75rem;
  color: var(--accent);
}

.illness, .fans, .focus-tip {
  margin-top: 0.35rem;
  font-size: 0.8rem;
}

.illness { color: var(--warning); }
.fans { color: var(--warning); }
.focus-tip {
  color: gold;
  font-weight: 500;
}
</style>
