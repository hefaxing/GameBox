# SOUL.md｜GameBox 小游戏合集
## 项目基础信息
项目路径：`./`
技术栈：Uni-App(Vite) + Vue3 + TypeScript，`<script setup lang="ts">`
目标多端：H5网页、微信小程序、Android离线SDK打包；**全部逻辑前端本地运行，无后端接口**。

项目标识：
- GitHub仓库名：GameBox
- 本地项目目录：GameBox
- App名称：Game Box
- H5标题：Game Box — Casual Puzzle Mini Games
- 小程序名称：游戏盒子
- Android packageName：com.game.gamebox

目录约定：
1. 主包首页：`src/pages/index/index.vue` 游戏大厅，展示全部游戏入口卡片。
2. 游戏统一放在分包：`src/games/[游戏名]/index.vue`，每个游戏独立文件夹。
3. `pages.json` games分包注册每一个游戏页面。
4. 禁止修改项目基础配置之外的无关文件。

## 硬性编码规则
1. 不引入第三方npm库，全部使用uni-app原生组件与API。
2. 兼容H5浏览器触摸、微信小程序真机touch事件；优先uni事件，不要浏览器原生event。
3. 本地存储统一使用 `uni.setStorageSync` / `uni.getStorageSync`，保存关卡记录、最佳成绩。
4. 每个游戏页面必须具备：【返回大厅】按钮，跳转回首页；游戏重置按钮。
5. 分包页面必须在 `pages.json` 的subPackages里面注册，否则页面打不开。
6. 样式使用rpx单位，适配手机屏幕。

## 🚨 OpenClaw工具调用强制规则
1. **完整源代码直接写入磁盘文件，聊天对话窗口禁止输出大段完整源码，只输出变更文件清单+简短diff摘要。**
2. 一次任务只做**一个游戏/一个小修改模块**，禁止同时开发多个游戏，任务完成就结束本轮，不要主动扩展任务范围。
3. 使用 `file.write` / `file.patch` 修改代码；不要把千行代码打印输出到聊天会话，避免token暴涨。
4. 如果输出发生截断，直接停止当前任务，不需要使用“继续”指令；等待用户重新下发指令接续。
5. 禁止并行多任务，串行执行。
6. 修改完成后输出简短报告：变更文件列表、关键改动点、注意事项。

## 开发流程约束
1. 新增一个游戏：
- 创建 `src/games/xxx/index.vue` 游戏主体页面
- 修改首页 `src/pages/index/index.vue`，追加游戏入口卡片
- 修改 `pages.json` subPackages注册该游戏分包页面
2. 迭代旧游戏：只修改对应游戏vue，不改动其他游戏。
3. 不要操作dist、node_modules、unpackage编译产物目录。

## 禁止行为
1. 不要引入网络请求、后端接口逻辑，本项目纯本地离线小游戏。
2. 不要使用history路由，app/小程序端默认hash路由。

## 用户本地调试命令
```bash
# H5调试
npm run dev:h5
# 微信小程序编译
npm run dev:mp-weixin
# H5生产打包
npm run build:h5
# 小程序生产打包
npm run build:mp-weixin

