<template>
  <view class="page">
    <!-- 顶部卡片：按钮 + 难度选择 + 状态条 -->
    <view class="card top-card">
      <view class="top-row">
        <view class="bar-btn btn-back" hover-class="bar-hover" @click="goBack">
          <text class="btn-back-text">← 返回大厅</text>
        </view>
        <view class="bar-btn btn-reset" hover-class="btn-reset-hover" @click="resetGame">
          <text class="btn-reset-text">重置游戏</text>
        </view>
      </view>

      <view class="diff-row">
        <view
          v-for="d in DIFFS"
          :key="d.key"
          class="diff-btn"
          :class="diff === d.key ? 'diff-active-' + d.key : ''"
          hover-class="diff-hover"
          @click="onDifficulty(d.key)"
        >
          <text class="diff-text" :class="diff === d.key ? 'diff-text-' + d.key : ''">{{ d.label }}</text>
        </view>
      </view>

      <view class="status-row">
        <view class="stat-box">
          <text class="stat-label">难度</text>
          <text class="stat-value" :class="'diff-color-' + diff">{{ DIFF_MAP[diff].label }}</text>
        </view>
        <view class="stat-box">
          <text class="stat-label">剩余</text>
          <text class="stat-value stat-num">{{ emptyCount }}</text>
        </view>
        <view class="stat-box">
          <text class="stat-label">用时</text>
          <text class="stat-value stat-num">{{ fmtTime(seconds) }}</text>
        </view>
        <view class="stat-box">
          <text class="stat-label">最佳</text>
          <text class="stat-value stat-num">{{ best > 0 ? fmtTime(best) : '--' }}</text>
        </view>
      </view>
    </view>

    <!-- 棋盘卡片：深色描边圆角容器 + 9x9 格 -->
    <view class="card board-card">
      <view class="board">
        <view
          v-for="i in COUNT"
          :key="i - 1"
          class="cell"
          :class="cellClass(i - 1)"
          @click="onSelect(i - 1)"
        >
          <text
            v-if="board[i - 1] > 0"
            class="cell-text"
            :class="cellTextClass(i - 1)"
          >{{ board[i - 1] }}</text>
        </view>
      </view>
      <text v-if="won" class="win-flag">🎉 已通关</text>
    </view>

    <!-- 数字键盘卡片：1–9（含填入计数）+ 擦除 -->
    <view class="card pad-card">
      <view
        v-for="n in NUMS"
        :key="'d' + n"
        class="pad-btn"
        hover-class="pad-hover"
        @click="onFill(n)"
      >
        <text class="pad-num">{{ n }}</text>
        <text v-if="remainHint" class="pad-count" :class="{ 'pad-count-done': digitCount[n] >= 9 }">剩{{ 9 - digitCount[n] }}</text>
      </view>
      <view class="pad-btn pad-erase" hover-class="pad-hover" @click="onErase">
        <text class="pad-erase-text">擦除</text>
      </view>
    </view>

    <view class="hint">
      <view
        class="hint-toggle"
        :class="{ 'hint-toggle-on': remainHint }"
        hover-class="bar-hover"
        @click="toggleRemainHint"
      >
        <text class="hint-toggle-text" :class="{ 'hint-toggle-on-text': remainHint }">剩余提示（{{ remainHint ? '开' : '关' }}）</text>
      </view>
      <text class="hint-text">点击格子选中，下方 1–9 填入；题目格不可改，冲突红字实时标出</text>
    </view>
  </view>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { onLoad, onUnload } from '@dcloudio/uni-app'

/** 棋盘边长（9x9） */
const SIZE = 9
/** 格子总数 */
const COUNT = SIZE * SIZE
/** 键盘数字 1–9 */
const NUMS: number[] = [1, 2, 3, 4, 5, 6, 7, 8, 9]

/** 难度键 */
type DiffKey = 'easy' | 'medium' | 'hard'

/** 各难度配置：挖空区间（空格数）与展示文案 */
const DIFFS: { key: DiffKey; label: string }[] = [
  { key: 'easy', label: '简单' },
  { key: 'medium', label: '中等' },
  { key: 'hard', label: '困难' },
]
const DIFF_MAP: Record<DiffKey, { label: string; min: number; max: number }> = {
  easy: { label: '简单', min: 20, max: 28 },
  medium: { label: '中等', min: 34, max: 42 },
  hard: { label: '困难', min: 44, max: 52 },
}

/** 本地存储 key（uni 原生 API，按难度分记最佳用时） */
const DIFF_KEY = 'SUDOKU_DIFF'
const REMAIN_HINT_KEY = 'SUDOKU_REMAIN_HINT'
const bestKeyOf = (d: DiffKey): string => 'SUDOKU_BEST_TIME_' + d.toUpperCase()

/** 当前棋盘：0 空格，1–9 已填 */
const board = ref<number[]>(new Array(COUNT).fill(0))
/** 题目格（初始 given，不可修改） */
const given = ref<boolean[]>(new Array(COUNT).fill(false))
/** 初始题目快照（重置时恢复用） */
const initial = ref<number[]>(new Array(COUNT).fill(0))
/** 参考唯一解 */
const solution = ref<number[]>(new Array(COUNT).fill(0))
/** 选中格索引（-1 未选中） */
const selected = ref(-1)
/** 是否通关 */
const won = ref(false)
/** 本局用时（秒） */
const seconds = ref(0)
/** 当前难度 */
const diff = ref<DiffKey>('easy')
/** 当前难度最佳（秒），0 无记录 */
const best = ref(0)
/** 剩余数量提示开关（UI 偏好，持久化；默认关闭） */
const remainHint = ref(false)
/** 生成中标记（遮罩防重入） */
const generating = ref(false)
/** 计时器句柄 */
let timer = 0

/** 剩余空格数 */
const emptyCount = computed(() => board.value.filter((v) => v === 0).length)

/** 是否冲突标记数组（随 board 响应式重算，避免每格重复计算） */
const conflictMap = computed(() => conflicts())
/** 冲突格数 */
const conflictCount = computed(() => conflictMap.value.filter(Boolean).length)

/** 各数字已"正确填入"的计数：1–9 → 正确放置于解位置的格子数（题目格也计入） */
const digitCount = computed<number[]>(() => {
  const arr: number[] = [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
  const b = board.value
  const s = solution.value
  for (let i = 0; i < COUNT; i++) {
    const v = b[i]
    if (v > 0 && v === s[i]) arr[v] += 1
  }
  return arr
})

/** 秒 → mm:ss */
function fmtTime(sec: number): string {
  const m = Math.floor(sec / 60)
  const s = sec % 60
  return (m < 10 ? '0' : '') + m + ':' + (s < 10 ? '0' : '') + s
}

/** 当前冲突格子集合（行/列/宫内重复数字所在位置） */
function conflicts(): boolean[] {
  const bad = new Array(COUNT).fill(false)
  const mark = (idxs: number[]) => {
    const seen: Record<number, number[]> = {}
    for (const i of idxs) {
      const v = board.value[i]
      if (v === 0) continue
      ;(seen[v] = seen[v] || []).push(i)
    }
    for (const k in seen) {
      if (seen[k].length > 1) for (const i of seen[k]) bad[i] = true
    }
  }
  for (let r = 0; r < SIZE; r++) {
    const row: number[] = []
    for (let c = 0; c < SIZE; c++) row.push(r * SIZE + c)
    mark(row)
  }
  for (let c = 0; c < SIZE; c++) {
    const col: number[] = []
    for (let r = 0; r < SIZE; r++) col.push(r * SIZE + c)
    mark(col)
  }
  for (let br = 0; br < 3; br++) {
    for (let bc = 0; bc < 3; bc++) {
      const box: number[] = []
      for (let dr = 0; dr < 3; dr++) {
        for (let dc = 0; dc < 3; dc++) box.push((br * 3 + dr) * SIZE + (bc * 3 + dc))
      }
      mark(box)
    }
  }
  return bad
}

/* ---------- 谜题生成 ---------- */

/** 随机顺序回填 + 回溯，生成一个完整合法解 */
function generateFullSolution(): number[] {
  const grid = new Array(COUNT).fill(0)
  const fill = (idx: number): boolean => {
    if (idx === COUNT) return true
    const nums = [1, 2, 3, 4, 5, 6, 7, 8, 9]
    for (let i = nums.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1))
      ;[nums[i], nums[j]] = [nums[j], nums[i]]
    }
    const row = Math.floor(idx / SIZE)
    const col = idx % SIZE
    const br = Math.floor(row / 3)
    const bc = Math.floor(col / 3)
    for (const n of nums) {
      let ok = true
      for (let c = 0; c < SIZE && ok; c++) if (grid[row * SIZE + c] === n) ok = false
      for (let r = 0; r < SIZE && ok; r++) if (grid[r * SIZE + col] === n) ok = false
      for (let dr = 0; dr < 3 && ok; dr++) {
        for (let dc = 0; dc < 3 && ok; dc++) {
          if (grid[(br * 3 + dr) * SIZE + (bc * 3 + dc)] === n) ok = false
        }
      }
      if (!ok) continue
      grid[idx] = n
      if (fill(idx + 1)) return true
      grid[idx] = 0
    }
    return false
  }
  fill(0)
  return grid
}

/** 回溯计数法：统计解的个数，最多数到 limit（=2）；返回 1 即唯一解 */
function countSolutions(puzzle: number[], limit = 2): number {
  const grid = puzzle.slice()
  const rec = (): number => {
    let empty = -1
    for (let i = 0; i < COUNT; i++) {
      if (grid[i] === 0) {
        empty = i
        break
      }
    }
    if (empty === -1) return 1
    const row = Math.floor(empty / SIZE)
    const col = empty % SIZE
    const br = Math.floor(row / 3)
    const bc = Math.floor(col / 3)
    let total = 0
    for (let n = 1; n <= 9 && total < limit; n++) {
      let ok = true
      for (let c = 0; c < SIZE && ok; c++) if (grid[row * SIZE + c] === n) ok = false
      for (let r = 0; r < SIZE && ok; r++) if (grid[r * SIZE + col] === n) ok = false
      for (let dr = 0; dr < 3 && ok; dr++) {
        for (let dc = 0; dc < 3 && ok; dc++) {
          if (grid[(br * 3 + dr) * SIZE + (bc * 3 + dc)] === n) ok = false
        }
      }
      if (!ok) continue
      grid[empty] = n
      total += rec()
      grid[empty] = 0
    }
    return total
  }
  return rec()
}

/** 生成一题（同步）：全解 → 按难度挖空，每挖一格校验唯一解，挖满目标数或遍历完为止 */
function newPuzzle(d: DiffKey): void {
  const solutionGrid = generateFullSolution()
  const cfg = DIFF_MAP[d]
  const puzzle = solutionGrid.slice()
  const order: number[] = []
  for (let i = 0; i < COUNT; i++) order.push(i)
  for (let i = order.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[order[i], order[j]] = [order[j], order[i]]
  }
  const target = cfg.min + Math.floor(Math.random() * (cfg.max - cfg.min + 1))
  let dug = 0
  for (const i of order) {
    if (dug >= target) break
    const keep = puzzle[i]
    puzzle[i] = 0
    if (countSolutions(puzzle, 2) === 1) {
      dug += 1
    } else {
      puzzle[i] = keep
    }
  }
  board.value = puzzle.slice()
  initial.value = puzzle.slice()
  solution.value = solutionGrid
  const g = new Array(COUNT).fill(false)
  puzzle.forEach((v, i) => (g[i] = v !== 0))
  given.value = g
  selected.value = -1
  won.value = false
  stopTimer()
  seconds.value = 0
}

/** 带 loading 提示的新题生成：先 toast 再让 UI 渲染，随后同步生成 */
function genNewPuzzle(d: DiffKey): void {
  if (generating.value) return
  generating.value = true
  uni.showToast({ title: '正在生成新题…', icon: 'none' })
  setTimeout(() => {
    newPuzzle(d)
    generating.value = false
  }, 0)
}

/* ---------- 交互 ---------- */

/** 选中格子 */
function onSelect(i: number): void {
  if (won.value || generating.value) return
  selected.value = i
}

/** 填入当前选中格：题目格不可改 */
function onFill(n: number): void {
  if (won.value || generating.value) return
  const i = selected.value
  if (i < 0) return
  if (given.value[i]) return
  if (board.value[i] === n) return
  if (!timer) startTimer()
  board.value[i] = n
  checkWin()
}

/** 擦除当前选中格（题目格不可擦） */
function onErase(): void {
  if (won.value || generating.value) return
  const i = selected.value
  if (i < 0) return
  if (given.value[i]) return
  if (board.value[i] === 0) return
  board.value[i] = 0
}

/** 胜利判定：全部填满 + 无冲突 → 更新该难度最佳 */
function checkWin(): void {
  if (emptyCount.value !== 0) return
  if (conflictCount.value !== 0) return
  won.value = true
  stopTimer()
  const isNewBest = best.value === 0 || seconds.value < best.value
  if (isNewBest) {
    best.value = seconds.value
    saveBest(diff.value, seconds.value)
  }
  uni.showToast({
    title: isNewBest
      ? `🎉 通关！新纪录 ${fmtTime(seconds.value)}`
      : `🎉 通关！用时 ${fmtTime(seconds.value)}`,
    icon: 'none',
  })
}

/** 开始计时（每秒 +1） */
function startTimer(): void {
  if (timer) return
  timer = setInterval(() => {
    seconds.value += 1
  }, 1000)
}

/** 停止并清理计时器 */
function stopTimer(): void {
  if (timer) {
    clearInterval(timer)
    timer = 0
  }
}

/** 重置游戏：恢复本局初始题目、计时清零、取消胜利 */
function resetGame(): void {
  if (generating.value) return
  board.value = initial.value.slice()
  selected.value = -1
  won.value = false
  stopTimer()
  seconds.value = 0
  uni.showToast({ title: '已重置，题目恢复初始状态', icon: 'none' })
}

/** 切换难度：记住选择、载入该难度最佳、立即生成新题 */
function onDifficulty(d: DiffKey): void {
  if (generating.value) return
  if (d === diff.value) return
  diff.value = d
  saveDiff(d)
  best.value = loadBest(d)
  genNewPuzzle(d)
}

/* ---------- 本地存储（uni 原生 API） ---------- */

function loadBest(d: DiffKey): number {
  try {
    const v = uni.getStorageSync(bestKeyOf(d))
    return typeof v === 'number' && v > 0 ? v : 0
  } catch {
    return 0
  }
}

function saveBest(d: DiffKey, v: number): void {
  try {
    uni.setStorageSync(bestKeyOf(d), v)
  } catch (e) {
    console.warn('写入最佳用时失败', e)
  }
}

function loadDiff(): DiffKey {
  try {
    const v = uni.getStorageSync(DIFF_KEY)
    if (v === 'easy' || v === 'medium' || v === 'hard') return v
  } catch {
    /* ignore */
  }
  return 'easy'
}

function saveDiff(d: DiffKey): void {
  try {
    uni.setStorageSync(DIFF_KEY, d)
  } catch {
    /* ignore */
  }
}

function loadRemainHint(): boolean {
  try {
    return uni.getStorageSync(REMAIN_HINT_KEY) === true
  } catch {
    return false
  }
}

function saveRemainHint(v: boolean): void {
  try {
    uni.setStorageSync(REMAIN_HINT_KEY, v)
  } catch (e) {
    console.warn('写入剩余提示开关失败', e)
  }
}

/** 切换"剩余数量提示"显示；纯 UI 偏好，不影响棋盘/计时/判定 */
function toggleRemainHint(): void {
  remainHint.value = !remainHint.value
  saveRemainHint(remainHint.value)
}

/** 返回大厅；页面栈无法回退时 reLaunch 到首页 */
function goBack(): void {
  uni.navigateBack({
    fail: () => {
      uni.reLaunch({ url: '/pages/index/index' })
    },
  })
}

/* ---------- 渲染辅助 ---------- */

/** 该格是否与选中格同行/同列/同宫 */
function isSamePeer(i: number): boolean {
  const s = selected.value
  if (s < 0) return false
  const r = Math.floor(i / SIZE)
  const c = i % SIZE
  const sr = Math.floor(s / SIZE)
  const sc = s % SIZE
  if (r === sr || c === sc) return true
  return Math.floor(r / 3) === Math.floor(sr / 3) && Math.floor(c / 3) === Math.floor(sc / 3)
}

/** 单元格 class：宫格粗线 + 选中/同行列宫/相同数字高亮 */
function cellClass(i: number): string {
  const cls: string[] = []
  const r = Math.floor(i / SIZE)
  const c = i % SIZE
  if (c % 3 === 2 && c !== SIZE - 1) cls.push('line-right-thick')
  if (r % 3 === 2 && r !== SIZE - 1) cls.push('line-bottom-thick')
  if (c === SIZE - 1) cls.push('line-right-last')
  if (r === SIZE - 1) cls.push('line-bottom-last')
  if (!generating.value) {
    const sv = board.value[selected.value >= 0 ? selected.value : -1]
    if (i === selected.value) {
      cls.push('cell-selected')
    } else {
      if (isSamePeer(i)) cls.push('cell-peer')
      if (sv > 0 && board.value[i] === sv) cls.push('cell-same')
    }
  }
  return cls.join(' ')
}

/** 单元格数字配色：冲突红字 > 题目深灰 > 填写蓝 */
function cellTextClass(i: number): string {
  if (conflictMap.value[i]) return 'text-conflict'
  return given.value[i] ? 'text-given' : 'text-filled'
}

onLoad(() => {
  remainHint.value = loadRemainHint()
  const d = loadDiff()
  diff.value = d
  best.value = loadBest(d)
  genNewPuzzle(d)
})

onUnload(() => {
  stopTimer()
})
</script>

<style scoped>
/* ---------- 页面：渐变背景 + 卡片化 ---------- */
.page {
  min-height: 100vh;
  background: linear-gradient(180deg, #dcebff 0%, #e6efff 45%, #eef0fd 100%);
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 24rpx 20rpx 40rpx;
  box-sizing: border-box;
}

.card {
  width: 730rpx;
  max-width: 100%;
  background: #ffffff;
  border-radius: 28rpx;
  box-shadow: 0 8rpx 28rpx rgba(58, 88, 140, 0.12);
  box-sizing: border-box;
  margin-bottom: 24rpx;
}

/* ---------- 顶部卡片 ---------- */
.top-card {
  padding: 28rpx 28rpx 24rpx;
}

.top-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24rpx;
}

.bar-btn {
  border-radius: 40rpx;
  padding: 16rpx 32rpx;
}

.btn-back {
  background: #f2f6fc;
  border: 1rpx solid #d9e3f2;
}

.bar-hover {
  opacity: 0.75;
}

.btn-back-text {
  font-size: 28rpx;
  color: #3a5890;
  font-weight: 600;
}

.btn-reset {
  background: #3f6fd0;
}

.btn-reset-hover {
  background: #3058ab;
}

.btn-reset-text {
  font-size: 28rpx;
  color: #ffffff;
  font-weight: 600;
}

/* 难度选择：横排胶囊分段按钮 */
.diff-row {
  display: flex;
  justify-content: center;
  margin-bottom: 24rpx;
}

.diff-btn {
  width: 196rpx;
  height: 76rpx;
  border-radius: 40rpx;
  background: #f1f4f8;
  border: 1rpx solid #e2e8f0;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-right: 20rpx;
}

.diff-btn:last-child {
  margin-right: 0;
}

.diff-hover {
  opacity: 0.85;
}

.diff-text {
  font-size: 30rpx;
  color: #5b6b80;
  font-weight: 600;
}

.diff-active-easy {
  background: #e3f5e9;
  border-color: #34a264;
}

.diff-active-medium {
  background: #e5eefb;
  border-color: #3f6fd0;
}

.diff-active-hard {
  background: #fdece8;
  border-color: #e05a3c;
}

.diff-text-easy {
  color: #21794a;
}

.diff-text-medium {
  color: #2e55a5;
}

.diff-text-hard {
  color: #c2401f;
}

/* 状态条：难度 / 剩余 / 用时 / 最佳 */
.status-row {
  display: flex;
  justify-content: space-between;
}

.stat-box {
  display: flex;
  flex-direction: column;
  align-items: center;
  flex: 1;
}

.stat-label {
  font-size: 22rpx;
  color: #8a97ab;
  margin-bottom: 6rpx;
}

.stat-value {
  font-size: 32rpx;
  font-weight: 700;
  color: #2c3e50;
}

.stat-num {
  color: #24509c;
}

.diff-color-easy {
  color: #21794a;
}

.diff-color-medium {
  color: #2e55a5;
}

.diff-color-hard {
  color: #c2401f;
}

/* ---------- 棋盘 ---------- */
.board-card {
  padding: 20rpx;
  display: flex;
  align-items: center;
  justify-content: center;
}

.board {
  width: 638rpx;
  background: #2c3e50;
  border-radius: 14rpx;
  padding: 4rpx;
  box-sizing: border-box;
  display: grid;
  grid-template-columns: repeat(9, 70rpx);
}

.cell {
  width: 70rpx;
  height: 70rpx;
  box-sizing: border-box;
  background: #ffffff;
  border-right: 1rpx solid #c8d2e0;
  border-bottom: 1rpx solid #c8d2e0;
  display: flex;
  align-items: center;
  justify-content: center;
}

.line-right-thick {
  border-right: 4rpx solid #2c3e50;
}

.line-bottom-thick {
  border-bottom: 4rpx solid #2c3e50;
}

.line-right-last {
  border-right: none;
}

.line-bottom-last {
  border-bottom: none;
}

.cell-selected {
  background: #b9d2f5;
}

.cell-peer {
  background: #eaf2fd;
}

.cell-same {
  background: #d5e6fb;
}

.cell-text {
  font-size: 34rpx;
  font-weight: 600;
}

.text-given {
  color: #2c3e50;
  font-weight: 700;
}

.text-filled {
  color: #2a6db8;
}

.text-conflict {
  color: #d43a2a;
  font-weight: 700;
}

.win-flag {
  display: block;
  text-align: center;
  font-size: 30rpx;
  color: #21794a;
  font-weight: 700;
  margin-top: 16rpx;
}

/* ---------- 数字键盘（横排单行） ---------- */
.pad-card {
  padding: 22rpx 12rpx;
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  row-gap: 14rpx;
  align-items: stretch;
}

.pad-btn {
  flex: 1 1 18%;
  min-width: 96rpx;
  height: 96rpx;
  border-radius: 16rpx;
  background: #f5f8fc;
  border: 1rpx solid #e2e8f0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  margin: 0 4rpx;
  box-sizing: border-box;
}

.pad-hover {
  background: #dce8f8;
  border-color: #b9cdec;
}

.pad-num {
  font-size: 28rpx;
  font-weight: 700;
  color: #2c4a7a;
  line-height: 34rpx;
}

.pad-count {
  font-size: 16rpx;
  color: #94a3b8;
  line-height: 22rpx;
}

.pad-count-done {
  color: #21794a;
  font-weight: 700;
}

.pad-erase {
  background: #f7f8fa;
}

.pad-erase-text {
  font-size: 20rpx;
  color: #55606f;
  font-weight: 600;
  line-height: 28rpx;
}

/* ---------- 底部提示 + 剩余提示开关 ---------- */
.hint {
  padding: 0 60rpx 8rpx;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.hint-toggle {
  align-self: center;
  margin-bottom: 16rpx;
  padding: 12rpx 32rpx;
  border-radius: 40rpx;
  background: #f2f6fc;
  border: 1rpx solid #d9e3f2;
}

.hint-toggle-text {
  font-size: 26rpx;
  color: #8a97ab;
  font-weight: 600;
}

.hint-toggle-on {
  background: #e3f5e9;
  border-color: #34a264;
}

.hint-toggle-on-text {
  color: #21794a;
}

.hint-text {
  font-size: 24rpx;
  color: #8a97ab;
}
</style>
