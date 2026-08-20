<template>
  <view class="page">
    <!-- 顶部工具栏：返回大厅 + 当前玩家/结果 + 重置游戏 -->
    <view class="top-bar">
      <view class="bar-btn back-btn" @click="goBack">
        <text class="bar-btn-text">← 返回大厅</text>
      </view>
      <view class="stat-box">
        <text class="stat-label">状态</text>
        <text class="stat-value">{{ statusText }}</text>
      </view>
      <view class="bar-btn reset-btn" @click="resetGame">
        <text class="bar-btn-text">重置游戏</text>
      </view>
    </view>

    <!-- 模式切换：双人对战 / 人机对战（互斥，当前模式高亮） -->
    <view class="mode-bar">
      <view class="mode-btn" :class="{ active: mode === 'pvp' }" @click="switchMode('pvp')">
        <text class="mode-btn-text">双人对战</text>
      </view>
      <view class="mode-btn" :class="{ active: mode === 'ai' }" @click="switchMode('ai')">
        <text class="mode-btn-text">人机对战</text>
      </view>
    </view>

    <!-- 人机战绩（仅人机模式显示，持久化恢复） -->
    <view v-if="mode === 'ai'" class="stats-line">
      <text class="stats-text">人胜 {{ aiStats.win }} · 平 {{ aiStats.draw }} · AI胜 {{ aiStats.lose }}</text>
    </view>

    <!-- 胜负提示 -->
    <view v-if="over" class="status-bar">
      <text class="status-text">{{ resultText }}</text>
    </view>

    <!-- 15x15 棋盘 -->
    <view class="board">
      <view v-for="(row, r) in board" :key="r" class="board-row">
        <view
          v-for="(v, c) in row"
          :key="c"
          class="cell"
          :class="cellClass(r, c)"
          @click="onTap(r, c)"
        >
          <view v-if="v !== 0" class="stone" :class="'stone-' + v"></view>
        </view>
      </view>
    </view>

    <view class="hint">
      <text class="hint-text">{{ hintText }}</text>
    </view>
  </view>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { onLoad } from '@dcloudio/uni-app'

/* ==================== 常量 ==================== */

/** 棋盘边长（15x15） */
const SIZE = 15
/** 棋盘取值：0 空、1 黑、2 白 */
const EMPTY = 0
const BLACK = 1
const WHITE = 2
/** 连成几子判胜 */
const WIN_COUNT = 5
/** 中心点（fallback 落点） */
const CENTER = 7
/** AI 落子延迟（ms），模拟思考 */
const AI_DELAY_MS = 300
/** 人机战绩持久化 key */
const STATS_STORAGE_KEY = 'gomoku_ai_stats'
/** 线"不可赢"标记（线上已有对手子） */
const BLOCKED = 6
/** 四个判定方向：横、竖、左斜、右斜 */
const DIRECTIONS: [number, number][] = [
  [0, 1],
  [1, 0],
  [1, 1],
  [1, -1],
]

/**
 * 打分基数：必须大于任何单格可能的 aiScore 之和，
 * 使 myScore * BASE + aiScore 的取值顺序严格等价于
 * 字典序 (myScore, aiScore)：先比进攻分，进攻分相同比防守分。
 * 上限：4 条线 × 单线最高 20000 = 80000 < 100000。
 */
const SCORE_BASE = 100_000

type Cell = [number, number]
type Mode = 'pvp' | 'ai'
type AiStats = { win: number; lose: number; draw: number }

/** 人(黑)线加分表：索引 = 线上已有己方子数 */
const MY_LINE_SCORE: readonly number[] = [0, 200, 400, 2000, 10000]
/** AI(白)线加分表：索引 = 线上已有己方子数（防守权重整体更高） */
const AI_LINE_SCORE: readonly number[] = [0, 220, 420, 2100, 20000]

/* ==================== 全量"5 连赢法线"预计算 ==================== */

/**
 * 枚举棋盘上所有含 5 个格子的连成线：
 * 横 15×11、竖 15×11、主对角线 11×11、副对角线 11×11。
 */
function buildAllLines(): Cell[][] {
  const lines: Cell[][] = []
  const push = (cells: Cell[]): void => {
    lines.push(cells)
  }
  // 横向：15 行 × 11 个起点
  for (let r = 0; r < SIZE; r++) {
    for (let c = 0; c <= SIZE - WIN_COUNT; c++) {
      const cells: Cell[] = []
      for (let i = 0; i < WIN_COUNT; i++) cells.push([r, c + i])
      push(cells)
    }
  }
  // 纵向：15 列 × 11 个起点
  for (let c = 0; c < SIZE; c++) {
    for (let r = 0; r <= SIZE - WIN_COUNT; r++) {
      const cells: Cell[] = []
      for (let i = 0; i < WIN_COUNT; i++) cells.push([r + i, c])
      push(cells)
    }
  }
  // 主对角线（↘）：11 × 11 个起点
  for (let r = 0; r <= SIZE - WIN_COUNT; r++) {
    for (let c = 0; c <= SIZE - WIN_COUNT; c++) {
      const cells: Cell[] = []
      for (let i = 0; i < WIN_COUNT; i++) cells.push([r + i, c + i])
      push(cells)
    }
  }
  // 副对角线（↙）：11 × 11 个起点
  for (let r = 0; r <= SIZE - WIN_COUNT; r++) {
    for (let c = WIN_COUNT - 1; c < SIZE; c++) {
      const cells: Cell[] = []
      for (let i = 0; i < WIN_COUNT; i++) cells.push([r + i, c - i])
      push(cells)
    }
  }
  return lines
}

/** 全部赢法线（横/竖/主斜/副斜），只枚举一次 */
const ALL_LINES: readonly Cell[][] = buildAllLines()
const LINE_COUNT: number = ALL_LINES.length

/** 反向索引：CELL_LINES[r][c] = 经过该格的所有线编号 */
const CELL_LINES: number[][][] = (() => {
  const idx: number[][][] = []
  for (let r = 0; r < SIZE; r++) {
    const row: number[][] = []
    for (let c = 0; c < SIZE; c++) row.push([])
    idx.push(row)
  }
  for (let k = 0; k < ALL_LINES.length; k++) {
    const cells = ALL_LINES[k]
    for (let i = 0; i < cells.length; i++) {
      const [r, c] = cells[i]
      idx[r][c].push(k)
    }
  }
  return idx
})()

/* ==================== 棋盘与对局状态 ==================== */

/** 生成空棋盘 */
function createEmptyBoard(): number[][] {
  const b: number[][] = []
  for (let r = 0; r < SIZE; r++) {
    const row: number[] = []
    for (let c = 0; c < SIZE; c++) row.push(EMPTY)
    b.push(row)
  }
  return b
}

/** 响应式棋盘（二维数组） */
const board = ref<number[][]>(createEmptyBoard())
/** 当前落子方：1 黑（先手）/ 2 白 */
const current = ref(BLACK)
/** 是否已结束（胜、负、和） */
const over = ref(false)
/** 获胜方：0 无、1 黑、2 白（和棋时保持 0） */
const winner = ref(0)
/** 获胜连线位置列表 "r-c"，用于高亮 */
const winLine = ref<string[]>([])
/** 已落子总数 */
const moves = ref(0)

/** 对局模式：pvp 双人 / ai 人机（人=黑、AI=白） */
const mode = ref<Mode>('pvp')
/** AI 正在"思考"（倒计时中），锁定人类点击 */
const aiThinking = ref(false)
let aiTimer: ReturnType<typeof setTimeout> | null = null

/** 人机战绩：win=人胜、lose=AI胜、draw=平局 */
const aiStats = ref<AiStats>({ win: 0, lose: 0, draw: 0 })

/* ==================== 人机战绩持久化 ==================== */

function toNonNegInt(v: unknown): number {
  return typeof v === 'number' && Number.isFinite(v) && v >= 0 ? Math.floor(v) : 0
}

/** onLoad 时从本地存储恢复人机战绩 */
function loadStats(): void {
  try {
    const raw: unknown = uni.getStorageSync(STATS_STORAGE_KEY)
    if (raw && typeof raw === 'object') {
      const o = raw as { win?: unknown; lose?: unknown; draw?: unknown }
      aiStats.value = {
        win: toNonNegInt(o.win),
        lose: toNonNegInt(o.lose),
        draw: toNonNegInt(o.draw),
      }
    }
  } catch (e) {
    // 存储不可用时保持默认零值
  }
}

function saveStats(): void {
  try {
    uni.setStorageSync(STATS_STORAGE_KEY, aiStats.value)
  } catch (e) {
    // 忽略写入失败，不影响对局
  }
}

onLoad(() => {
  loadStats()
})

/* ==================== AI 启发式引擎（经典"赢法线计分"） ==================== */
/*
 * 每条 5 连线维护两个计数：
 *   myWin[k]  —— 人(黑)在该线的子数；若线上已有 AI(白)子则记 BLOCKED(6)
 *   aiWin[k]  —— AI(白)在该线的子数；若线上已有 人(黑)子则记 BLOCKED(6)
 * 空位打分：
 *   myScore = Σ 该位所属线加分（己方 1/2/3/4 连 → 200/400/2000/10000）
 *   aiScore = Σ 同理（防守 1/2/3/4 连 → 220/420/2100/20000）
 * 选点：字典序 max(myScore, aiScore)，即进攻分优先、防守分 tie-break，
 * 效果上优先"自己成五/堵住对方成五"的威胁点。
 * 全 0 分时回落：离已有棋子最近的空位，再无则中心 (7,7)。
 */

/** myWin/aiWin 为非响应式内部数组（不进入渲染） */
let myWin: number[] = []
let aiWin: number[] = []

/** 按当前棋盘状态重新计一条线上指定一方的子数（含不可赢判定） */
function recountLine(k: number, own: number, opp: number): number {
  const cells = ALL_LINES[k]
  let count = 0
  for (let i = 0; i < cells.length; i++) {
    const r = cells[i][0]
    const c = cells[i][1]
    const v = board.value[r][c]
    if (v === opp) return BLOCKED
    if (v === own) count++
  }
  return count
}

/** 落子后仅重算经过该格的线（O(cellLines) 增量更新） */
function updateAiStateAt(r: number, c: number): void {
  const ks = CELL_LINES[r][c]
  for (let i = 0; i < ks.length; i++) {
    const k = ks[i]
    myWin[k] = recountLine(k, BLACK, WHITE)
    aiWin[k] = recountLine(k, WHITE, BLACK)
  }
}

/** 重置/切换模式时整体重建：按当前棋盘全量重算两条数组 */
function rebuildAiState(): void {
  if (myWin.length !== LINE_COUNT) {
    myWin = new Array<number>(LINE_COUNT).fill(0)
    aiWin = new Array<number>(LINE_COUNT).fill(0)
  } else {
    myWin.fill(0)
    aiWin.fill(0)
  }
  for (let k = 0; k < LINE_COUNT; k++) {
    myWin[k] = recountLine(k, BLACK, WHITE)
    aiWin[k] = recountLine(k, WHITE, BLACK)
  }
}

/** 单个空位的综合分数（myScore 主分 + aiScore tie-break 合并为整数比较） */
function scoreEmptyCell(r: number, c: number): number {
  let my = 0
  let ai = 0
  const ks = CELL_LINES[r][c]
  for (let i = 0; i < ks.length; i++) {
    const k = ks[i]
    const m = myWin[k]
    if (m >= 0 && m < MY_LINE_SCORE.length) my += MY_LINE_SCORE[m]
    const a = aiWin[k]
    if (a >= 0 && a < AI_LINE_SCORE.length) ai += AI_LINE_SCORE[a]
  }
  return my * SCORE_BASE + ai
}

/** AI 选点：最高 (进攻分, 防守分) 空位；全 0 时离最近棋子；无棋子则中心 */
function chooseAiMove(b: number[][]): Cell {
  let bestR = -1
  let bestC = -1
  let bestScore = 0
  let hasStone = false
  for (let r = 0; r < SIZE; r++) {
    for (let c = 0; c < SIZE; c++) {
      if (b[r][c] !== EMPTY) {
        hasStone = true
        continue
      }
      const s = scoreEmptyCell(r, c)
      if (s > bestScore) {
        bestScore = s
        bestR = r
        bestC = c
      }
    }
  }
  if (bestR !== -1) return [bestR, bestC]
  if (!hasStone) return [CENTER, CENTER]
  // Fallback：与已有棋子曼哈顿距离最近的空位（首个达到最小距离者）
  let fallbackR = -1
  let fallbackC = -1
  let minDist = Infinity
  for (let r = 0; r < SIZE; r++) {
    for (let c = 0; c < SIZE; c++) {
      if (b[r][c] !== EMPTY) continue
      for (let sr = 0; sr < SIZE; sr++) {
        for (let sc = 0; sc < SIZE; sc++) {
          if (b[sr][sc] === EMPTY) continue
          const d = Math.abs(sr - r) + Math.abs(sc - c)
          if (d < minDist) {
            minDist = d
            fallbackR = r
            fallbackC = c
          }
        }
      }
    }
  }
  return [fallbackR, fallbackC]
}

/* ==================== 对局流程 ==================== */

/** 取消未触发的 AI 落子定时器 */
function cancelAiTimer(): void {
  if (aiTimer !== null) {
    clearTimeout(aiTimer)
    aiTimer = null
  }
}

/** 调度 AI 一步：延迟 300ms 后落子（模拟思考，避免阻塞） */
function scheduleAiMove(): void {
  cancelAiTimer()
  aiThinking.value = true
  aiTimer = setTimeout(() => {
    aiTimer = null
    aiThinking.value = false
    if (over.value || mode.value !== 'ai' || current.value !== WHITE) return
    const [r, c] = chooseAiMove(board.value)
    placeStone(r, c, WHITE)
  }, AI_DELAY_MS)
}

/** 人类点击格子落子（双/人机通用入口，含各种锁定守卫） */
function onTap(r: number, c: number): void {
  if (over.value) return
  if (aiThinking.value) return
  if (board.value[r][c] !== EMPTY) return
  if (mode.value === 'ai') {
    // 人=黑、AI=白：轮到 AI（白）时禁止人类落子
    if (current.value === WHITE) return
    placeStone(r, c, BLACK)
  } else {
    placeStone(r, c, current.value)
  }
}

/** 落子核心：写入棋盘 → 增量更新 AI 状态 → 判胜/判和 → 换手与 AI 调度 */
function placeStone(r: number, c: number, color: number): void {
  if (board.value[r][c] !== EMPTY) return
  board.value[r][c] = color
  moves.value += 1
  if (mode.value === 'ai') updateAiStateAt(r, c)

  const line = findWinLine(board.value, r, c)
  if (line) {
    winner.value = color
    winLine.value = line
    over.value = true
    if (mode.value === 'ai') {
      if (color === BLACK) aiStats.value.win += 1
      else aiStats.value.lose += 1
      saveStats()
    }
    uni.showToast({
      title:
        mode.value === 'ai'
          ? color === BLACK
            ? '🎉 你赢了！五子连珠'
            : '🤖 AI 获胜！五子连珠'
          : color === BLACK
            ? '黑方胜利！'
            : '白方胜利！',
      icon: 'none',
    })
    return
  }

  if (moves.value === SIZE * SIZE) {
    over.value = true
    if (mode.value === 'ai') {
      aiStats.value.draw += 1
      saveStats()
    }
    uni.showToast({ title: '和棋：棋盘已下满', icon: 'none' })
    return
  }

  current.value = color === BLACK ? WHITE : BLACK
  if (mode.value === 'ai' && current.value === WHITE) scheduleAiMove()
}

/** 清空棋盘/当前方/胜负/高亮/思考锁（战绩不清零），AI 线计数按当前棋盘全量重建 */
function clearGame(): void {
  cancelAiTimer()
  aiThinking.value = false
  board.value = createEmptyBoard()
  current.value = BLACK
  over.value = false
  winner.value = 0
  winLine.value = []
  moves.value = 0
  rebuildAiState()
}

/** 切换模式：互斥；按重置语义清局并提示 */
function switchMode(target: Mode): void {
  if (mode.value === target) return
  mode.value = target
  clearGame()
  uni.showToast({
    title: mode.value === 'ai' ? '已切换人机对战，你执黑先行' : '已切换双人对战，黑方先行',
    icon: 'none',
  })
}

/** 重置：清棋盘/当前方/胜负/高亮/lock；人机模式回到黑(人)先行；不清战绩 */
function resetGame(): void {
  clearGame()
  uni.showToast({
    title: mode.value === 'ai' ? '已重置，你执黑先行' : '已重置，黑方先行',
    icon: 'none',
  })
}

/** 返回上一页；若页面栈已到底（无法回退）则 reLaunch 到首页 */
function goBack(): void {
  uni.navigateBack({
    fail: () => {
      uni.reLaunch({ url: '/pages/index/index' })
    },
  })
}

/* ==================== 判定与展示 ==================== */

/** 在棋盘内判断坐标 */
function inBoard(r: number, c: number): boolean {
  return r >= 0 && r < SIZE && c >= 0 && c < SIZE
}

/**
 * 胜负判定：从 (r,c) 沿四个方向双向统计同色连续子。
 * 任一方向数量 >= WIN_COUNT 则判定获胜，返回高亮位置列表（含 (r,c) 的 5+ 连）。
 */
function findWinLine(b: number[][], r: number, c: number): string[] | null {
  const color = b[r][c]
  for (const [dr, dc] of DIRECTIONS) {
    const line: string[] = [r + '-' + c]
    // 正向
    let nr = r + dr
    let nc = c + dc
    while (inBoard(nr, nc) && b[nr][nc] === color) {
      line.push(nr + '-' + nc)
      nr += dr
      nc += dc
    }
    // 反向
    nr = r - dr
    nc = c - dc
    while (inBoard(nr, nc) && b[nr][nc] === color) {
      line.unshift(nr + '-' + nc)
      nr -= dr
      nc -= dc
    }
    if (line.length >= WIN_COUNT) return line
  }
  return null
}

/** 生成格子的附加 class（最后列/行去边框 + 胜利线高亮） */
function cellClass(r: number, c: number): string {
  const cls: string[] = []
  if (c === SIZE - 1) cls.push('no-right')
  if (r === SIZE - 1) cls.push('no-bottom')
  if (winLine.value.indexOf(r + '-' + c) !== -1) cls.push('win')
  return cls.join(' ')
}

/** 顶部状态文案 */
const statusText = computed<string>(() => {
  if (over.value) {
    if (winner.value === BLACK) return mode.value === 'ai' ? '你胜' : '● 黑胜'
    if (winner.value === WHITE) return mode.value === 'ai' ? 'AI 胜' : '○ 白胜'
    return '和棋'
  }
  if (mode.value === 'ai') {
    return aiThinking.value || current.value === WHITE ? 'AI 思考中…' : '● 你的回合'
  }
  return current.value === BLACK ? '● 黑先' : '○ 白方'
})

/** 结果横幅文案（含和棋） */
const resultText = computed<string>(() => {
  if (winner.value === BLACK) {
    return mode.value === 'ai' ? '🎉 你赢了！五子连珠' : '🎉 黑方胜利！五子连珠'
  }
  if (winner.value === WHITE) {
    return mode.value === 'ai' ? '🤖 AI 获胜！五子连珠' : '🎉 白方胜利！五子连珠'
  }
  return '和棋：棋盘已下满'
})

/** 底部规则提示（随模式变化） */
const hintText = computed<string>(() => {
  return mode.value === 'ai'
    ? '你执黑先行，AI 执白应战；横竖斜任意方向连成五子者胜'
    : '黑先白后，横竖斜任意方向连成五子者胜'
})
</script>

<style scoped>
.page {
  min-height: 100vh;
  background-color: #f6f7f9;
  display: flex;
  flex-direction: column;
  align-items: center;
  box-sizing: border-box;
}

/* ---------- 顶部工具栏 ---------- */
.top-bar {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 36rpx 32rpx 20rpx;
  box-sizing: border-box;
  gap: 16rpx;
}

.bar-btn {
  border-radius: 14rpx;
  padding: 16rpx 22rpx;
}

.back-btn {
  background-color: #fff;
  border: 1rpx solid #d5d8dc;
}

.bar-btn-text {
  font-size: 28rpx;
  color: #555;
}

.reset-btn {
  background-color: #3f68a8;
}

.reset-btn .bar-btn-text {
  color: #fff;
  font-weight: 600;
}

.stat-box {
  background-color: #eef3fa;
  border-radius: 14rpx;
  padding: 10rpx 28rpx;
  display: flex;
  flex-direction: column;
  align-items: center;
  min-width: 140rpx;
}

.stat-label {
  font-size: 20rpx;
  color: #6b7f9c;
}

.stat-value {
  font-size: 28rpx;
  font-weight: 700;
  color: #2c4a7a;
}

/* ---------- 模式切换 ---------- */
.mode-bar {
  display: flex;
  flex-direction: row;
  gap: 20rpx;
  margin-top: 16rpx;
}

.mode-btn {
  border-radius: 999rpx;
  padding: 14rpx 44rpx;
  background-color: #fff;
  border: 1rpx solid #d5d8dc;
}

.mode-btn.active {
  background-color: #3f68a8;
  border-color: #3f68a8;
}

.mode-btn-text {
  font-size: 28rpx;
  color: #555;
}

.mode-btn.active .mode-btn-text {
  color: #fff;
  font-weight: 600;
}

/* ---------- 人机战绩 ---------- */
.stats-line {
  margin-top: 16rpx;
  padding: 0 60rpx;
}

.stats-text {
  font-size: 24rpx;
  color: #6b7f9c;
}

/* ---------- 胜负提示 ---------- */
.status-bar {
  width: 100%;
  padding: 0 40rpx 12rpx;
  box-sizing: border-box;
}

.status-text {
  display: inline-block;
  font-size: 26rpx;
  color: #4a7b50;
  background-color: #e5f3e8;
  border-radius: 10rpx;
  padding: 8rpx 20rpx;
}

/* ---------- 棋盘 ---------- */
.board {
  margin-top: 20rpx;
  display: flex;
  flex-direction: column;
  background-color: #ecd29a;
  /* 外侧两条线：左侧 + 顶部（右侧/底部由末列/末行格子的边框补上） */
  /* 外框四条边，末列/末行格子自身已去掉右边框/下边框，避免双线重叠 */
  border: 1rpx solid #7a5c33;
  border-radius: 8rpx;
  overflow: hidden;
}

.board-row {
  display: flex;
  flex-direction: row;
}

.cell {
  width: 44rpx;
  height: 44rpx;
  box-sizing: border-box;
  border-right: 1rpx solid #7a5c33;
  border-bottom: 1rpx solid #7a5c33;
  display: flex;
  align-items: center;
  justify-content: center;
}

.no-right {
  border-right: none;
}

.no-bottom {
  border-bottom: none;
}

/* ---------- 棋子 ---------- */
.stone {
  width: 36rpx;
  height: 36rpx;
  border-radius: 50%;
}

.stone-1 {
  background-color: #1c1c1c;
  box-shadow: 0 2rpx 4rpx rgba(0, 0, 0, 0.35);
}

.stone-2 {
  background-color: #fdfdfd;
  border: 1rpx solid #b9b9b9;
  box-shadow: 0 2rpx 4rpx rgba(0, 0, 0, 0.2);
}

/* ---------- 获胜连线高亮 ---------- */
.win .stone {
  box-shadow: 0 0 0 4rpx #e6a23c, 0 2rpx 6rpx rgba(0, 0, 0, 0.3);
}

/* ---------- 底部提示 ---------- */
.hint {
  margin-top: 28rpx;
  padding: 0 60rpx;
}

.hint-text {
  font-size: 24rpx;
  color: #a0a6af;
}
</style>
