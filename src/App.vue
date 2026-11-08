<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'
import { Check, Crown, History, RotateCcw, Sparkles, Trophy, X } from '@lucide/vue'

type PlayerId = 'north' | 'south'
type Player = { id: PlayerId; name: string; score: number; bet: number }
type RoundRecord = {
  id: number
  winner: PlayerId
  winnerName: string
  northName: string
  southName: string
  northBet: number
  southBet: number
  transfer: number
  time: string
}
type SavedGame = { initialScore: number; players: Player[]; records: RoundRecord[] }

const STORAGE_KEY = 'jin-hua-scoreboard-v1'
const chipValues = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
const emptyGame: SavedGame = {
  initialScore: 0,
  players: [
    { id: 'north', name: '上家', score: 0, bet: 0 },
    { id: 'south', name: '下家', score: 0, bet: 0 },
  ],
  records: [],
}

const game = ref<SavedGame>(structuredClone(emptyGame))
const isStarted = ref(false)
const showScoreDialog = ref(true)
const scoreDialogMode = ref<'start' | 'reset'>('start')
const nextInitialScore = ref('')
const scoreError = ref('')
const renaming = ref<PlayerId | null>(null)
const nameDraft = ref('')
const toast = ref('')
let toastTimer: ReturnType<typeof setTimeout> | undefined

const north = computed(() => game.value.players[0])
const south = computed(() => game.value.players[1])
const currentPot = computed(() => north.value.bet + south.value.bet)
const recentRecords = computed(() => game.value.records)
const leader = computed(() => {
  if (north.value.score === south.value.score) return null
  return north.value.score > south.value.score ? 'north' : 'south'
})

function loadGame(): boolean {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    if (!raw) return false
    const saved = JSON.parse(raw) as SavedGame
    const valid =
      typeof saved.initialScore === 'number' && saved.initialScore > 0 &&
      Array.isArray(saved.players) && saved.players.length === 2 &&
      saved.players.every((player) => typeof player.score === 'number' && typeof player.bet === 'number') &&
      Array.isArray(saved.records)
    if (!valid) return false
    if ('round' in saved) {
      saved.players = saved.players.map((player) => {
        const balance = Math.max(0, player.score - player.bet)
        const ante = balance >= 1 ? 1 : 0
        return { ...player, score: balance - ante, bet: player.bet + ante }
      })
    }
    saved.records = saved.records.map((record) => ({
      ...record,
      northName: record.northName || saved.players[0].name || '上家',
      southName: record.southName || saved.players[1].name || '下家',
    }))
    game.value = saved
    isStarted.value = true
    return true
  } catch {
    localStorage.removeItem(STORAGE_KEY)
    return false
  }
}

onMounted(() => {
  if (loadGame()) {
    showScoreDialog.value = false
    scoreDialogMode.value = 'reset'
  }
})

watch(game, (value) => {
  if (isStarted.value) localStorage.setItem(STORAGE_KEY, JSON.stringify(value))
}, { deep: true })

function notify(message: string) {
  toast.value = message
  if (toastTimer) clearTimeout(toastTimer)
  toastTimer = setTimeout(() => (toast.value = ''), 2200)
}

function addBet(playerId: PlayerId, amount: number) {
  const player = game.value.players.find((item) => item.id === playerId)!
  if (amount > player.score) {
    notify('下注不能超过当前点数')
    return
  }
  player.score -= amount
  player.bet += amount
}

function postAnte(player: Player) {
  if (player.score >= 1) {
    player.score -= 1
    player.bet = 1
  } else {
    player.bet = 0
  }
}

function settle(winnerId: PlayerId) {
  const winner = game.value.players.find((player) => player.id === winnerId)!
  const transfer = currentPot.value
  const round: RoundRecord = {
    id: Date.now(),
    winner: winnerId,
    winnerName: winner.name,
    northName: north.value.name,
    southName: south.value.name,
    northBet: north.value.bet,
    southBet: south.value.bet,
    transfer,
    time: new Intl.DateTimeFormat('zh-CN', { hour: '2-digit', minute: '2-digit' }).format(new Date()),
  }

  winner.score += transfer
  game.value.records.unshift(round)
  north.value.bet = 0
  south.value.bet = 0
  postAnte(north.value)
  postAnte(south.value)
  notify(transfer > 0 ? `${winner.name} 赢下 ${transfer} 点` : `${winner.name} 获胜，本局未产生转分`)
}

function beginRename(player: Player) {
  renaming.value = player.id
  nameDraft.value = player.name
}

function saveName(playerId: PlayerId) {
  const cleanName = nameDraft.value.trim().slice(0, 8)
  if (cleanName) game.value.players.find((player) => player.id === playerId)!.name = cleanName
  renaming.value = null
}

function openReset() {
  scoreDialogMode.value = 'reset'
  nextInitialScore.value = ''
  scoreError.value = ''
  showScoreDialog.value = true
}

function startMatch() {
  const parsedScore = Number(nextInitialScore.value)
  if (!Number.isFinite(parsedScore) || parsedScore < 1) {
    scoreError.value = '请输入至少 1 点的初始分数'
    return
  }
  const score = Math.min(999999, Math.floor(parsedScore))
  const [northName, southName] = [north.value.name, south.value.name]
  game.value = {
    initialScore: score,
    players: [
      { id: 'north', name: northName, score, bet: 0 },
      { id: 'south', name: southName, score, bet: 0 },
    ],
    records: [],
  }
  postAnte(north.value)
  postAnte(south.value)
  isStarted.value = true
  scoreDialogMode.value = 'reset'
  showScoreDialog.value = false
  scoreError.value = ''
  nextInitialScore.value = ''
  notify(`牌局开始，双方各有 ${score} 点`)
}

function closeScoreDialog() {
  if (scoreDialogMode.value === 'reset') showScoreDialog.value = false
}

function formatNumber(value: number) {
  return new Intl.NumberFormat('zh-CN').format(value)
}
</script>

<template>
  <main class="app-shell">
    <section class="player-panel player-panel--north" aria-label="上家操作区">
      <div class="player-heading">
        <div class="player-title">
          <span class="seat-badge">上</span>
          <template v-if="renaming === 'north'">
            <form class="rename-form" @submit.prevent="saveName('north')">
              <input v-model="nameDraft" aria-label="修改上家名字" maxlength="8" autofocus />
              <button class="small-icon" type="submit" aria-label="保存名字"><Check :size="16" /></button>
            </form>
          </template>
          <button v-else class="name-button" @click="beginRename(north)">{{ north.name }} <span>编辑</span></button>
          <Crown v-if="leader === 'north'" :size="15" class="player-crown" aria-label="当前领先" />
        </div>
        <div class="player-total"><span>可用积分</span><strong>{{ formatNumber(north.score) }}</strong></div>
      </div>
      <div class="chip-row" aria-label="上家下注点数">
        <button
          v-for="amount in chipValues"
          :key="`north-${amount}`"
          class="chip-button"
          :disabled="north.bet + amount > north.score"
          @click="addBet('north', amount)"
        >{{ amount }}<small>点</small></button>
      </div>
      <div class="player-actions">
        <button class="win-button win-button--north" @click="settle('north')"><Trophy :size="17" /> {{ north.name }}赢了</button>
      </div>
    </section>

    <section class="scoreboard" aria-label="下注情况和对局记录">
      <div class="pot-line">
        <div class="pot-copy"><span class="pot-icon"><Sparkles :size="15" /></span><span>总下注</span><strong>{{ formatNumber(currentPot) }}<small>点</small></strong></div>
        <button class="reset-button" :disabled="!isStarted" @click="openReset"><RotateCcw :size="14" /> 重置</button>
      </div>

      <div class="bet-summary" aria-label="双方本局下注">
        <div class="bet-summary-side bet-summary-side--north"><span>{{ north.name }}下注</span><strong>{{ formatNumber(north.bet) }}<small>点</small></strong></div>
        <span class="bet-divider">·</span>
        <div class="bet-summary-side bet-summary-side--south"><span>{{ south.name }}下注</span><strong>{{ formatNumber(south.bet) }}<small>点</small></strong></div>
      </div>

      <div class="history-heading"><div><History :size="15" /><strong>对局记录</strong></div></div>
      <div v-if="recentRecords.length" class="history-list">
        <article v-for="record in recentRecords" :key="record.id" class="history-item">
          <span class="history-winner"><Trophy :size="13" /></span>
          <div class="history-main">
            <strong>{{ record.winnerName }} 获胜</strong>
            <span>{{ record.time }} · {{ record.northName }} {{ record.northBet }} / {{ record.southName }} {{ record.southBet }}</span>
          </div>
          <span class="history-transfer">+{{ formatNumber(record.transfer) }}</span>
        </article>
      </div>
      <div v-else class="empty-history"><History :size="15" /><span>每局结束后会自动记录</span></div>
    </section>

    <section class="player-panel player-panel--south" aria-label="下家操作区">
      <div class="player-heading">
        <div class="player-title">
          <span class="seat-badge">下</span>
          <template v-if="renaming === 'south'">
            <form class="rename-form" @submit.prevent="saveName('south')">
              <input v-model="nameDraft" aria-label="修改下家名字" maxlength="8" autofocus />
              <button class="small-icon" type="submit" aria-label="保存名字"><Check :size="16" /></button>
            </form>
          </template>
          <button v-else class="name-button" @click="beginRename(south)">{{ south.name }} <span>编辑</span></button>
          <Crown v-if="leader === 'south'" :size="15" class="player-crown" aria-label="当前领先" />
        </div>
        <div class="player-total"><span>可用积分</span><strong>{{ formatNumber(south.score) }}</strong></div>
      </div>
      <div class="chip-row" aria-label="下家下注点数">
        <button
          v-for="amount in chipValues"
          :key="`south-${amount}`"
          class="chip-button"
          :disabled="south.bet + amount > south.score"
          @click="addBet('south', amount)"
        >{{ amount }}<small>点</small></button>
      </div>
      <div class="player-actions">
        <button class="win-button win-button--south" @click="settle('south')"><Trophy :size="17" /> {{ south.name }}赢了</button>
      </div>
    </section>

    <Transition name="toast"><div v-if="toast" class="toast-message"><Check :size="16" />{{ toast }}</div></Transition>

    <Transition name="modal">
      <div v-if="showScoreDialog" class="modal-backdrop" @click.self="closeScoreDialog">
        <section class="modal-card" role="dialog" aria-modal="true" aria-labelledby="score-dialog-title">
          <div class="modal-top">
            <span class="modal-icon"><Sparkles :size="20" /></span>
            <button v-if="scoreDialogMode === 'reset'" class="modal-close" aria-label="取消重置" @click="closeScoreDialog"><X :size="19" /></button>
          </div>
          <h2 id="score-dialog-title">{{ scoreDialogMode === 'start' ? '开始新牌局' : '重置并开始新牌局' }}</h2>
          <p class="modal-intro">{{ scoreDialogMode === 'start' ? '先设置双方相同的起始分数。开始后双方自动各压 1 点。' : '双方分数将重新相同并各压 1 点，当前下注与对局记录会清空。' }}</p>
          <form @submit.prevent="startMatch">
            <label class="score-input-label" for="initial-score">双方初始分数</label>
            <div class="score-input-wrap"><input id="initial-score" v-model="nextInitialScore" type="number" min="1" max="999999" step="1" inputmode="numeric" placeholder="请输入分数" autofocus /><span>点 / 人</span></div>
            <p v-if="scoreError" class="score-error" role="alert">{{ scoreError }}</p>
            <div class="quick-scores"><button v-for="amount in [50, 100, 200, 500]" :key="amount" type="button" :class="{ active: Number(nextInitialScore) === amount }" @click="nextInitialScore = String(amount)">{{ amount }}</button></div>
            <button class="modal-primary" type="submit"><Trophy :size="16" /> {{ scoreDialogMode === 'start' ? '开始计分' : '确认重置' }}</button>
          </form>
        </section>
      </div>
    </Transition>
  </main>
</template>
