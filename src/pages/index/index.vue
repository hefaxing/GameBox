<template>
  <view class="page">
    <view class="header">
      <view class="header-title-row">
        <view class="header-accent"></view>
        <text class="header-title">游戏大厅</text>
      </view>
      <text class="header-sub">选择一款小游戏开始</text>
    </view>

    <view class="grid">
      <view
        v-for="game in games"
        :key="game.id"
        class="card"
        @click="gotoGame(game)"
      >
        <view class="card-icon" :style="{ backgroundColor: game.color }">
          <text class="card-icon-text">{{ game.iconLetter }}</text>
        </view>
        <view class="card-body">
          <text class="card-title">{{ game.title }}</text>
          <text class="card-desc">{{ game.desc }}</text>
        </view>
      </view>
    </view>

    <view class="empty" v-if="games.length === 0">
      <text class="empty-text">游戏正在上架中，敬请期待</text>
    </view>
  </view>
</template>

<script setup lang="ts">
import { ref } from 'vue'

/** 单个游戏的描述信息 */
interface GameItem {
  id: string
  title: string
  desc: string
  /** 跳转目标路径（分包页面） */
  url: string
  /** 卡片图标底色 */
  color: string
  /** 卡片图标上的字母 */
  iconLetter: string
}

/**
 * 游戏列表数据源（两列网格，6 款正好 3 行）。
 * 以后新增游戏，只需往数组里追加一项即可，页面自动渲染。
 */
const games = ref<GameItem[]>([
  {
    id: '2048',
    title: '2048',
    desc: '滑动合并，冲刺2048',
    url: '/games/2048/index',
    color: '#faf0be',
    iconLetter: '2',
  },
  {
    id: 'sliding',
    title: '数字华容道',
    desc: '滑动方块，排齐数字',
    url: '/games/sliding/index',
    color: '#f5a623',
    iconLetter: '15',
  },
  {
    id: 'gomoku',
    title: '五子棋',
    desc: '横竖斜，先成五连',
    url: '/games/gomoku/index',
    color: '#e8e8e8',
    iconLetter: '●○',
  },
  {
    id: 'mines',
    title: '扫雷',
    desc: '点按翻格，长按插旗',
    url: '/games/mines/index',
    color: '#2c3e50',
    iconLetter: '💣',
  },
  {
    id: 'sudoku',
    title: '数独',
    desc: '推理填数，行列不重',
    url: '/games/sudoku/index',
    color: '#7fb069',
    iconLetter: '9',
  },
  {
    id: 'sokoban',
    title: '推箱子',
    desc: '推动箱子，全部就位',
    url: '/games/sokoban/index',
    color: '#d9822b',
    iconLetter: '📦',
  },
])

/** 跳转到指定游戏页面 */
function gotoGame(game: GameItem): void {
  uni.navigateTo({
    url: game.url,
    fail: (err) => {
      console.error('navigateTo fail', err)
      uni.showToast({ title: '打开游戏失败', icon: 'none' })
    },
  })
}
</script>

<style scoped>
.page {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  /* 极浅的纵向渐变：顶部稍带蓝调，向下过渡到原底色，H5 与微信小程序均支持 */
  background: linear-gradient(180deg, #eef2f9 0%, #f6f7f9 42%, #f6f7f9 100%);
  box-sizing: border-box;
}

/* ---------- 头部区 ---------- */
.header {
  display: flex;
  flex-direction: column;
  padding: 64rpx 48rpx 16rpx;
}

.header-title-row {
  display: flex;
  flex-direction: row;
  align-items: center;
}

/* 标题左侧的渐变色条：纯 CSS 装饰，无图片资源 */
.header-accent {
  width: 8rpx;
  height: 44rpx;
  margin-right: 16rpx;
  border-radius: 4rpx;
  background: linear-gradient(180deg, #4f8cff 0%, #7f6bff 100%);
}

.header-title {
  font-size: 48rpx;
  font-weight: 700;
  color: #222;
  letter-spacing: 2rpx;
}

.header-sub {
  margin: 14rpx 0 0 24rpx;
  font-size: 26rpx;
  color: #888;
  letter-spacing: 1rpx;
}

/* ---------- 两列网格 ----------
 * 左右留白 = grid padding 48rpx + 卡片左/右 margin 12rpx = 60rpx（明显大于原先约 32rpx）。
 */
.grid {
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  /* 左右 48rpx 加大边距；上下留白承担首尾节奏 */
  padding: 28rpx 48rpx 40rpx;
  align-content: flex-start;
}

/*
 * 宽度算术（保证一行两卡精确铺满、两侧对称）：
 *   注意：flex 子项的百分比相对「内容盒」解析（已扣除 grid 左右 padding）：
 *   卡片宽度     = 50% - 24rpx
 *   两张卡+边距  = 2 × (50% - 24rpx) + 4 × 12rpx = 100%   ✓ 恰好铺满内容盒
 *   卡间间隙 = 12rpx × 2 = 24rpx；两侧留白 = grid 48 + 卡 12 = 60rpx；行距 24rpx。
 *   （若按整屏 100% 推算会多算 96rpx，导致余量堆到右侧、两侧不对称。）
 */
.card {
  width: calc(50% - 24rpx);
  margin: 0 12rpx 24rpx;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  align-items: center;
  background-color: #fff;
  border-radius: 28rpx;
  padding: 30rpx 24rpx 28rpx;
  box-shadow: 0 10rpx 24rpx rgba(31, 45, 70, 0.08);
  transition: transform 0.15s ease, box-shadow 0.15s ease, background-color 0.15s ease;
}

/* 轻量的按压反馈：:active 在 H5 与微信小程序均生效，无需任何库 */
.card:active {
  transform: scale(0.96);
  background-color: #f4f6fa;
  box-shadow: 0 4rpx 12rpx rgba(31, 45, 70, 0.12);
}

/* 上方图标区：竖版卡片的视觉主体，加大一档更醒目 */
.card-icon {
  width: 144rpx;
  height: 144rpx;
  margin: 6rpx auto 28rpx;
  border-radius: 32rpx;
  display: flex;
  align-items: center;
  justify-content: center;
}

.card-icon-text {
  font-size: 64rpx;
  font-weight: 700;
  color: #6b5b2e;
  line-height: 1.1;
}

/* 下方文案区：标题/描述层次更清晰 */
.card-body {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.card-title {
  font-size: 34rpx;
  font-weight: 600;
  color: #1f2430;
  text-align: center;
  letter-spacing: 2rpx;
}

.card-desc {
  margin-top: 12rpx;
  font-size: 24rpx;
  line-height: 1.5;
  color: #8a90a0;
  text-align: center;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 100%;
}

/* ---------- 空列表兜底 ---------- */
.empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding-top: 220rpx;
}

.empty-text {
  font-size: 28rpx;
  color: #aaa;
  letter-spacing: 1rpx;
}
</style>
