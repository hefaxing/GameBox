<template>
  <view class="page">
    <!-- 顶部工具栏：返回 + 步数/最佳 + 重置 -->
    <view class="top-bar">
      <view class="bar-btn back-btn" @click="goBack">
        <text class="bar-btn-text">← 返回</text>
      </view>
      <view class="stat-box">
        <text class="stat-label">步数</text>
        <text class="stat-value">{{ steps }}</text>
      </view>
      <view class="stat-box">
        <text class="stat-label">最少</text>
        <text class="stat-value">{{ best > 0 ? best : '--' }}</text>
      </view>
      <view class="bar-btn reset-btn" @click="resetGame">
        <text class="bar-btn-text">重置</text>
      </view>
    </view>

    <!-- 胜利提示 -->
    <view v-if="won" class="status-bar">
      <text class="status-text">🎉 通关成功，共用 {{ steps }} 步</text>
    </view>

    <!-- 4x4 棋盘（点击与空格相邻的数字块即可滑动） -->
    <view class="board">
      <view
        v-for="(n, i) in board"
        :key="i"
        class="tile"
        :style="tileStyle(n)"
        @click="onTap(i)"
      >
        <text v-if="n > 0" class="tile-text">{{ n }}</text>
      </view>
    </view>

    <view class="hint">
      <text class="hint-text">点击与空格相邻的数字块滑动，把数字排成 1–15 即通关</text>
    </view>
  </view>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { onLoad } from '@dcloudio/uni-app'

/** 棋盘边长（4x4） */
const SIZE = 4
/** 方块总数（含 1 个空格） */
const COUNT = SIZE * SIZE
/** 打乱时执行的随机合法移动次数 */
const SHUFFLE_MOVES = 150
/** "最少步数"记录的本地存储 key（uni 原生 API） */
const BEST_KEY = 'SLIDING_BEST_STEPS'

/** 生成已通关的棋盘：1–15 顺序排列，空格在右下角（索引 15） */
function createSolvedBoard(): number[] {
  const b: number[] = []
  for (let i = 1; i < COUNT; i++) b.push(i)
  b.push(0)
  return b
}

/** 判断棋盘是否已通关（1–15 顺序 + 空格在最后） */
function isSolvedBoard(b: number[]): boolean {
  for (let i = 0; i < COUNT; i++) {
    const expect = i + 1 === COUNT ? 0 : i + 1
    if (b[i] !== expect) return false
  }
  return true
}

/** 返回某位置所有上下左右相邻位置 */
function neighborsOf(index: number): number[] {
  const r = Math.floor(index / SIZE)
  const c = index % SIZE
  const out: number[] = []
  if (r > 0) out.push(index - SIZE)
  if (r < SIZE - 1) out.push(index + SIZE)
  if (c > 0) out.push(index - 1)
  if (c < SIZE - 1) out.push(index + 1)
  return out
}

/**
 * 执行一次随机合法移动：随机选择可滑入空格的方块。
 * 传入上一次空格位置，禁止立刻撤回到上一步（避免原地打转），
 * 返回移动前的空格位置（供下一次移动作为"上次位置"）。
 */
function randomMove(b: number[], lastEmpty: number): number {
  const empty = b.indexOf(0)
  const candidates = neighborsOf(empty).filter((i) => i !== lastEmpty)
  const pool = candidates.length > 0 ? candidates : neighborsOf(empty)
  const pick = pool[Math.floor(Math.random() * pool.length)]
  ;[b[empty], b[pick]] = [b[pick], b[empty]]
  return empty
}

/** 打乱：从已通关状态执行 N 次随机合法移动，结果必然可解 */
function shuffleBoard(): number[] {
  const b = createSolvedBoard()
  let lastEmpty = -1
  for (let i = 0; i < SHUFFLE_MOVES; i++) {
    lastEmpty = randomMove(b, lastEmpty)
  }
  // 极小概率打乱后又回到通关状态，则再补一步
  if (isSolvedBoard(b)) randomMove(b, lastEmpty)
  return b
}

/** 响应式棋盘（扁平数组，索引 = 行*SIZE+列，0 为空格） */
const board = ref<number[]>(createSolvedBoard())
/** 当前步数 */
const steps = ref(0)
/** 历史最少步数记录 */
const best = ref(0)
/** 是否已通关 */
const won = ref(false)

/** 读取本地存储中的"最少步数"记录 */
function loadBest(): number {
  try {
    const v = uni.getStorageSync(BEST_KEY)
    return typeof v === 'number' && v > 0 ? v : 0
  } catch {
    return 0
  }
}

/** 写入"最少步数"记录 */
function saveBest(v: number): void {
  try {
    uni.setStorageSync(BEST_KEY, v)
  } catch (e) {
    console.warn('写入最少步数记录失败', e)
  }
}

/** 数字块背景色：按数值从冷到暖渐变，空格透明 */
function tileStyle(n: number): Record<string, string> {
  if (n === 0) return { backgroundColor: 'transparent' }
  return { backgroundColor: `hsl(${n * 22}, 60%, 64%)` }
}

/** 点击数字块：仅当与空格相邻时滑入空格 */
function onTap(index: number): void {
  if (won.value) return
  if (board.value[index] === 0) return
  const emptyIdx = board.value.indexOf(0)
  if (neighborsOf(emptyIdx).indexOf(index) === -1) return
  const b = board.value.slice()
  ;[b[emptyIdx], b[index]] = [b[index], b[emptyIdx]]
  board.value = b
  steps.value += 1
  if (isSolvedBoard(b)) {
    won.value = true
    if (best.value === 0 || steps.value < best.value) {
      best.value = steps.value
      saveBest(steps.value)
      uni.showToast({ title: `🎉 通关！新建最少步数 ${steps.value} 步`, icon: 'none' })
    }
  }
}

/** 开始新的一局（打乱 + 步数清零 + 清除胜利状态） */
function newGame(): void {
  board.value = shuffleBoard()
  steps.value = 0
  won.value = false
}

/** 重置：重新打乱、步数清零 */
function resetGame(): void {
  newGame()
  uni.showToast({ title: '已重新打乱，开冲！', icon: 'none' })
}

/** 返回上一页；若页面栈已到底（无法回退）则 reLaunch 到首页 */
function goBack(): void {
  uni.navigateBack({
    fail: () => {
      uni.reLaunch({ url: '/pages/index/index' })
    },
  })
}

onLoad(() => {
  best.value = loadBest()
  newGame()
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
  padding: 16rpx 26rpx;
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
  min-width: 120rpx;
}

.stat-label {
  font-size: 20rpx;
  color: #6b7f9c;
}

.stat-value {
  font-size: 34rpx;
  font-weight: 700;
  color: #2c4a7a;
}

/* ---------- 胜利提示 ---------- */
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
  width: 660rpx;
  margin-top: 16rpx;
  background-color: #d8dce4;
  border-radius: 18rpx;
  padding: 14rpx;
  box-sizing: border-box;
  display: flex;
  flex-wrap: wrap;
  gap: 10rpx;
}

.tile {
  width: 150rpx;
  height: 150rpx;
  border-radius: 12rpx;
  display: flex;
  align-items: center;
  justify-content: center;
}

.tile-text {
  font-size: 52rpx;
  font-weight: 700;
  color: #2c3e50;
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
