<template>
  <div class="game-view">
    <GameHeader
      :state="state"
      :days-left="daysLeft"
      :profit="profit"
      :theme="theme"
      @back="$emit('back')"
      @toggle-theme="$emit('toggle-theme')"
    />

    <div class="game-body">
      <aside class="sidebar">
        <div class="focus-info" v-if="focusedCount !== undefined">
          <span>重点培养 {{ focusedCount }}/{{ maxFocus }}</span>
          <span class="focus-hint">限{{ maxFocus }}人，训练+35%</span>
        </div>
        <div class="trainee-grid">
          <TraineeCard
            v-for="t in activeTrainees"
            :key="t.id"
            :trainee="t"
            :score="calcScore(t)"
            @toggle-focus="$emit('toggle-focus', t.id)"
            @update-plan="(stat, val) => $emit('update-plan', t.id, stat, val)"
          />
        </div>
      </aside>

      <main class="main-panel">
        <SchedulePanel
          :trainees="activeTrainees"
          :schedule="state.schedule"
          :can-end="canEndDay"
          @set="(id, act) => $emit('set-schedule', id, act)"
          @clear="$emit('clear-schedule')"
          @end-day="$emit('end-day')"
        />
        <DayLog :logs="state.logs" />
      </main>

      <aside class="right-panel">
        <GroupsPanel
          :groups="state.groups"
          :trainees="state.trainees"
          :money="state.money"
          @release="(id) => $emit('release-single', id)"
        />
        <RelationshipPanel
          :trainees="state.trainees"
          :relationships="state.relationships"
        />
      </aside>
    </div>

    <RatingModal
      v-if="state.pendingRating && state.gameStatus === 'playing'"
      :results="ratingResults"
      @close="$emit('dismiss-rating')"
      @debut="showDebut = true"
    />

    <DebutModal
      v-if="showDebut"
      :candidates="ratingResults"
      @close="showDebut = false"
      @confirm="onDebut"
    />

    <EventModal
      v-if="state.pendingEvent"
      :event="state.pendingEvent"
      @resolve="(keep) => $emit('resolve-poaching', keep)"
    />

    <GameOverModal
      v-if="state.gameStatus !== 'playing'"
      :status="state.gameStatus"
      :state="state"
      :profit="profit"
      @back="$emit('back')"
    />

    <div v-if="toast" class="toast" :class="toastType">{{ toast }}</div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import { GAME_CONFIG } from '../config/gameConfig'
import GameHeader from './GameHeader.vue'
import TraineeCard from './TraineeCard.vue'
import SchedulePanel from './SchedulePanel.vue'
import DayLog from './DayLog.vue'
import GroupsPanel from './GroupsPanel.vue'
import RelationshipPanel from './RelationshipPanel.vue'
import RatingModal from './RatingModal.vue'
import DebutModal from './DebutModal.vue'
import EventModal from './EventModal.vue'
import GameOverModal from './GameOverModal.vue'

const props = defineProps({
  state: Object,
  activeTrainees: Array,
  daysLeft: Number,
  profit: Number,
  theme: String,
  canEndDay: Boolean,
  ratingResults: Array,
  calcScore: Function,
  focusedCount: Number,
})

const maxFocus = computed(() => GAME_CONFIG.focus.maxFocusCount)

const emit = defineEmits([
  'back',
  'toggle-theme',
  'set-schedule',
  'clear-schedule',
  'end-day',
  'dismiss-rating',
  'debut',
  'resolve-poaching',
  'release-single',
  'toggle-focus',
  'update-plan',
])

const showDebut = ref(false)
const toast = ref('')
const toastType = ref('normal')
let toastTimer = null
let lastLogCount = 0

watch(
  () => props.state?.logs?.length,
  (newLen, oldLen) => {
    if (!props.state?.logs) return
    oldLen = oldLen || lastLogCount || 0
    if (newLen <= oldLen) {
      lastLogCount = newLen
      return
    }
    const newLogs = props.state.logs.slice(oldLen, newLen)
    lastLogCount = newLen

    const achievementLogs = newLogs.filter((l) => l.text.includes('🏆'))
    const planCompleteLogs = newLogs.filter((l) =>
      l.text.includes('预备王牌')
    )

    if (planCompleteLogs.length > 0) {
      const first = planCompleteLogs[0]
      showToast(`🎉 ${first.text.replace('🏆', '').split('！')[0]}！巅峰达成！`, 'success')
    } else if (achievementLogs.length > 0) {
      const first = achievementLogs[0]
      const match = first.text.match(/【(.+?)】(.+?)的(.+?)达成(.+?)（/)
      if (match) {
        showToast(`🌟 ${match[2]} 的${match[3]}达成${match[4]}！获得【${match[1]}】奖励！`, 'milestone')
      } else {
        showToast(first.text, 'milestone')
      }
    }

    for (const t of props.activeTrainees || []) {
      if (!t.personalPlan) continue
      const stat = t.stats[t.personalPlan.targetStat]
      const target = t.personalPlan.targetValue
      if (
        stat >= target &&
        !t.personalPlan.notifiedMilestones?.includes(`plan_complete_${target}`)
      ) {
        t.personalPlan.notifiedMilestones = [
          ...(t.personalPlan.notifiedMilestones || []),
          `plan_complete_${target}`,
        ]
        showToast(`🎯 ${t.name} 的专属计划目标达成！${GAME_CONFIG.statLabels[t.personalPlan.targetStat]} ${stat}/${target}`, 'plan')
      }
    }
  },
  { immediate: true }
)

watch(
  () => props.state?.day,
  () => {
    if (props.state?.trainees) {
      for (const t of props.state.trainees) {
        if (!t.personalPlan?.notifiedMilestones) {
          t.personalPlan = {
            ...t.personalPlan,
            notifiedMilestones: [],
          }
        }
      }
    }
  },
  { immediate: true }
)

function showToast(message, type = 'normal', duration = 3500) {
  if (toastTimer) clearTimeout(toastTimer)
  toast.value = message
  toastType.value = type
  toastTimer = setTimeout(() => {
    toast.value = ''
  }, duration)
}

function onDebut(memberIds, groupName) {
  emit('debut', memberIds, groupName, (result) => {
    if (result?.success) {
      showDebut.value = false
      showToast('出道成功！', 'success')
    } else if (result?.message) {
      showToast(result.message, 'error', 3000)
    }
  })
}
</script>

<style scoped>
.game-view {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.game-body {
  display: grid;
  grid-template-columns: 1fr 1.1fr 0.9fr;
  gap: 1rem;
  padding: 1rem;
  flex: 1;
}

@media (max-width: 1100px) {
  .game-body {
    grid-template-columns: 1fr;
  }
}

.sidebar .trainee-grid {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.focus-info {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: linear-gradient(135deg, rgba(255, 215, 0, 0.1), rgba(255, 215, 0, 0.05));
  border: 1px solid rgba(255, 215, 0, 0.3);
  border-radius: 8px;
  padding: 0.5rem 0.75rem;
  margin-bottom: 0.75rem;
  font-size: 0.8rem;
}

.focus-info > span:first-child {
  font-weight: 700;
  color: #b8860b;
}

.focus-hint {
  color: var(--text-muted);
  font-size: 0.72rem;
}

.main-panel,
.right-panel {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.toast {
  position: fixed;
  bottom: 2rem;
  left: 50%;
  transform: translateX(-50%);
  background: var(--bg-card);
  border: 1px solid var(--accent);
  padding: 0.75rem 1.5rem;
  border-radius: 8px;
  z-index: 200;
  box-shadow: var(--shadow);
  max-width: 90vw;
  text-align: center;
  animation: toastIn 0.3s ease-out;
}

@keyframes toastIn {
  from {
    opacity: 0;
    transform: translateX(-50%) translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
  }
}

.toast.success {
  background: linear-gradient(135deg, #1a472a, #2d5a3f);
  border-color: #4ade80;
  color: #bbf7d0;
}

.toast.milestone {
  background: linear-gradient(135deg, #422006, #713f12);
  border-color: #fbbf24;
  color: #fef3c7;
  font-weight: 600;
}

.toast.plan {
  background: linear-gradient(135deg, #1e1b4b, #312e81);
  border-color: #818cf8;
  color: #c7d2fe;
  font-weight: 600;
}

.toast.error {
  background: linear-gradient(135deg, #450a0a, #7f1d1d);
  border-color: #f87171;
  color: #fecaca;
}
</style>
