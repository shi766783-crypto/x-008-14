<template>
  <div class="page">
    <div class="page-head">
      <div>
        <h2>日历</h2>
        <p class="page-sub">按天回顾收支，点击日期查看当天明细</p>
      </div>
      <button class="btn btn-primary" @click="goToday">回到今天</button>
    </div>

    <!-- 日历视图 -->
    <template v-if="!selectedDate">
      <div class="card calendar-card">
        <div class="cal-toolbar">
          <button class="nav-arrow" @click="prevMonth" title="上一月">‹</button>
          <div class="cal-title">{{ monthLabel(cursorMonth) }}</div>
          <button class="nav-arrow" @click="nextMonth" title="下一月">›</button>
        </div>

        <div class="month-summary">
          <span class="ms-item">收入 <b class="income">¥{{ money(monthTotals.income) }}</b></span>
          <span class="ms-item">支出 <b class="expense">¥{{ money(monthTotals.expense) }}</b></span>
          <span class="ms-item">结余 <b :class="{ neg: monthTotals.income - monthTotals.expense < 0 }">¥{{ money(monthTotals.income - monthTotals.expense) }}</b></span>
        </div>

        <div class="week-row">
          <div v-for="w in WEEKDAYS" :key="w" class="week-cell">{{ w }}</div>
        </div>

        <div class="day-grid">
          <div v-for="blank in firstWeekday" :key="'blank-' + blank" class="day-cell blank"></div>
          <button
            v-for="day in daysInMonth"
            :key="day.dateStr"
            class="day-cell"
            :class="{ today: day.dateStr === today, 'has-tx': day.hasTx }"
            @click="openDay(day.dateStr)"
          >
            <span class="day-num">{{ day.day }}</span>
            <span v-if="day.income > 0" class="day-amt income">+{{ shortMoney(day.income) }}</span>
            <span v-if="day.expense > 0" class="day-amt expense">-{{ shortMoney(day.expense) }}</span>
            <span v-if="!day.hasTx" class="day-empty"></span>
          </button>
        </div>

        <div class="cal-legend">
          <span><i class="dot income-dot"></i>收入</span>
          <span><i class="dot expense-dot"></i>支出</span>
          <span class="muted">空白格表示当天没有记录，依然可以点击补记</span>
        </div>
      </div>
    </template>

    <!-- 当天明细视图 -->
    <template v-else>
      <div class="detail-head">
        <button class="btn back-btn" @click="backToCalendar">← 返回日历</button>
        <h3 class="detail-title">{{ dayTitle }}</h3>
      </div>

      <div class="day-kpis">
        <div class="card kpi">
          <span class="kpi-label">当天收入</span>
          <b class="kpi-value income">¥{{ money(dayTotals.income) }}</b>
        </div>
        <div class="card kpi">
          <span class="kpi-label">当天支出</span>
          <b class="kpi-value expense">¥{{ money(dayTotals.expense) }}</b>
        </div>
        <div class="card kpi">
          <span class="kpi-label">当天结余</span>
          <b class="kpi-value" :class="{ neg: dayTotals.income - dayTotals.expense < 0 }">¥{{ money(dayTotals.income - dayTotals.expense) }}</b>
        </div>
      </div>

      <div class="card list-card">
        <div class="detail-list-head">
          <h3 class="list-title">明细（{{ dayTransactions.length }} 笔）</h3>
          <button class="btn btn-primary btn-sm" @click="openAdd">＋ 记一笔</button>
        </div>
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
          <div v-if="dayTransactions.length === 0" class="empty-row">
            这一天还没有记录
            <div class="empty-actions">
              <button class="btn btn-primary btn-sm" @click="openAdd">为这一天记一笔</button>
            </div>
          </div>
        </div>
      </div>
    </template>

    <Modal v-if="addOpen" title="记一笔" @close="addOpen = false">
      <form id="cal-tx-form" @submit.prevent="submitAdd" class="form">
        <div class="seg type-seg">
          <button type="button" class="seg-btn wide" :class="{ active: form.type === 'income' }" @click="switchType('income')">收入</button>
          <button type="button" class="seg-btn wide" :class="{ active: form.type === 'expense' }" @click="switchType('expense')">支出</button>
          <button type="button" class="seg-btn wide" :class="{ active: form.type === 'transfer' }" @click="switchType('transfer')">转账</button>
        </div>

        <label class="field">
          <span>{{ form.type === 'transfer' ? '转出账户' : '账户' }}</span>
          <select v-model="form.accountId" required>
            <option value="" disabled>选择账户</option>
            <option v-for="a in store.accounts" :key="a.id" :value="a.id">{{ a.name }}</option>
          </select>
        </label>

        <label class="field" v-if="form.type === 'transfer'">
          <span>转入账户</span>
          <select v-model="form.toAccountId" required>
            <option value="" disabled>选择转入账户</option>
            <option v-for="a in otherAccounts" :key="a.id" :value="a.id">{{ a.name }}</option>
          </select>
        </label>

        <label class="field">
          <span>金额</span>
          <input v-model.number="form.amount" type="number" min="0.01" step="0.01" required placeholder="0.00" />
        </label>

        <label class="field" v-if="form.type !== 'transfer'">
          <span>类别</span>
          <select v-model="form.category">
            <option v-for="c in currentCategories" :key="c" :value="c">{{ c }}</option>
          </select>
        </label>

        <label class="field">
          <span>日期</span>
          <input v-model="form.date" type="date" required />
        </label>

        <label class="field">
          <span>备注</span>
          <input v-model="form.note" placeholder="选填" />
        </label>

        <label class="check field-check">
          <input type="checkbox" v-model="form.isLarge" />
          大额支出（单笔 ≥ 1000 元）
        </label>
      </form>

      <template #footer>
        <button type="button" class="btn" @click="addOpen = false">取消</button>
        <button type="submit" class="btn btn-primary" form="cal-tx-form">保存</button>
      </template>
    </Modal>
  </div>
</template>

<script setup>
import { ref, reactive, computed } from 'vue'
import { useStore, refreshKeys, controllersApi } from '../data/store.js'
import { money, monthLabel, todayStr } from '../core/utils.js'
import { INCOME_CATEGORIES, EXPENSE_CATEGORIES, TRANSACTION_TYPES } from '../core/constants.js'
import Modal from '../components/Modal.vue'

const store = useStore()
const { transaction: txApi } = controllersApi

const WEEKDAYS = ['一', '二', '三', '四', '五', '六', '日']
const today = todayStr()

// 日历光标所在月（YYYY-MM），跨月翻页只改这里
const cursorMonth = ref(today.slice(0, 7))
// 点开的日期（YYYY-MM-DD），为空时展示日历
const selectedDate = ref('')

const shiftMonth = (delta) => {
  const [y, m] = cursorMonth.value.split('-').map(Number)
  const d = new Date(y, m - 1 + delta, 1)
  cursorMonth.value = `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}`
}
const prevMonth = () => shiftMonth(-1)
const nextMonth = () => shiftMonth(1)
const goToday = () => {
  selectedDate.value = ''
  cursorMonth.value = today.slice(0, 7)
}

// 当月全部交易
const monthTransactions = computed(() => store.transactions.filter((t) => t.date.startsWith(cursorMonth.value)))

const monthTotals = computed(() => {
  let income = 0
  let expense = 0
  for (const t of monthTransactions.value) {
    if (t.type === TRANSACTION_TYPES.INCOME) income += t.amount
    else if (t.type === TRANSACTION_TYPES.EXPENSE) expense += t.amount
  }
  return { income, expense }
})

// 以日期为键的收支合计（同一数据源同时驱动网格与月汇总，保证翻页切换正确）
const dailyMap = computed(() => {
  const map = new Map()
  for (const t of monthTransactions.value) {
    if (t.type === TRANSACTION_TYPES.TRANSFER) continue
    const row = map.get(t.date) || { income: 0, expense: 0 }
    if (t.type === TRANSACTION_TYPES.INCOME) row.income += t.amount
    else row.expense += t.amount
    map.set(t.date, row)
  }
  return map
})

const firstWeekday = computed(() => {
  const [y, m] = cursorMonth.value.split('-').map(Number)
  // JS getDay() 周日为 0，转成周一为 0
  return (new Date(y, m - 1, 1).getDay() + 6) % 7
})

const daysInMonth = computed(() => {
  const [y, m] = cursorMonth.value.split('-').map(Number)
  const count = new Date(y, m, 0).getDate()
  const out = []
  for (let day = 1; day <= count; day++) {
    const dateStr = `${cursorMonth.value}-${String(day).padStart(2, '0')}`
    const stat = dailyMap.value.get(dateStr)
    out.push({ day, dateStr, income: stat?.income || 0, expense: stat?.expense || 0, hasTx: Boolean(stat) })
  }
  return out
})

const openDay = (dateStr) => {
  selectedDate.value = dateStr
}
const backToCalendar = () => {
  // 回到当前光标所在月，不重置翻页位置
  selectedDate.value = ''
}

// 当天明细
const dayTransactions = computed(() =>
  store.transactions
    .filter((t) => t.date === selectedDate.value)
    .sort((a, b) => b.createdAt - a.createdAt)
)

const dayTotals = computed(() => {
  let income = 0
  let expense = 0
  for (const t of dayTransactions.value) {
    if (t.type === TRANSACTION_TYPES.INCOME) income += t.amount
    else if (t.type === TRANSACTION_TYPES.EXPENSE) expense += t.amount
  }
  return { income, expense }
})

const dayTitle = computed(() => {
  if (!selectedDate.value) return ''
  const [y, m, d] = selectedDate.value.split('-').map(Number)
  const week = WEEKDAYS[(new Date(y, m - 1, d).getDay() + 6) % 7]
  return `${y} 年 ${m} 月 ${d} 日 · 周${week}`
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

// 单元格空间有限，金额做紧凑展示
const shortMoney = (value) => {
  const n = Number(value) || 0
  if (n >= 100000) return `${(n / 10000).toFixed(1)}万`
  return Number.isInteger(n) ? String(n) : n.toFixed(1).replace(/\.0$/, '')
}

const remove = (t) => {
  if (txApi.removeTransaction(t.id)) {
    refreshKeys('transactions', 'accounts')
    controllersApi.achievement.updateAchievements()
    refreshKeys('achievements', 'points')
  }
}

// 为这一天补记：在当前页弹出表单，日期自动带入选中日期
const addOpen = ref(false)
const form = reactive(txApi.emptyTransactionForm())
const otherAccounts = computed(() => store.accounts.filter((a) => a.id !== form.accountId))
const currentCategories = computed(() => (form.type === TRANSACTION_TYPES.INCOME ? INCOME_CATEGORIES : EXPENSE_CATEGORIES))

const switchType = (type) => {
  form.type = type
  form.category = type === TRANSACTION_TYPES.INCOME ? INCOME_CATEGORIES[0] : EXPENSE_CATEGORIES[0]
  form.toAccountId = ''
}

const openAdd = () => {
  Object.assign(form, txApi.emptyTransactionForm(), {
    accountId: store.accounts[0]?.id || '',
    toAccountId: store.accounts[1]?.id || '',
    date: selectedDate.value
  })
  addOpen.value = true
}

const submitAdd = () => {
  if (!form.accountId || !form.amount) return
  if (form.type === TRANSACTION_TYPES.TRANSFER && form.accountId === form.toAccountId) {
    alert('转账账户不能相同')
    return
  }
  txApi.addTransaction(form)
  refreshKeys('transactions', 'accounts')
  controllersApi.achievement.updateAchievements()
  refreshKeys('achievements', 'points')
  addOpen.value = false
}
</script>

<style scoped>
/* 日历 */
.calendar-card {
  padding: 18px;
}
.cal-toolbar {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  margin-bottom: 6px;
}
.cal-title {
  font-size: 17px;
  font-weight: 800;
  min-width: 150px;
  text-align: center;
}
.nav-arrow {
  border: 1px solid var(--border-color);
  background: var(--card-bg);
  color: var(--text-primary);
  width: 34px;
  height: 34px;
  border-radius: 10px;
  font-size: 20px;
  line-height: 1;
  cursor: pointer;
  transition: background 0.15s;
}
.nav-arrow:hover { background: var(--bg-elevated); }

.month-summary {
  display: flex;
  justify-content: center;
  gap: 24px;
  font-size: 13px;
  color: var(--text-secondary);
  margin-bottom: 14px;
  flex-wrap: wrap;
}
.ms-item b { margin-left: 4px; }

.week-row,
.day-grid {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 6px;
}
.week-cell {
  text-align: center;
  font-size: 12px;
  font-weight: 700;
  color: var(--text-secondary);
  padding: 4px 0;
}
.day-grid {
  grid-auto-rows: 1fr;
}
.day-cell {
  position: relative;
  min-height: 84px;
  border: 1px solid var(--border-color);
  border-radius: 12px;
  background: var(--card-bg);
  padding: 6px 6px 4px;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 2px;
  cursor: pointer;
  transition: border-color 0.15s, background 0.15s, transform 0.1s;
  text-align: left;
}
.day-cell:hover {
  border-color: var(--accent);
  background: rgba(79, 141, 249, 0.06);
}
.day-cell:active { transform: scale(0.98); }
.day-cell.has-tx { background: var(--bg-elevated); }
.day-cell.has-tx:hover { background: rgba(79, 141, 249, 0.08); }
.day-cell.blank {
  border: none;
  background: transparent;
  cursor: default;
  min-height: 84px;
}
.day-num {
  font-size: 13px;
  font-weight: 700;
  color: var(--text-primary);
}
.day-cell.today {
  border-color: var(--accent);
  box-shadow: inset 0 0 0 1px var(--accent);
}
.day-cell.today .day-num {
  background: var(--accent);
  color: #fff;
  border-radius: 999px;
  width: 22px;
  height: 22px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
}
.day-amt {
  font-size: 11px;
  font-weight: 700;
  line-height: 1.3;
  max-width: 100%;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.day-amt.income { color: var(--income); }
.day-amt.expense { color: var(--expense); }
.day-empty { flex: 1; }

.cal-legend {
  display: flex;
  align-items: center;
  gap: 16px;
  margin-top: 14px;
  font-size: 12px;
  color: var(--text-secondary);
  flex-wrap: wrap;
}
.dot {
  display: inline-block;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  margin-right: 4px;
}
.income-dot { background: var(--income); }
.expense-dot { background: var(--expense); }

/* 当天明细 */
.detail-head {
  display: flex;
  align-items: center;
  gap: 14px;
  margin-bottom: 14px;
  flex-wrap: wrap;
}
.back-btn {
  font-weight: 700;
}
.detail-title {
  margin: 0;
  font-size: 17px;
}
.day-kpis {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
  margin-bottom: 16px;
}
.kpi {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.kpi-label {
  font-size: 13px;
  color: var(--text-secondary);
}
.kpi-value {
  font-size: 20px;
  font-weight: 800;
}
.kpi-value.income { color: var(--income); }
.kpi-value.expense { color: var(--expense); }

.list-card { padding-bottom: 8px; }
.detail-list-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 6px;
}
.list-title {
  margin: 0;
  font-size: 15px;
}
.btn-sm {
  padding: 6px 12px;
  font-size: 13px;
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
  padding: 28px 0;
  font-size: 13px;
}
.empty-actions {
  margin-top: 12px;
}

@media (max-width: 768px) {
  .day-cell { min-height: 64px; }
  .day-cell.blank { min-height: 64px; }
  .day-kpis { grid-template-columns: 1fr; }
  .month-summary { gap: 14px; }
}
</style>
