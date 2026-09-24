<template>
  <div class="page">
    <!-- 日历视图 -->
    <template v-if="!selectedDate">
      <div class="page-head">
        <div>
          <h2>收支日历</h2>
          <p class="page-sub">按天回顾收入与支出，点击任意日期查看当天明细</p>
        </div>
      </div>

      <div class="card cal-card">
        <div class="cal-head">
          <button class="icon-btn month-btn" @click="changeMonth(-1)" title="上一月" aria-label="上一月">‹</button>
          <div class="month-title">{{ titleMonth }}</div>
          <button class="icon-btn month-btn" @click="changeMonth(1)" title="下一月" aria-label="下一月">›</button>
          <button v-if="!isViewingCurrentMonth" class="btn btn-today" @click="goToday">回到本月</button>
        </div>

        <div class="month-summary">
          <span>本月收入 <b class="inc">¥{{ money(monthTotals.income) }}</b></span>
          <span>本月支出 <b class="exp">¥{{ money(monthTotals.expense) }}</b></span>
          <span>结余 <b :class="monthTotals.income - monthTotals.expense < 0 ? 'exp' : 'inc'">¥{{ money(monthTotals.income - monthTotals.expense) }}</b></span>
        </div>

        <div class="week-row">
          <div v-for="w in WEEK_LABELS" :key="w" class="week-cell">{{ w }}</div>
        </div>

        <div class="day-grid">
          <button
            v-for="cell in calendarCells"
            :key="cell.dateStr"
            class="day-cell"
            :class="{ muted: !cell.inMonth, today: cell.isToday }"
            @click="pickDay(cell)"
          >
            <span class="day-num">{{ cell.day }}</span>
            <template v-if="cell.has">
              <span v-if="cell.expense > 0" class="day-exp">支 ¥{{ compactMoney(cell.expense) }}</span>
              <span v-if="cell.income > 0" class="day-inc">收 ¥{{ compactMoney(cell.income) }}</span>
              <span v-if="cell.transfer > 0 && cell.income === 0 && cell.expense === 0" class="day-transfer">转账 ×{{ cell.transfer }}</span>
            </template>
            <span v-else class="day-empty">·</span>
          </button>
        </div>
      </div>
    </template>

    <!-- 单日明细视图 -->
    <template v-else>
      <div class="page-head">
        <div>
          <h2>{{ dayTitle }}</h2>
          <p class="page-sub">当天记账明细</p>
        </div>
        <button class="btn" @click="backToCalendar">← 返回日历</button>
      </div>

      <div class="back-bar">
        <button class="link-btn" @click="backToCalendar">← 返回日历（{{ titleMonth }}）</button>
      </div>

      <div class="card day-summary">
        <div class="summary-item">
          <div class="summary-label">收入</div>
          <div class="summary-value inc">¥{{ money(dayTotals.income) }}</div>
        </div>
        <div class="summary-item">
          <div class="summary-label">支出</div>
          <div class="summary-value exp">¥{{ money(dayTotals.expense) }}</div>
        </div>
        <div class="summary-item">
          <div class="summary-label">结余</div>
          <div class="summary-value" :class="dayTotals.income - dayTotals.expense < 0 ? 'exp' : 'inc'">
            ¥{{ money(dayTotals.income - dayTotals.expense) }}
          </div>
        </div>
        <div v-if="dayTotals.transfer > 0" class="summary-item">
          <div class="summary-label">转账</div>
          <div class="summary-value muted">{{ dayTotals.transfer }} 笔</div>
        </div>
      </div>

      <div class="card list-card">
        <h3 class="list-title">明细（{{ dayTransactions.length }} 笔）</h3>
        <div class="tx-list">
          <div v-for="t in dayTransactions" :key="t.id" class="tx-item">
            <div class="tx-icon" :class="t.type">$</div>
            <div class="tx-main">
              <div class="tx-title">
                <span>{{ renderTitle(t) }}</span>
                <span v-if="t.isLarge" class="badge badge-large">大额</span>
              </div>
              <div class="tx-meta">{{ renderMeta(t) }}</div>
            </div>
            <div class="tx-amount" :class="t.type">
              {{ t.type === 'income' ? '+' : t.type === 'expense' ? '-' : '' }}¥{{ money(t.amount) }}
            </div>
            <button class="icon-btn" @click="remove(t)" title="删除">✕</button>
          </div>
          <div v-if="dayTransactions.length === 0" class="empty-row">这一天还没有记账记录</div>
        </div>
      </div>
    </template>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { useStore, refreshKeys, controllersApi } from '../data/store.js'
import { money, toDateStr } from '../core/utils.js'
import { TRANSACTION_TYPES } from '../core/constants.js'

const WEEK_LABELS = ['一', '二', '三', '四', '五', '六', '日']

const store = useStore()
const { transaction: txApi } = controllersApi

const now = new Date()
const viewYear = ref(now.getFullYear())
const viewMonth = ref(now.getMonth()) // 0-based
const selectedDate = ref(null) // 'YYYY-MM-DD'，非空时展示单日明细

// 按日期聚合的收支索引：dateStr -> { income, expense, transfer }
const dayIndex = computed(() => {
  const map = new Map()
  for (const t of store.transactions) {
    let slot = map.get(t.date)
    if (!slot) {
      slot = { income: 0, expense: 0, transfer: 0 }
      map.set(t.date, slot)
    }
    if (t.type === TRANSACTION_TYPES.INCOME) slot.income += Number(t.amount) || 0
    else if (t.type === TRANSACTION_TYPES.EXPENSE) slot.expense += Number(t.amount) || 0
    else if (t.type === TRANSACTION_TYPES.TRANSFER) slot.transfer += 1
  }
  return map
})

const titleMonth = computed(() => `${viewYear.value}年${viewMonth.value + 1}月`)
const currentMonthStr = computed(() => `${viewYear.value}-${String(viewMonth.value + 1).padStart(2, '0')}`)
const isViewingCurrentMonth = computed(() => {
  const d = new Date()
  return viewYear.value === d.getFullYear() && viewMonth.value === d.getMonth()
})

const monthTotals = computed(() => {
  const totals = { income: 0, expense: 0 }
  for (const [dateStr, slot] of dayIndex.value) {
    if (dateStr.startsWith(currentMonthStr.value)) {
      totals.income += slot.income
      totals.expense += slot.expense
    }
  }
  return totals
})

// 生成包含上月末 / 下月初补齐日期的 6 行 ×7 列格子（周一起始）
const calendarCells = computed(() => {
  const first = new Date(viewYear.value, viewMonth.value, 1)
  const offset = (first.getDay() + 6) % 7 // 周一为 0
  const start = new Date(viewYear.value, viewMonth.value, 1 - offset)
  const todayStr = toDateStr(new Date())
  const cells = []
  for (let i = 0; i < 42; i++) {
    const d = new Date(start.getFullYear(), start.getMonth(), start.getDate() + i)
    const dateStr = toDateStr(d)
    const slot = dayIndex.value.get(dateStr)
    cells.push({
      dateStr,
      day: d.getDate(),
      inMonth: d.getMonth() === viewMonth.value,
      isToday: dateStr === todayStr,
      has: Boolean(slot && (slot.income > 0 || slot.expense > 0 || slot.transfer > 0)),
      income: slot?.income || 0,
      expense: slot?.expense || 0,
      transfer: slot?.transfer || 0
    })
  }
  return cells
})

const changeMonth = (delta) => {
  const d = new Date(viewYear.value, viewMonth.value + delta, 1)
  viewYear.value = d.getFullYear()
  viewMonth.value = d.getMonth()
}
const goToday = () => {
  const d = new Date()
  viewYear.value = d.getFullYear()
  viewMonth.value = d.getMonth()
}

// 点击非本月日期先翻到所在月份；点击本月日期打开当天明细
const pickDay = (cell) => {
  if (!cell.inMonth) {
    const [y, m] = cell.dateStr.split('-').map(Number)
    viewYear.value = y
    viewMonth.value = m - 1
    return
  }
  selectedDate.value = cell.dateStr
}
const backToCalendar = () => {
  selectedDate.value = null
}

const dayTransactions = computed(() => {
  if (!selectedDate.value) return []
  return store.transactions
    .filter((t) => t.date === selectedDate.value)
    .sort((a, b) => b.createdAt - a.createdAt)
})

const dayTotals = computed(() => dayIndex.value.get(selectedDate.value) || { income: 0, expense: 0, transfer: 0 })

const dayTitle = computed(() => {
  if (!selectedDate.value) return ''
  const [y, m, d] = selectedDate.value.split('-').map(Number)
  const weekday = WEEK_LABELS[(new Date(y, m - 1, d).getDay() + 6) % 7]
  return `${y}年${m}月${d}日 · 周${weekday}`
})

const accountName = (id) => store.accounts.find((a) => a.id === id)?.name || '未知账户'
const renderTitle = (t) => {
  if (t.type === TRANSACTION_TYPES.TRANSFER) return `${accountName(t.fromAccountId)} → ${accountName(t.toAccountId)}`
  return t.category || (t.type === TRANSACTION_TYPES.INCOME ? '收入' : '支出')
}
const renderMeta = (t) => {
  const parts = []
  if (t.type === TRANSACTION_TYPES.TRANSFER) parts.push('转账')
  else parts.push(accountName(t.accountId), t.type === TRANSACTION_TYPES.INCOME ? '收入' : '支出')
  if (t.note) parts.push(t.note)
  return parts.join(' · ')
}

const remove = (t) => {
  if (txApi.removeTransaction(t.id)) {
    refreshKeys('transactions', 'accounts')
    controllersApi.achievement.updateAchievements()
    refreshKeys('achievements', 'points')
  }
}

// 金额紧凑展示：过万以「万」为单位
const compactMoney = (value) => {
  const n = Number(value) || 0
  if (n >= 10000) {
    const v = n / 10000
    return `${v >= 100 ? Math.round(v) : v.toFixed(1).replace(/\.0$/, '')}万`
  }
  if (n >= 1000) return String(Math.round(n))
  return money(n)
}

const onKeydown = (e) => {
  if (e.key === 'Escape' && selectedDate.value) backToCalendar()
}
onMounted(() => window.addEventListener('keydown', onKeydown))
onUnmounted(() => window.removeEventListener('keydown', onKeydown))
</script>

<style scoped>
/* ---- 日历 ---- */
.cal-card {
  padding: 18px 16px 20px;
}
.cal-head {
  display: flex;
  align-items: center;
  gap: 6px;
}
.month-title {
  font-size: 17px;
  font-weight: 800;
  min-width: 110px;
  text-align: center;
}
.month-btn {
  font-size: 22px;
  font-weight: 700;
  width: 34px;
  height: 34px;
}
.btn-today {
  margin-left: auto;
  padding: 6px 14px;
  font-size: 13px;
}
.month-summary {
  display: flex;
  flex-wrap: wrap;
  gap: 8px 20px;
  margin: 14px 2px 4px;
  font-size: 13px;
  color: var(--text-secondary);
}
.month-summary b {
  font-weight: 800;
  margin-left: 2px;
}
.inc { color: var(--income); }
.exp { color: var(--expense); }

.week-row,
.day-grid {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
}
.week-row {
  margin-top: 10px;
}
.week-cell {
  text-align: center;
  font-size: 12px;
  font-weight: 700;
  color: var(--text-secondary);
  padding: 6px 0;
}
.day-grid {
  gap: 6px;
}
.day-cell {
  position: relative;
  min-height: 78px;
  border: 1px solid var(--border-color);
  border-radius: 12px;
  background: var(--card-bg);
  padding: 6px 8px;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 2px;
  text-align: left;
  transition: border-color 0.15s, box-shadow 0.15s, transform 0.1s;
}
.day-cell:hover {
  border-color: var(--accent);
  box-shadow: 0 2px 8px rgba(79, 141, 249, 0.18);
}
.day-cell:active { transform: scale(0.98); }
.day-cell.muted {
  background: var(--bg-elevated);
  opacity: 0.55;
}
.day-num {
  font-size: 13px;
  font-weight: 700;
  color: var(--text-primary);
  line-height: 1.3;
}
.day-cell.today .day-num {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  background: var(--accent);
  color: #fff;
}
.day-cell.muted .day-num { color: var(--text-secondary); }
.day-exp,
.day-inc,
.day-transfer {
  font-size: 11px;
  font-weight: 700;
  line-height: 1.35;
  white-space: nowrap;
  max-width: 100%;
  overflow: hidden;
  text-overflow: ellipsis;
}
.day-exp { color: var(--expense); }
.day-inc { color: var(--income); }
.day-transfer { color: var(--accent); }
.day-empty {
  color: var(--border-color);
  font-size: 12px;
  line-height: 1;
}

/* ---- 单日明细 ---- */
.back-bar {
  margin: -8px 0 12px;
  font-size: 13px;
}
.day-summary {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
  margin-bottom: 16px;
}
.summary-item {
  flex: 1;
  min-width: 120px;
  background: var(--bg-elevated);
  border-radius: 12px;
  padding: 12px 16px;
}
.summary-label {
  font-size: 12px;
  color: var(--text-secondary);
  margin-bottom: 4px;
}
.summary-value {
  font-size: 18px;
  font-weight: 800;
}
.list-card {
  padding-bottom: 8px;
}
.list-title {
  margin: 0 0 6px;
  font-size: 15px;
}
.tx-list {
  display: flex;
  flex-direction: column;
}
.tx-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 4px;
  border-bottom: 1px solid var(--border-color);
}
.tx-item:last-child {
  border-bottom: none;
}
.tx-icon {
  width: 38px;
  height: 38px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 800;
  color: #fff;
  flex-shrink: 0;
}
.tx-icon.income { background: var(--income); }
.tx-icon.expense { background: var(--expense); }
.tx-icon.transfer { background: var(--accent); }
.tx-main {
  flex: 1;
  min-width: 0;
}
.tx-title {
  display: flex;
  align-items: center;
  gap: 8px;
  font-weight: 600;
  font-size: 14px;
}
.tx-meta {
  font-size: 12px;
  color: var(--text-secondary);
  margin-top: 2px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.tx-amount {
  font-weight: 800;
  white-space: nowrap;
}
.tx-amount.income { color: var(--income); }
.tx-amount.expense { color: var(--expense); }
.tx-amount.transfer { color: var(--accent); }
.empty-row {
  text-align: center;
  color: var(--text-secondary);
  padding: 24px 0;
  font-size: 13px;
}

@media (max-width: 768px) {
  .day-cell {
    min-height: 54px;
    padding: 4px 5px;
    border-radius: 8px;
  }
  .day-grid { gap: 4px; }
  .day-num { font-size: 12px; }
  .day-cell.today .day-num { width: 19px; height: 19px; }
  .day-exp,
  .day-inc,
  .day-transfer { font-size: 10px; }
  .month-summary { gap: 6px 14px; }
}
</style>
