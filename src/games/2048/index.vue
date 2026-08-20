<template>
  <view class="game-page">
    <!-- 顶部信息栏：返回 + 标题 + 重置按钮 -->
    <view class="top-bar">
      <view class="pill-btn back-btn" :hover-class="pill-btn-press" @click="goBack">
        <text class="pill-text">← 大厅</text>
      </view>
      <text class="page-title">2048</text>
      <view class="pill-btn reset-btn" :hover-class="pill-btn-press" @click="reset">
        <text class="pill-text">重置游戏</text>
      </view>
    </view>

    <!-- 分数 / 最高分（双格并列） -->
    <view class="score-row">
      <view class="score-box">
        <text class="score-label">分数</text>
        <text class="score-value">{{ score }}</text>
      </view>
      <view class="score-box">
        <text class="score-label">最高分</text>
        <text class="score-value">{{ best }}</text>
      </view>
    </view>

    <!-- 状态提示（胜利 / 结束） -->
    <view v-if="status" class="status-bar">
      <text class="status-text" :class="{ 'status-over': over }">{{ status }}</text>
    </view>

    <!-- 控制台线框容器（纵横线条大框：棋盘 + 方向键） -->
    <view class="frame">
    <!-- 4x4 棋盘 -->
    <view
      class="board"
      @touchstart="onTouchStart"
      @touchmove="onTouchMove"
      @touchend="onTouchEnd"
    >
      <view v-for="(row, r) in board" :key="r" class="board-row">
        <view
          v-for="(n, c) in row"
          :key="c"
          class="cell"
          :class="n > 0 ? tileClass(n) : 'cell-empty'"
        >
          <text v-if="n > 0" class="cell-text">{{ n }}</text>
        </view>
      </view>
    </view>

    <!-- 横分隔线 -->
    <view class="frame-line"></view>

    <!-- 方向按钮组（九宫格十字布局，带纵横网格线） -->
    <view class="dir-pad">
      <view class="dir-row">
        <view class="dir-slot dir-corner"></view>
        <view class="dir-slot">
          <view class="dir-btn" :hover-class="dir-btn-press" @click="move('up')">
            <text class="dir-text">▲</text>
          </view>
        </view>
        <view class="dir-slot dir-corner"></view>
      </view>
      <view class="dir-row">
        <view class="dir-slot">
          <view class="dir-btn" :hover-class="dir-btn-press" @click="move('left')">
            <text class="dir-text">◀</text>
          </view>
        </view>
        <view class="dir-slot">
          <view class="dir-core">
            <text class="dir-core-text">2048</text>
          </view>
        </view>
        <view class="dir-slot">
          <view class="dir-btn" :hover-class="dir-btn-press" @click="move('right')">
            <text class="dir-text">▶</text>
          </view>
        </view>
      </view>
      <view class="dir-row">
        <view class="dir-slot dir-corner"></view>
        <view class="dir-slot">
          <view class="dir-btn" :hover-class="dir-btn-press" @click="move('down')">
            <text class="dir-text">▼</text>
          </view>
        </view>
        <view class="dir-slot dir-corner"></view>
      </view>
    </view>
    </view>

    <view class="hint">
      <text class="hint-text">在棋盘上滑动或点击方向按钮移动数字，相同数字合并得分</text>
    </view>
  </view>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { onLoad } from '@dcloudio/uni-app'

/** 棋盘为 4x4 二维数字矩阵，0 表示空格 */
type Board = number[][]
/** 滑动方向 */
type Dir = 'left' | 'right' | 'up' | 'down'

const SIZE = 4
/** 判定滑动的最小位移（px） */
const SLIDE_THRESHOLD = 40
/** 数字块达到该值即胜利 */
const WIN_VALUE = 2048
/** 本地存储键 */
const STORAGE_KEY_SCORE = 'game_2048_score'
const STORAGE_KEY_BEST = 'game_2048_best'

function createEmptyBoard(): Board {
  return Array.from({ length: SIZE }, () => Array<number>(SIZE).fill(0))
}

/** 响应式棋盘数据 */
const board = ref<Board>(createEmptyBoard())
/** 当前分数 */
const score = ref(0)
/** 是否游戏结束（棋盘满且无可合并） */
const over = ref(false)
/** 是否已出现 WIN_VALUE */
const won = ref(false)
/** 最高分 */
const best = ref(0)
/** 起点坐标（touchstart 时记录） */
let startTouch: { x: number; y: number } | null = null
/** touchmove 阶段记录的最后触点坐标（touchend 回退用） */
let lastTouch: { x: number; y: number } | null = null

/** 安全读取数字存储值 */
function readNum(key: string): number {
  try {
    const v = uni.getStorageSync(key)
    return typeof v === 'number' && Number.isFinite(v) ? v : 0
  } catch (e) {
    return 0
  }
}

/** 写入数字存储值 */
function writeNum(key: string, value: number): void {
  try {
    uni.setStorageSync(key, value)
  } catch (e) {
    /* 忽略存储失败 */
  }
}

/** 状态提示文案 */
const status = computed<string>(() => {
  if (over.value) return '游戏结束，点击“重新开始”再来一局'
  if (won.value) return '达成 2048！继续挑战更高分吧'
  return ''
})

/** 数字块背景色 class */
function tileClass(n: number): string {
  const sizeClass = n <= WIN_VALUE ? `tile-${n}` : 'tile-super'
  const darkText = n === 2 || n === 4 ? ' tile-dark' : ''
  const longNum = n >= 1000 ? ' tile-long' : ''
  return `${sizeClass}${darkText}${longNum}`
}

/** 随机生成数字：在空格随机位置放入 2（90%）或 4（10%） */
function randomAddTile(b: Board): void {
  const empties: Array<{ r: number; c: number }> = []
  b.forEach((row, r) =>
    row.forEach((n, c) => {
      if (n === 0) empties.push({ r, c })
    }),
  )
  if (empties.length === 0) return
  const pos = empties[Math.floor(Math.random() * empties.length)]
  b[pos.r][pos.c] = Math.random() < 0.9 ? 2 : 4
}

/** 重置棋盘：回到随机 2 块的初始状态（保留最高分） */
function reset(): void {
  board.value = createEmptyBoard()
  randomAddTile(board.value)
  randomAddTile(board.value)
  score.value = 0
  over.value = false
  won.value = false
  writeNum(STORAGE_KEY_SCORE, 0)
  uni.showToast({ title: '新的一局开始！', icon: 'none' })
}

/** 返回大厅：有上一页则回退，否则兜底重定向到首页 */
function goBack(): void {
  try {
    if (getCurrentPages().length > 1) {
      uni.navigateBack({ delta: 1 })
      return
    }
  } catch (e) {
    /* 直接走兜底 */
  }
  try {
    uni.switchTab({ url: '/pages/index/index' })
  } catch (e) {
    uni.reLaunch({ url: '/pages/index/index' })
  }
}

/** 将单行向左滑动并合并，返回合并后行与得分 */
function slideRowLeft(row: number[]): { row: number[]; gained: number } {
  const vals = row.filter((n) => n > 0)
  const out: number[] = []
  let gained = 0
  for (let i = 0; i < vals.length; i++) {
    // 与右侧相邻且相等则合并
    if (i + 1 < vals.length && vals[i] === vals[i + 1]) {
      out.push(vals[i] * 2)
      gained += vals[i] * 2
      i++
    } else {
      out.push(vals[i])
    }
  }
  while (out.length < SIZE) out.push(0)
  return { row: out, gained }
}

/** 两棋盘是否完全相同 */
function boardEqual(a: Board, b: Board): boolean {
  for (let r = 0; r < SIZE; r++) {
    for (let c = 0; c < SIZE; c++) {
      if (a[r][c] !== b[r][c]) return false
    }
  }
  return true
}

/** 棋盘是否已满且无可合并（游戏结束） */
function isOver(): boolean {
  const b = board.value
  for (let r = 0; r < SIZE; r++) {
    for (let c = 0; c < SIZE; c++) {
      const n = b[r][c]
      if (n === 0) return false
      if (c + 1 < SIZE && b[r][c + 1] === n) return false
      if (r + 1 < SIZE && b[r + 1][c] === n) return false
    }
  }
  return true
}

/** 按方向移动：滑动 + 合并 + 生成新块 + 计分 + 胜负判断 */
function move(dir: Dir): void {
  if (over.value) return
  const before = board.value.map((row) => [...row])
  const next = createEmptyBoard()
  let gained = 0

  if (dir === 'left' || dir === 'right') {
    for (let r = 0; r < SIZE; r++) {
      const row = before[r].slice()
      if (dir === 'right') row.reverse()
      const res = slideRowLeft(row)
      if (dir === 'right') res.row.reverse()
      next[r] = res.row
      gained += res.gained
    }
  } else {
    for (let c = 0; c < SIZE; c++) {
      const col = [
        before[0][c],
        before[1][c],
        before[2][c],
        before[3][c],
      ]
      if (dir === 'down') col.reverse()
      const res = slideRowLeft(col)
      if (dir === 'down') res.row.reverse()
      for (let r = 0; r < SIZE; r++) next[r][c] = res.row[r]
      gained += res.gained
    }
  }

  // 该方向无变化则忽略本次滑动
  if (boardEqual(next, before)) return

  board.value = next
  score.value += gained
  randomAddTile(board.value)

  if (!won.value && board.value.some((row) => row.some((n) => n >= WIN_VALUE))) {
    won.value = true
    uni.showToast({ title: '🎉 达成 2048！', icon: 'none' })
  }
  if (isOver()) over.value = true
}

/* ---------------- 触摸手势（手写，兼容 H5 与微信小程序） ---------------- */

/** 跨端安全地读取触点坐标：touches → changedTouches → clientX → detail */
function readTouchPoint(e: TouchEvent): { x: number; y: number } | null {
  const ev = e as any
  const arr =
    (ev.touches && ev.touches[0]) ||
    (ev.changedTouches && ev.changedTouches[0])
  if (arr && typeof arr.clientX === 'number') {
    return { x: arr.clientX, y: arr.clientY }
  }
  if (typeof ev.clientX === 'number' && typeof ev.clientY === 'number') {
    return { x: ev.clientX, y: ev.clientY }
  }
  if (ev.detail && typeof ev.detail.clientX === 'number') {
    return { x: ev.detail.clientX, y: ev.detail.clientY }
  }
  return null
}

function onTouchStart(e: TouchEvent): void {
  const p = readTouchPoint(e)
  if (!p) return
  startTouch = p
  lastTouch = p
}

function onTouchMove(e: TouchEvent): void {
  const p = readTouchPoint(e)
  if (p) lastTouch = p
}

function onTouchEnd(e: TouchEvent): void {
  const s = startTouch
  // touchend 时 touches 可能为空，优先事件坐标，回退 touchmove 记录的最后触点
  const end = readTouchPoint(e) || lastTouch
  // 先重置本次触摸状态，避免残留影响下次触摸
  startTouch = null
  lastTouch = null
  if (!s || !end) return
  const dx = end.x - s.x
  const dy = end.y - s.y
  // 位移小于阈值（快速轻点 / 误触）不触发移动
  if (Math.max(Math.abs(dx), Math.abs(dy)) < SLIDE_THRESHOLD) return
  // 取绝对值更大的轴作为滑动方向，调用按钮同款 move()
  if (Math.abs(dx) > Math.abs(dy)) {
    move(dx > 0 ? 'right' : 'left')
  } else {
    move(dy > 0 ? 'down' : 'up')
  }
}

/* ---------------- 生命周期 ---------------- */

onLoad(() => {
  // 初始化：随机放 2 个数字块
  reset()
})
</script>

<style scoped>
.game-page {
  min-height: 100vh;
  background-color: #faf8ef;
  display: flex;
  flex-direction: column;
  align-items: center;
  box-sizing: border-box;
}

/* ---------- 顶部栏 ---------- */
.top-bar {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 40rpx 48rpx 12rpx;
  box-sizing: border-box;
}

.page-title {
  font-size: 52rpx;
  font-weight: 800;
  letter-spacing: 8rpx;
  color: #bbada0;
}

/* 胶囊按钮（返回大厅 / 重置游戏） */
.pill-btn {
  background-color: #f2eadf;
  border: 2rpx solid #e2d8c9;
  border-radius: 40rpx;
  padding: 14rpx 32rpx;
}

.pill-text {
  font-size: 28rpx;
  font-weight: 600;
  color: #8f7a66;
}

/* 按下反馈（:hover-class，H5 与小程序通用） */
.pill-btn-press {
  background-color: #8f7a66;
  border-color: #8f7a66;
}

.pill-btn-press .pill-text {
  color: #ffffff;
}

/* ---------- 分数 / 最高分 ---------- */
.score-row {
  width: 100%;
  display: flex;
  justify-content: center;
  padding: 4rpx 48rpx 0;
  box-sizing: border-box;
}

.score-box {
  background-color: #bbada0;
  border-radius: 18rpx;
  padding: 14rpx 40rpx;
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 280rpx;
}

.score-box + .score-box {
  margin-left: 24rpx;
}

.score-label {
  font-size: 22rpx;
  color: #eee4da;
}

.score-value {
  font-size: 44rpx;
  font-weight: 700;
  color: #fff;
  line-height: 1.2;
}

/* ---------- 状态栏 ---------- */
.status-bar {
  width: 100%;
  padding: 0 48rpx 16rpx;
  text-align: center;
}

.status-text {
  display: inline-block;
  font-size: 26rpx;
  color: #4a7b50;
  background-color: #e5f3e8;
  border-radius: 10rpx;
  padding: 8rpx 20rpx;
}

.status-over {
  color: #a3453e;
  background-color: #f6e3e1;
}

/* ---------- 控制台线框容器（大框：棋盘 + 方向键） ---------- */
.frame {
  width: 704rpx;
  margin-top: 24rpx;
  background-color: #f6f0e4;
  border: 4rpx solid #8f7a66;
  border-radius: 28rpx;
  box-sizing: border-box;
  padding: 20rpx 20rpx 28rpx;
}

/* 棋盘与方向键之间的横分隔线 */
.frame-line {
  height: 4rpx;
  background-color: #8f7a66;
  margin: 30rpx 0 0;
}

/* ---------- 棋盘（4 纵 4 横线框：底色即线条色，gap 透出 3 横 3 纵，外圈统一描边） ---------- */
.board {
  width: 614rpx;
  margin: 0 auto;
  background-color: #cabbab;
  border: 4rpx solid #8f7a66;
  border-radius: 18rpx;
  padding: 10rpx;
  box-sizing: border-box;
}

.board-row {
  display: flex;
  margin-bottom: 10rpx;
}

.board-row:last-child {
  margin-bottom: 0;
}

.cell {
  width: 139rpx;
  height: 139rpx;
  margin-right: 10rpx;
  border-radius: 12rpx;
  display: flex;
  align-items: center;
  justify-content: center;
}

.cell:last-child {
  margin-right: 0;
}

.cell-empty {
  background-color: #f2e9db;
}

.cell-text {
  font-size: 46rpx;
  font-weight: 700;
  color: #f9f6f2;
}

/* 2 和 4 用小号颜色背景、深号文字 */
.tile-dark .cell-text {
  color: #776860;
}

/* 4 位数字缩小字号 */
.tile-long .cell-text {
  font-size: 34rpx;
}

/* ---------- 各数字块颜色（经典 2048 配色） ---------- */
.tile-2 {
  background-color: #eee4da;
}
.tile-4 {
  background-color: #ede0c8;
}
.tile-8 {
  background-color: #f2b179;
}
.tile-16 {
  background-color: #f59563;
}
.tile-32 {
  background-color: #f67c5f;
}
.tile-64 {
  background-color: #f65e3b;
}
.tile-128 {
  background-color: #edcf72;
}
.tile-256 {
  background-color: #edcc61;
}
.tile-512 {
  background-color: #edc850;
}
.tile-1024 {
  background-color: #edc53f;
}
.tile-2048 {
  background-color: #edc22e;
}
/* 超过 2048（4096 等） */
.tile-super {
  background-color: #3c3a32;
}

/* ---------- 方向键（九宫格十字布局 + 纵横网格线） ---------- */
.dir-pad {
  width: 368rpx;
  margin: 30rpx auto 0;
  border: 4rpx solid #8f7a66;
  border-radius: 18rpx;
  overflow: hidden;
  box-sizing: border-box;
}

.dir-row {
  display: flex;
  align-items: stretch;
}

/* 九宫格单元格：外框 + 纵横分隔线构成网格 */
.dir-slot {
  width: 120rpx;
  height: 120rpx;
  box-sizing: border-box;
  border-right: 4rpx solid #d3c9ba;
  border-bottom: 4rpx solid #d3c9ba;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #f6f0e4;
}

.dir-slot:last-child {
  border-right: 0;
}

.dir-row:last-child .dir-slot {
  border-bottom: 0;
}

/* 四角装饰格 */
.dir-corner {
  background-color: #efe7d8;
}

/* 方向按钮：白底棕字，圆角柔和 */
.dir-btn {
  width: 100%;
  height: 100%;
  background-color: #fffdf7;
  display: flex;
  align-items: center;
  justify-content: center;
}

.dir-text {
  font-size: 44rpx;
  font-weight: 700;
  color: #8f7a66;
  line-height: 1;
}

/* 按下反馈（:hover-class） */
.dir-btn-press {
  background-color: #8f7a66;
}

.dir-btn-press .dir-text {
  color: #ffffff;
}

/* 中心装饰圆 */
.dir-core {
  width: 84rpx;
  height: 84rpx;
  border-radius: 50%;
  background-color: #edc22e;
  display: flex;
  align-items: center;
  justify-content: center;
}

.dir-core-text {
  font-size: 22rpx;
  font-weight: 800;
  color: #6b5615;
}

/* ---------- 底部提示 ---------- */
.hint {
  margin-top: 28rpx;
}

.hint-text {
  font-size: 24rpx;
  color: #a09585;
}
</style>
