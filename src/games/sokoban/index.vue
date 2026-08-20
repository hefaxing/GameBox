<template>
  <view class="page">
    <!-- 顶部工具栏：返回大厅 + 撤销 + 重置游戏 -->
    <view class="top-bar">
      <view class="bar-btn back-btn" @click="goBack">
        <text class="bar-btn-text">← 返回大厅</text>
      </view>
      <view class="bar-btn back-btn" @click="onUndo">
        <text class="bar-btn-text">撤销</text>
      </view>
      <view class="bar-btn reset-btn" @click="resetGame">
        <text class="bar-btn-text">重置游戏</text>
      </view>
    </view>

    <!-- 状态条：胜利 / 当前关卡最佳提示 -->
    <view class="status-bar">
      <text v-if="won" class="status-text win">🎉 第 {{ level + 1 }} 关通关，共 {{ steps }} 步</text>
      <text v-else class="status-text plain">
        第 {{ level + 1 }} / {{ LEVELS.length }} 关 · 本关最佳：{{ bests[level] > 0 ? bests[level] + ' 步' : '--' }}
      </text>
    </view>

    <!-- 关卡统计 -->
    <view class="stat-row">
      <view class="stat-box">
        <text class="stat-label">关卡</text>
        <text class="stat-value">{{ level + 1 }}/{{ LEVELS.length }}</text>
      </view>
      <view class="stat-box">
        <text class="stat-label">当前步数</text>
        <text class="stat-value">{{ steps }}</text>
      </view>
      <view class="stat-box">
        <text class="stat-label">本关最佳</text>
        <text class="stat-value">{{ bests[level] > 0 ? bests[level] : '--' }}</text>
      </view>
    </view>

    <!-- 棋盘：行列网格，墙/目标点/箱子/玩家区分配色 -->
    <view class="board">
      <view v-for="r in rows" :key="'r' + (r - 1)" class="board-row">
        <view
          v-for="c in cols"
          :key="'c' + (c - 1)"
          class="cell"
          :class="cellClass(r - 1, c - 1)"
          :style="cellSizeStyle"
        >
          <view v-if="hasBox(r - 1, c - 1)" class="box" :class="{ 'box-done': isGoal(r - 1, c - 1) }"></view>
          <view v-else-if="isPlayer(r - 1, c - 1)" class="player"></view>
          <view v-else-if="isGoal(r - 1, c - 1)" class="goal-dot"></view>
        </view>
      </view>
    </view>

    <view class="hint">
      <text class="hint-text">推箱子到所有目标点上即过关；撞上墙或箱子叠箱则不动</text>
    </view>

    <!-- 底部方向键：上 / 左下右 十字布局 -->
    <view class="pad">
      <view class="pad-row">
        <view class="pad-none"></view>
        <view class="pad-btn" @click="onMove(-1, 0)">
          <text class="pad-btn-text">↑</text>
        </view>
        <view class="pad-none"></view>
      </view>
      <view class="pad-row">
        <view class="pad-btn" @click="onMove(0, -1)">
          <text class="pad-btn-text">←</text>
        </view>
        <view class="pad-btn" @click="onMove(1, 0)">
          <text class="pad-btn-text">↓</text>
        </view>
        <view class="pad-btn" @click="onMove(0, 1)">
          <text class="pad-btn-text">→</text>
        </view>
      </view>
    </view>

    <!-- 进入下一关（最后一关时回第一关） -->
    <view class="next-row">
      <view class="bar-btn next-btn" @click="nextLevel">
        <text class="bar-btn-text next-btn-text">{{ level + 1 < LEVELS.length ? '下一关 →' : '🔄 再玩第一关' }}</text>
      </view>
    </view>
  </view>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { onLoad } from '@dcloudio/uni-app'

/** 网格坐标 */
interface Pos {
  r: number
  c: number
}

/** 一次成功移动的快照（撤销用） */
interface Snapshot {
  p: Pos
  boxes: string[]
}

/**
 * 内置 3 个小型关卡（已人工验证可解）。
 * 图例：# 墙，@ 玩家，$ 箱子，. 目标点，* 箱子在目标点，+ 人在目标点，空格地面
 */
const LEVELS: string[][] = [
  // 第 1 关：5x6，1 个箱子，先左推再下推
  [
    '#####',
    '#   #',
    '#@  #',
    '# $ #',
    '#.  #',
    '#####',
  ],
  // 第 2 关：5x6，1 个箱子，绕到右侧左推再上绕下推
  [
    '#####',
    '#   #',
    '# $ #',
    '#.  #',
    '#@  #',
    '#####',
  ],
  // 第 3 关：7x7，2 个箱子，上下包抄分别下推
  [
    '#######',
    '#     #',
    '# $ $ #',
    '# .@. #',
    '#     #',
    '#     #',
    '#######',
  ],
]

/** 每关最佳步数存储 key 前缀（形如 SOKOBAN_BEST_1） */
const BEST_PREFIX = 'SOKOBAN_BEST_'

/** 当前关卡下标（0 起） */
const level = ref(0)
/** 当前玩家坐标 */
const player = ref<Pos>({ r: 0, c: 0 })
/** 墙格集合（key: "r,c"） */
const walls = ref<Set<string>>(new Set<string>())
/** 目标点集合 */
const goals = ref<Set<string>>(new Set<string>())
/** 箱子位置集合 */
const boxes = ref<Set<string>>(new Set<string>())
/** 当前步数（仅成功移动 +1） */
const steps = ref(0)
/** 是否已过本关（过关后停止操作） */
const won = ref(false)
/** 每关最佳步数，0 表示无记录 */
const bests = ref<number[]>([0, 0, 0])
/** 移动历史栈（撤销用） */
let history: Snapshot[] = []

/** 关卡行数 */
const rows = computed(() => LEVELS[level.value].length)
/** 关卡列数 */
const cols = computed(() => LEVELS[level.value][0].length)

/** 格子固定边长（rpx），随列数变化 */
const cellSizeStyle = computed(() => {
  const side = Math.floor(708 / cols.value)
  return { width: side + 'rpx', height: side + 'rpx' }
})

/** 坐标 → 存储 key */
function key(r: number, c: number): string {
  return r + ',' + c
}

/** 该格是否为墙 */
function isWall(r: number, c: number): boolean {
  return walls.value.has(key(r, c))
}

/** 该格是否为目标点 */
function isGoal(r: number, c: number): boolean {
  return goals.value.has(key(r, c))
}

/** 该格是否有箱子 */
function hasBox(r: number, c: number): boolean {
  return boxes.value.has(key(r, c))
}

/** 该格是否为玩家 */
function isPlayer(r: number, c: number): boolean {
  return player.value.r === r && player.value.c === c
}

/** 棋盘单元格 class：墙 / 目标点 / 地面 */
function cellClass(r: number, c: number): string {
  if (isWall(r, c)) return 'cell-wall'
  return isGoal(r, c) ? 'cell-goal' : 'cell-floor'
}

/** 解析并加载指定关卡的初始布局 */
function loadLevel(i: number): void {
  level.value = i
  steps.value = 0
  won.value = false
  history = []

  const map = LEVELS[i]
  const w = new Set<string>()
  const g = new Set<string>()
  const b = new Set<string>()
  const p: Pos = { r: 0, c: 0 }
  for (let r = 0; r < map.length; r++) {
    for (let c = 0; c < map[r].length; c++) {
      const ch = map[r][c]
      const k = key(r, c)
      if (ch === '#') w.add(k)
      if (ch === '.' || ch === '*' || ch === '+') g.add(k)
      if (ch === '$' || ch === '*') b.add(k)
      if (ch === '@' || ch === '+') {
        p.r = r
        p.c = c
      }
    }
  }
  player.value = p
  walls.value = w
  goals.value = g
  boxes.value = b
}

/** 读取某关最佳步数记录 */
function loadBest(i: number): number {
  try {
    const v = uni.getStorageSync(BEST_PREFIX + (i + 1))
    return typeof v === 'number' && v > 0 ? v : 0
  } catch {
    return 0
  }
}

/** 写入某关最佳步数记录 */
function saveBest(i: number, v: number): void {
  try {
    uni.setStorageSync(BEST_PREFIX + (i + 1), v)
  } catch (e) {
    console.warn('写入推箱子最佳步数失败', e)
  }
}

/** 方向移动：dr/dc 为行列偏移；撞墙不动，推箱子受前方阻挡规则约束 */
function onMove(dr: number, dc: number): void {
  if (won.value) return
  const p = player.value
  const nr = p.r + dr
  const nc = p.c + dc
  const nk = key(nr, nc)
  if (isWall(nr, nc)) return

  // 目标是箱子：箱子前方是墙或另一箱子则不动，否则人与箱子同向各移一格
  if (hasBox(nr, nc)) {
    const br = nr + dr
    const bc = nc + dc
    const bk = key(br, bc)
    if (isWall(br, bc) || hasBox(br, bc)) return
    history.push({ p: { r: p.r, c: p.c }, boxes: Array.from(boxes.value) })
    boxes.value.delete(nk)
    boxes.value.add(bk)
    player.value = { r: nr, c: nc }
    steps.value = history.length
    checkWin()
    return
  }

  // 普通移动
  history.push({ p: { r: p.r, c: p.c }, boxes: Array.from(boxes.value) })
  player.value = { r: nr, c: nc }
  steps.value = history.length
}

/** 撤销一步：回退到上一次移动前的快照 */
function onUndo(): void {
  if (won.value) return
  const last = history.pop()
  if (!last) {
    uni.showToast({ title: '没有可撤销的移动', icon: 'none' })
    return
  }
  player.value = { r: last.p.r, c: last.p.c }
  boxes.value = new Set(last.boxes)
  steps.value = history.length
}

/** 胜利判定：全部箱子落在目标点上 */
function checkWin(): void {
  let allOnGoal = true
  boxes.value.forEach((k) => {
    if (!goals.value.has(k)) allOnGoal = false
  })
  if (!allOnGoal) return
  won.value = true
  const i = level.value
  const isNewBest = bests.value[i] === 0 || steps.value < bests.value[i]
  if (isNewBest) {
    bests.value[i] = steps.value
    saveBest(i, steps.value)
  }
  uni.showToast({
    title: isNewBest ? `🎉 第 ${i + 1} 关通关！新最佳 ${steps.value} 步` : `🎉 第 ${i + 1} 关通关！共 ${steps.value} 步`,
    icon: 'none',
  })
}

/** 进入下一关（最后一关之后回到第 1 关） */
function nextLevel(): void {
  const i = (level.value + 1) % LEVELS.length
  loadLevel(i)
  uni.showToast({ title: '第 ' + (i + 1) + ' 关开始', icon: 'none' })
}

/** 重置游戏：当前关回到初始布局、步数清零、清除胜利状态 */
function resetGame(): void {
  loadLevel(level.value)
  uni.showToast({ title: '已重置本关', icon: 'none' })
}

/** 返回大厅；若页面栈已到底（无法回退）则 reLaunch 到首页 */
function goBack(): void {
  uni.navigateBack({
    fail: () => {
      uni.reLaunch({ url: '/pages/index/index' })
    },
  })
}

onLoad(() => {
  // 初始化第 1 关，并读取各关最佳步数
  for (let i = 0; i < LEVELS.length; i++) {
    bests.value[i] = loadBest(i)
  }
  loadLevel(0)
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
  padding-bottom: 40rpx;
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

.back-btn .bar-btn-text {
  font-size: 28rpx;
  color: #555;
}

.reset-btn {
  background-color: #3f68a8;
}

.reset-btn .bar-btn-text {
  font-size: 28rpx;
  color: #fff;
  font-weight: 600;
}

/* ---------- 状态条 ---------- */
.status-bar {
  width: 100%;
  display: flex;
  justify-content: center;
  padding: 0 40rpx 16rpx;
  box-sizing: border-box;
  min-height: 64rpx;
}

.status-text {
  display: inline-block;
  font-size: 26rpx;
  border-radius: 10rpx;
  padding: 8rpx 20rpx;
}

.status-text.plain {
  color: #6b7f9c;
  background-color: #eef3fa;
}

.status-text.win {
  color: #4a7b50;
  background-color: #e5f3e8;
}

/* ---------- 关卡统计 ---------- */
.stat-row {
  width: 100%;
  display: flex;
  justify-content: center;
  gap: 24rpx;
  padding: 0 40rpx 8rpx;
  box-sizing: border-box;
}

.stat-box {
  background-color: #eef3fa;
  border-radius: 14rpx;
  padding: 10rpx 32rpx;
  display: flex;
  flex-direction: column;
  align-items: center;
  min-width: 150rpx;
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

/* ---------- 棋盘 ---------- */
.board {
  width: 720rpx;
  margin-top: 16rpx;
  background-color: #8a93a2;
  border-radius: 12rpx;
  padding: 6rpx;
  box-sizing: border-box;
  border: 6rpx solid #8a93a2;
  display: flex;
  flex-direction: column;
}

.board-row {
  display: flex;
}

.cell {
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: center;
}

.cell-wall {
  background-color: #5f6b7e;
}

.cell-floor {
  background-color: #fbfcfd;
  border-right: 1rpx solid #e0e4ea;
  border-bottom: 1rpx solid #e0e4ea;
}

.cell-goal {
  background-color: #f2f6ee;
  border-right: 1rpx solid #e0e4ea;
  border-bottom: 1rpx solid #e0e4ea;
}

/* 箱子 */
.box {
  width: 78%;
  height: 78%;
  border-radius: 12rpx;
  background-color: #d9822b;
  box-shadow: inset 0 -8rpx 0 rgba(0, 0, 0, 0.18);
}

/* 箱子已在目标点上 */
.box-done {
  background-color: #7fb069;
  box-shadow: inset 0 -8rpx 0 rgba(0, 0, 0, 0.15);
}

/* 玩家 */
.player {
  width: 78%;
  height: 78%;
  border-radius: 50%;
  background-color: #3f68a8;
  box-shadow: inset 0 -8rpx 0 rgba(0, 0, 0, 0.2);
}

/* 空的目标点标记 */
.goal-dot {
  width: 32rpx;
  height: 32rpx;
  border-radius: 50%;
  background-color: #7fb069;
  opacity: 0.55;
}

/* ---------- 提示 ---------- */
.hint {
  margin-top: 20rpx;
  padding: 0 60rpx;
}

.hint-text {
  font-size: 24rpx;
  color: #a0a6af;
}

/* ---------- 底部方向键 ---------- */
.pad {
  margin-top: 24rpx;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16rpx;
}

.pad-row {
  display: flex;
  gap: 16rpx;
}

.pad-none {
  width: 120rpx;
  height: 104rpx;
}

.pad-btn {
  width: 120rpx;
  height: 104rpx;
  border-radius: 16rpx;
  background-color: #fff;
  border: 1rpx solid #d5d8dc;
  display: flex;
  align-items: center;
  justify-content: center;
}

.pad-btn-text {
  font-size: 48rpx;
  font-weight: 700;
  color: #2c4a7a;
}

/* ---------- 下一关按钮 ---------- */
.next-row {
  margin-top: 28rpx;
}

.next-btn {
  background-color: #fff;
  border: 1rpx dashed #3f68a8;
  padding: 18rpx 48rpx;
}

.next-btn-text {
  font-size: 28rpx;
  color: #3f68a8;
  font-weight: 600;
}
</style>
