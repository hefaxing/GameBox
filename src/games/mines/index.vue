<template>
  <view class="page">
    <!-- 顶部工具栏：白底玻璃卡片，两行结构：第一行按钮(返回大厅/重置游戏)两端对齐，第二行三格统计居中 -->
    <view class="top-bar">
      <!-- 第一行：返回大厅(浅色描边胶囊) + 重置游戏(蓝紫渐变胶囊) -->
      <view class="btn-row">
        <view class="bar-btn back-btn" @click="goBack">
          <text class="back-btn-text">返回大厅</text>
        </view>
        <view class="bar-btn reset-btn" @click="resetGame()">
          <text class="reset-btn-text">重置游戏</text>
        </view>
      </view>
      <!-- 第二行：三格统计居中 -->
      <view class="stat-group">
        <view class="stat-box">
          <text class="stat-label">剩余雷</text>
          <text class="stat-value">{{ remainingMines }}</text>
        </view>
        <view class="stat-box">
          <text class="stat-label">用时</text>
          <text class="stat-value">{{ seconds }}s</text>
        </view>
        <view class="stat-box">
          <text class="stat-label">最佳</text>
          <text class="stat-value">{{ best > 0 ? best + 's' : '--' }}</text>
        </view>
      </view>
    </view>

    <!-- 难度选项卡（原生 view 切换，不新增页面） -->
    <view class="diff-bar">
      <view
        v-for="p in diffTabs"
        :key="p.id"
        class="diff-btn"
        :class="{ 'diff-active': difficulty === p.id }"
        @click="selectDiff(p.id)"
      >
        <text class="diff-btn-text">{{ p.label }}</text>
      </view>
    </view>

    <!-- 自定义布局：行/列/雷数输入 + 渐变开始按钮（仅自定义难度时展开） -->
    <view v-if="difficulty === 'custom'" class="custom-row">
      <input class="mini-input" type="number" v-model="customRows" placeholder="行 5~30" />
      <text class="custom-x">×</text>
      <input class="mini-input" type="number" v-model="customCols" placeholder="列 5~30" />
      <text class="custom-x">雷</text>
      <input class="mini-input" type="number" v-model="customMines" placeholder="雷数" />
      <view class="mini-btn" @click="applyCustom">
        <text class="mini-btn-text">开局</text>
      </view>
    </view>

    <!-- 状态条：胜利绿 / 失败红 / 默认蓝灰 胶囊 -->
    <view class="status-bar">
      <text v-if="won" class="status-text win">🎉 通关成功，用时 {{ seconds }} 秒</text>
      <text v-else-if="lost" class="status-text lose">💥 踩到地雷，本局失败</text>
      <text v-else class="status-text plain">历史最佳：{{ best > 0 ? best + ' 秒' : '--' }}</text>
    </view>

    <!-- 棋盘：easy 档沿用原格式（rpx 自适应，flex-wrap 铺满 750rpx，不滚动） -->
    <view v-if="isEasy" class="board board-easy">
      <view
        v-for="i in rows * cols"
        :key="i - 1"
        class="cell cell-easy"
        :class="cellClass(i - 1)"
        @click="onTap(i - 1)"
        @longpress="onLongPress(i - 1)"
        @mousedown="onCellMouseDown(i - 1)"
        @mouseup="cancelMouseLongPress"
        @mousemove="cancelMouseLongPress"
        @mouseleave="cancelMouseLongPress"
      >
        <text v-if="cellText(i - 1)" class="cell-text cell-text-easy">{{ cellText(i - 1) }}</text>
        <text v-else-if="isRevealed(i - 1) && counts[i - 1] > 0" :class="'num-' + counts[i - 1]" class="cell-text cell-text-easy">{{ counts[i - 1] }}</text>
      </view>
    </view>

    <!-- fixed 档（medium/hard/custom 开局后）：原生 scroll-view 双轴滚动，板宽内联 calc 精确计算 -->
    <scroll-view v-else scroll-x scroll-y class="board-scroll" :show-scrollbar="false">
      <view class="board board-fixed" :style="fixedBoardStyle">
        <view
          v-for="i in rows * cols"
          :key="i - 1"
          class="cell cell-fixed"
          :class="cellClass(i - 1)"
          @click="onTap(i - 1)"
          @longpress="onLongPress(i - 1)"
          @mousedown="onCellMouseDown(i - 1)"
          @mouseup="cancelMouseLongPress"
          @mousemove="cancelMouseLongPress"
          @mouseleave="cancelMouseLongPress"
        >
          <text v-if="cellText(i - 1)" class="cell-text cell-text-fixed">{{ cellText(i - 1) }}</text>
          <text v-else-if="isRevealed(i - 1) && counts[i - 1] > 0" :class="'num-' + counts[i - 1]" class="cell-text cell-text-fixed">{{ counts[i - 1] }}</text>
        </view>
      </view>
    </scroll-view>

    <view class="hint">
      <text class="hint-text">点击翻开格子，长按插旗🚩（长按已有旗帜可取消），翻开所有非雷格子即通关</text>
    </view>
  </view>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { onLoad, onUnload } from '@dcloudio/uni-app'

/** 难度预设（行数/列数/雷数） */
const DIFFS: Record<string, { rows: number; cols: number; mines: number }> = {
  easy: { rows: 9, cols: 9, mines: 10 },
  medium: { rows: 16, cols: 16, mines: 40 },
  hard: { rows: 16, cols: 30, mines: 99 },
}
/** 难度选项卡（原生 view 切换，不新增页面） */
const diffTabs = [
  { id: 'easy', label: '初级' },
  { id: 'medium', label: '中级' },
  { id: 'hard', label: '专家' },
  { id: 'custom', label: '自定义' },
]
/** 当前难度（easy/medium/hard/custom） */
const difficulty = ref('easy')
/** 动态棋盘尺寸（取代写死的 9/9/10） */
const rows = ref(9)
const cols = ref(9)
/** 本局雷的总数 */
const minesTotal = ref(10)
/** 自定义布局输入值（input 文本，applyCustom 时解析） */
const customRows = ref('')
const customCols = ref('')
const customMines = ref('')

/** 响应式状态：true 表示该格是雷（首次点击前尚未布雷，全为 false） */
const mines = ref<boolean[]>(new Array(rows.value * cols.value).fill(false))
/** 每格相邻雷数（首次点击布雷后计算，之前全 0） */
const counts = ref<number[]>(new Array(rows.value * cols.value).fill(0))
/** 已翻开的格子 */
const revealed = ref<boolean[]>(new Array(rows.value * cols.value).fill(false))
/** 已插旗的格子（仅未翻开格可插旗） */
const flags = ref<boolean[]>(new Array(rows.value * cols.value).fill(false))
/** 是否已开局（玩家第一次翻开后为 true，此时雷位已生成） */
const started = ref(false)
/** 是否已失败（踩雷） */
const lost = ref(false)
/** 是否已胜利 */
const won = ref(false)
/** 本局用时（秒） */
const seconds = ref(0)
/** 历史最佳用时（秒），0 表示无记录 */
const best = ref(0)
/** 计时器句柄（需在 onUnload 清理） */
let timer = 0
/** 长按守卫：记录最近一次 longpress 的格子与时刻，用于抑制 H5 上 longpress 之后伴随触发的 click，避免插旗后又翻开 */
let longPressGuardIdx = -1
let longPressGuardTs = 0
/** H5 桌面鼠标长按定时器（微信小程序端不存在 mousedown 事件，该分支不会触发） */
let mouseLongTimer = 0

/** 是否为 easy 档（easy 沿用 rpx 自适应格式不滚动；medium/hard/custom 走 rem 固定格 + scroll-view） */
const isEasy = computed(() => difficulty.value === 'easy')

/** fixed 档板内尺寸（rem/px 双端通用，inline style 不走 rpx）：宽 = 列 × 1.5rem + (列 − 1) × 3px + 左右各 12px；高同理 */
const fixedBoardStyle = computed(() => {
  const C = cols.value
  const R = rows.value
  const w = `(${C} * 1.5rem + ${(C - 1)} * 3px + 24px)`
  const h = `(${R} * 1.5rem + ${(R - 1)} * 3px + 24px)`
  return `width: calc(${w}); min-width: calc(${w}); height: calc(${h});`
})

/** 剩余雷数 = 总雷数 − 已插旗数 */
const remainingMines = computed(() => minesTotal.value - flags.value.filter(Boolean).length)

/** 返回某格（row,col 坐标制）上下左右 + 四个斜向的相邻格，最多 8 个 */
function neighborsOf(index: number): number[] {
  const C = cols.value
  const R = rows.value
  const r = Math.floor(index / C)
  const c = index % C
  const out: number[] = []
  for (let dr = -1; dr <= 1; dr++) {
    for (let dc = -1; dc <= 1; dc++) {
      if (dr === 0 && dc === 0) continue
      const nr = r + dr
      const nc = c + dc
      if (nr < 0 || nr >= R || nc < 0 || nc >= C) continue
      out.push(nr * C + nc)
    }
  }
  return out
}

/**
 * 布雷：在排除"首点格 + 其 8 邻域"之后，随机选出 MINES 个雷位。
 * 首点调用，保证第一次点击及其邻域绝对安全（首点保护）。
 */
function placeMines(safeIdx: number): void {
  const safe = new Set<number>([safeIdx, ...neighborsOf(safeIdx)])
  const pool: number[] = []
  for (let i = 0; i < rows.value * cols.value; i++) {
    if (!safe.has(i)) pool.push(i)
  }
  // Fisher-Yates 洗牌后取前 MINES 个
  for (let i = pool.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[pool[i], pool[j]] = [pool[j], pool[i]]
  }
  const m = new Array(rows.value * cols.value).fill(false)
  pool.slice(0, minesTotal.value).forEach((i) => (m[i] = true))
  mines.value = m
  // 依据雷位计算每格相邻雷数
  const c = new Array(rows.value * cols.value).fill(0)
  for (let i = 0; i < rows.value * cols.value; i++) {
    if (m[i]) continue
    for (const n of neighborsOf(i)) {
      if (m[n]) c[i] += 1
    }
  }
  counts.value = c
}

/** 递归翻开：翻开水区（计数 0）时使用 BFS 自动展开邻域 */
function reveal(idx: number): void {
  const stack: number[] = [idx]
  const rev = revealed.value.slice()
  while (stack.length > 0) {
    const i = stack.pop() as number
    if (rev[i] || flags.value[i]) continue
    rev[i] = true
    // 相邻雷数为 0 时继续向外扩散
    if (counts.value[i] === 0) {
      for (const n of neighborsOf(i)) {
        if (!rev[n] && !flags.value[n]) stack.push(n)
      }
    }
  }
  revealed.value = rev
}

/** 判断是否胜利：所有非雷格子全部翻开 */
function checkWin(): void {
  if (!started.value) return
  const nonMine = rows.value * cols.value - minesTotal.value
  if (revealed.value.filter((v, i) => v && !mines.value[i]).length >= nonMine) {
    won.value = true
    stopTimer()
    const isNewBest = best.value === 0 || seconds.value < best.value
    if (isNewBest) {
      best.value = seconds.value
      saveBest(seconds.value)
    }
    uni.showToast({
      title: isNewBest ? `🎉 通关！新最佳 ${seconds.value} 秒` : `🎉 通关！用时 ${seconds.value} 秒`,
      icon: 'none',
    })
  }
}

/** 翻到雷：判负，翻开所有雷、暴露误插的旗 */
function lose(): void {
  lost.value = true
  stopTimer()
  const rev = revealed.value.slice()
  for (let i = 0; i < rows.value * cols.value; i++) {
    if (mines.value[i]) rev[i] = true
  }
  revealed.value = rev
}

/** 普通点击：翻开格子（首点会触发布雷） */
function onTap(index: number): void {
  if (won.value || lost.value) return
  if (index === longPressGuardIdx && Date.now() - longPressGuardTs < 800) return
  if (revealed.value[index] || flags.value[index]) return
  if (!started.value) {
    placeMines(index)
    started.value = true
    startTimer()
  }
  if (mines.value[index]) {
    lose()
    return
  }
  reveal(index)
  checkWin()
}

/** 长按：未翻开格循环 无旗→红旗→无旗；已翻开格不响应 */
function onLongPress(index: number): void {
  if (won.value || lost.value) return
  if (revealed.value[index]) return
  longPressGuardIdx = index
  longPressGuardTs = Date.now()
  flags.value[index] = !flags.value[index]
}

/** H5 桌面鼠标长按兜底（按住 500ms 触发）：原生 @longpress 仅 touch/小程序侧派发的兜底路径；小程序端 mousedown 不存在，不影响 */
function onCellMouseDown(index: number): void {
  if (won.value || lost.value) return
  if (revealed.value[index]) return
  clearTimeout(mouseLongTimer)
  mouseLongTimer = setTimeout(() => {
    mouseLongTimer = 0
    onLongPress(index)
  }, 500)
}

/** 取消鼠标长按定时器（移开、抬起或移出格子时复位） */
function cancelMouseLongPress(): void {
  clearTimeout(mouseLongTimer)
  mouseLongTimer = 0
}

/** 开始计时（每秒 +1） */
function startTimer(): void {
  if (timer) return
  timer = setInterval(() => {
    seconds.value += 1
  }, 1000)
}

/** 停止计时并清理句柄 */
function stopTimer(): void {
  if (timer) {
    clearInterval(timer)
    timer = 0
  }
}

/** 重置：清空本局雷/旗/翻开/计时并重新开局（silent=true 时不弹 toast，用于切换难度/页面恢复） */
function resetGame(silent = false): void {
  stopTimer()
  mines.value = new Array(rows.value * cols.value).fill(false)
  counts.value = new Array(rows.value * cols.value).fill(0)
  revealed.value = new Array(rows.value * cols.value).fill(false)
  flags.value = new Array(rows.value * cols.value).fill(false)
  started.value = false
  lost.value = false
  won.value = false
  seconds.value = 0
  longPressGuardIdx = -1
  if (!silent) {
    uni.showToast({ title: '已重置，开冲！', icon: 'none' })
  }
}

/** 切换预设难度（easy/medium/hard）：立即生效并静默重新开局 */
function applyPreset(id: string): void {
  const d = DIFFS[id]
  if (!d) return
  difficulty.value = id
  rows.value = d.rows
  cols.value = d.cols
  minesTotal.value = d.mines
  best.value = loadBest()
  saveDifficulty()
  resetGame(true)
}

/** 难度选项卡点击：预设立即生效；custom 只置难度（页面重进时也只恢复选中、不自动开局） */
function selectDiff(id: string): void {
  if (id === 'custom') {
    if (difficulty.value !== 'custom') {
      difficulty.value = 'custom'
      best.value = loadBest()
      saveDifficulty()
    }
    return
  }
  applyPreset(id)
}

/** 自定义布局：解析输入，校验行/列 5~30、雷数 ∈ [1, 行×列−1]；不合法 toast 提示且不开局 */
function applyCustom(): void {
  const r = parseInt(customRows.value, 10)
  const c = parseInt(customCols.value, 10)
  const m = parseInt(customMines.value, 10)
  if (!(r >= 5 && r <= 30)) {
    uni.showToast({ title: '行数需在 5~30 之间', icon: 'none' })
    return
  }
  if (!(c >= 5 && c <= 30)) {
    uni.showToast({ title: '列数需在 5~30 之间', icon: 'none' })
    return
  }
  if (!(m >= 1 && m < r * c)) {
    uni.showToast({ title: '雷数需在 1~行×列−1 之间', icon: 'none' })
    return
  }
  difficulty.value = 'custom'
  rows.value = r
  cols.value = c
  minesTotal.value = m
  best.value = loadBest()
  saveDifficulty()
  resetGame(true)
}

/** 读取本地存储中的"最佳用时"记录（按难度分档：mines_best_<difficulty>） */
function loadBest(): number {
  try {
    const v = uni.getStorageSync('mines_best_' + difficulty.value)
    const n = Number(v)
    return Number.isFinite(n) && n > 0 ? n : 0
  } catch {
    return 0
  }
}

/** 写入"最佳用时"记录 */
function saveBest(v: number): void {
  try {
    uni.setStorageSync('mines_best_' + difficulty.value, v)
  } catch (e) {
    console.warn('写入最佳用时记录失败', e)
  }
}

/** 记忆当前选中难度（页面重进恢复选项卡选中态） */
function saveDifficulty(): void {
  try {
    uni.setStorageSync('mines_difficulty', difficulty.value)
  } catch {
    // 忽略存储失败，不影响游戏
  }
}

/** 返回上一页；若页面栈已到底（无法回退）则 reLaunch 到首页 */
function goBack(): void {
  uni.navigateBack({
    fail: () => {
      uni.reLaunch({ url: '/pages/index/index' })
    },
  })
}

/* ---------- 渲染辅助 ---------- */

/** 单元格状态 class：翻开显示底色，踩雷/误旗特殊标注 */
function cellClass(i: number): string {
  const cls: string[] = []
  if (revealed.value[i]) cls.push('cell-open')
  if (lost.value) {
    if (mines.value[i]) cls.push('cell-mine')
    if (flags.value[i]) cls.push('cell-wrong')
  }
  return cls.join(' ')
}

/** 单元格显示文本：已翻开的雷 / 误flag 用表情标注，其余留空给数字分支 */
function cellText(i: number): string {
  if (revealed.value[i] && mines.value[i]) return lost.value ? '💥' : '💣'
  if (lost.value && flags.value[i]) return '❌'
  if (!revealed.value[i] && flags.value[i]) return '🚩'
  return ''
}

/** 是否为已翻开格（供模板中的数字分支使用） */
function isRevealed(i: number): boolean {
  return revealed.value[i]
}

onLoad(() => {
  // 恢复上次选中难度：预设档直接开局；custom 档只恢复选中态，不自动开局
  let saved: string = ''
  try {
    saved = String(uni.getStorageSync('mines_difficulty') || '')
  } catch {
    saved = ''
  }
  if (saved === 'easy' || saved === 'medium' || saved === 'hard') {
    best.value = loadBest()
    applyPreset(saved)
  } else if (saved === 'custom') {
    difficulty.value = 'custom'
    best.value = loadBest()
  } else {
    best.value = loadBest()
  }
})

onUnload(() => {
  stopTimer()
})
</script>

<style scoped>
.page {
  min-height: 100vh;
  background-color: #f4f6fa;
  display: flex;
  flex-direction: column;
  align-items: center;
  box-sizing: border-box;
}

/* ---------- 顶部工具栏：白底玻璃感卡片（两行：按钮行 + 统计行） ---------- */
.top-bar {
  width: 92%;
  max-width: 750rpx;
  margin: 24rpx auto 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12rpx;
  padding: 16rpx 20rpx;
  box-sizing: border-box;
  background-color: rgba(255, 255, 255, 0.92);
  border: 1rpx solid rgba(255, 255, 255, 0.8);
  border-radius: 28rpx;
  box-shadow: 0 8rpx 24rpx rgba(46, 74, 122, 0.1);
  backdrop-filter: blur(10px);
}

/* 第一行按钮：两端对齐（约 146+146=292rpx ≤ 650rpx 可用宽） */
.btn-row {
  width: 100%;
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: space-between;
}

/* 第二行统计区：三盒居中（各盒 min-width 88rpx 保底，数字不被裁切） */
.stat-group {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10rpx;
}

.bar-btn {
  border-radius: 44rpx;
  padding: 12rpx 20rpx;
  display: flex;
  align-items: center;
  flex-shrink: 0;
}

.back-btn {
  background-color: #fff;
  border: 1rpx solid #c3cede;
  box-shadow: 0 4rpx 10rpx rgba(46, 74, 122, 0.08);
}

.back-btn-text {
  font-size: 26rpx;
  color: #4b576b;
}

.reset-btn {
  background: linear-gradient(135deg, #4f7cff, #8b5cff);
  box-shadow: 0 8rpx 18rpx rgba(94, 92, 255, 0.35);
}

.reset-btn-text {
  font-size: 26rpx;
  color: #fff;
  font-weight: 600;
}

.stat-box {
  background-color: #eef3fb;
  border: 1rpx solid #e0e8f4;
  border-radius: 18rpx;
  padding: 8rpx 16rpx;
  display: flex;
  flex-direction: column;
  align-items: center;
  min-width: 88rpx;
  flex-shrink: 0;
}

.stat-label {
  font-size: 20rpx;
  color: #6b7f9c;
}

.stat-value {
  font-size: 32rpx;
  font-weight: 700;
  color: #2c4a7a;
}

/* ---------- 难度选项卡：选中蓝紫渐变高亮白字 ---------- */
.diff-bar {
  width: 92%;
  max-width: 750rpx;
  display: flex;
  justify-content: space-between;
  gap: 12rpx;
  margin-top: 20rpx;
  box-sizing: border-box;
  padding: 0 8rpx;
}

.diff-btn {
  flex: 1;
  border-radius: 40rpx;
  padding: 14rpx 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #e9ecf1;
  border: 1rpx solid #dde2ea;
}

.diff-btn-text {
  font-size: 26rpx;
  color: #67707e;
}

.diff-active {
  background: linear-gradient(135deg, #4f7cff, #8b5cff);
  border-color: transparent;
  box-shadow: 0 8rpx 18rpx rgba(94, 92, 255, 0.32);
}

.diff-active .diff-btn-text {
  color: #fff;
  font-weight: 600;
}

/* ---------- 自定义行：圆角输入框 + 渐变开始按钮 ---------- */
.custom-row {
  width: 92%;
  max-width: 750rpx;
  display: flex;
  align-items: center;
  gap: 12rpx;
  margin-top: 20rpx;
  padding: 16rpx 20rpx;
  box-sizing: border-box;
  background-color: #fff;
  border: 1rpx solid #e4e9f2;
  border-radius: 22rpx;
  box-shadow: 0 6rpx 16rpx rgba(46, 74, 122, 0.06);
}

.mini-input {
  flex: 1;
  min-width: 0;
  height: 64rpx;
  line-height: 64rpx;
  background-color: #f5f7fa;
  border: 1rpx solid #d5dbe4;
  border-radius: 14rpx;
  padding: 0 18rpx;
  font-size: 26rpx;
  color: #333;
}

.custom-x {
  font-size: 26rpx;
  color: #8a93a3;
  flex-shrink: 0;
}

.mini-btn {
  flex-shrink: 0;
  border-radius: 40rpx;
  padding: 14rpx 34rpx;
  display: flex;
  align-items: center;
  background: linear-gradient(135deg, #4f7cff, #8b5cff);
  box-shadow: 0 6rpx 14rpx rgba(94, 92, 255, 0.3);
}

.mini-btn-text {
  font-size: 26rpx;
  color: #fff;
  font-weight: 600;
}

/* ---------- 状态条：胜绿 / 败红 / 默认蓝灰 胶囊 ---------- */
.status-bar {
  width: 100%;
  display: flex;
  justify-content: center;
  padding: 16rpx 40rpx 4rpx;
  box-sizing: border-box;
  min-height: 70rpx;
}

.status-text {
  display: inline-block;
  font-size: 26rpx;
  border-radius: 999rpx;
  padding: 8rpx 30rpx;
}

.status-text.plain {
  color: #5a6b8c;
  background-color: #e8eef8;
}

.status-text.win {
  color: #2f7d4f;
  background-color: #dff2e6;
  box-shadow: 0 4rpx 10rpx rgba(47, 125, 79, 0.12);
}

.status-text.lose {
  color: #b0483f;
  background-color: #fbe4e1;
  box-shadow: 0 4rpx 10rpx rgba(176, 72, 63, 0.12);
}

/* ---------- scroll-view 容器（fixed 档）：显式 62vh 高度，双轴滚动 ---------- */
.board-scroll {
  height: 62vh;
  width: 100%;
  box-sizing: border-box;
  padding: 12px;
}

/* ---------- 棋盘 ---------- */
/* easy 档：沿用原格式，9 列 × 70rpx 格子 + 8 缝 × 6rpx gap + 左右 2 × 12rpx padding = 702rpx，flex-wrap 铺满 750rpx 屏宽、不滚动 */
.board-easy {
  width: 702rpx;
  margin-top: 16rpx;
  background-color: #dde3ec;
  border-radius: 18rpx;
  padding: 12rpx;
  box-sizing: border-box;
  display: flex;
  flex-wrap: wrap;
  gap: 6rpx;
}

/* fixed 档：宽/高由内联 calc 精确给出（列 × 1.5rem + (列−1) × 3px + 24px），flex-wrap 换行铺满 */
/* margin: 0 auto 水平居中：板比容器窄时居中，比容器宽时退化为左对齐并可横向滚动（不覆盖 inline calc，无冲突） */
.board-fixed {
  background-color: #dde3ec;
  border-radius: 16px;
  padding: 12px;
  box-sizing: border-box;
  display: flex;
  flex-wrap: wrap;
  gap: 3px;
  margin: 0 auto;
}

/* easy 格：70rpx（沿用现有格式） */
.cell-easy {
  width: 70rpx;
  height: 70rpx;
  border-radius: 10rpx;
  background-color: #eceff4;
  border: 1rpx solid #c9cdd5;
  user-select: none;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* fixed 格：1.5rem（rem 在 H5 与 mp-weixin 内联/样式表均生效） */
.cell-fixed {
  width: 1.5rem;
  height: 1.5rem;
  flex-shrink: 0;
  border-radius: 8px;
  background-color: #eceff4;
  border: 1px solid #c9cdd5;
  user-select: none;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: center;
}

.cell {
  user-select: none;
}

.cell-open {
  background-color: #fff;
  border-color: #e4e7ec;
}

.cell-mine {
  background-color: #f5b5ae;
  border-color: #ec9a90;
}

.cell-wrong {
  background-color: #f7d9d2;
  border-color: #ecb5aa;
}

/* 格子文字：easy 沿用 40rpx；fixed 档约 0.9rem */
.cell-text-easy {
  font-size: 40rpx;
}

.cell-text-fixed {
  font-size: 0.9rem;
}

/* ---------- 数字 1–8 配色（经典扫雷色系，微调） ---------- */
.num-1 { color: #2a6db8; font-weight: 700; }
.num-2 { color: #3c8c4e; font-weight: 700; }
.num-3 { color: #c0392b; font-weight: 700; }
.num-4 { color: #6c4bbf; font-weight: 700; }
.num-5 { color: #a0522d; font-weight: 700; }
.num-6 { color: #1e8f8f; font-weight: 700; }
.num-7 { color: #444; font-weight: 700; }
.num-8 { color: #888; font-weight: 700; }

/* ---------- 底部提示 ---------- */
.hint {
  margin-top: 28rpx;
  padding: 0 60rpx 40rpx;
}

.hint-text {
  font-size: 24rpx;
  color: #a0a6af;
}
</style>
