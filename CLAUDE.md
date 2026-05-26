# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此代码库中工作时提供指导。

## 项目概述

LX Music Desktop（洛雪音乐助手）是一个基于 Electron + Vue 3 开发的桌面音乐应用，支持 Windows、macOS 和 Linux。主要功能包括音乐搜索、播放列表管理、桌面歌词以及用户可扩展的 API 脚本。

**技术栈**: Electron 37.6.0、Vue 3.3.13、TypeScript 5.9.2、Webpack 5、better-sqlite3、Pug 模板、Less 样式表

## 常用命令

### 开发
```bash
npm run dev          # 启动开发模式（4个webpack进程 + Electron）
npm run lint         # 运行 ESLint
npm run lint:fix     # 自动修复 ESLint 问题
```

### 构建
```bash
npm run build               # 构建所有进程（主进程、渲染进程、歌词窗口、脚本）
npm run build:main          # 仅构建主进程
npm run build:renderer      # 仅构建渲染进程
npm run build:renderer-lyric # 仅构建歌词窗口
npm run build:theme         # 重新生成主题 CSS 文件
```

### 打包
```bash
npm run pack             # 打包 Windows x64 安装包
npm run pack:win         # 打包所有 Windows 版本
npm run pack:linux       # 打包所有 Linux 版本
npm run pack:mac         # 打包 macOS dmg（x64 + arm64）
```

## 核心架构模式

### 多进程架构

这是一个 Electron 应用，包含**三个独立的渲染进程**，每个都有独立的 webpack 配置：

1. **主进程** (`src/main/`)
   - 入口：`src/main/index.ts`
   - 输出：`dist/main.js`
   - Webpack：`build-config/main/webpack.config.{base,dev,prod}.js`
   - 管理所有窗口、worker、数据库、IPC 协调

2. **渲染进程 - 主窗口** (`src/renderer/`)
   - 入口：`src/renderer/main.ts`
   - 输出：`dist/index.html`、`dist/renderer.js`
   - Webpack：`build-config/renderer/webpack.config.{base,dev,prod}.js`
   - 开发服务器：端口 9080
   - 路由：`/search`、`/songList/list`、`/songList/detail`、`/leaderboard`、`/list`、`/download`、`/setting`

3. **渲染进程 - 歌词窗口** (`src/renderer-lyric/`)
   - 入口：`src/renderer-lyric/main.ts`
   - 输出：`dist/lyric.html`、`dist/renderer-lyric.js`
   - Webpack：`build-config/renderer-lyric/webpack.config.{base,dev,prod}.js`
   - 开发服务器：端口 9081
   - 独立的桌面歌词悬浮窗口

4. **用户 API 预加载脚本** (`src/main/modules/userApi/renderer/`)
   - 入口：`src/main/modules/userApi/renderer/preload.js`
   - 输出：`dist/user-api-preload.js`
   - Webpack：`build-config/renderer-scripts/webpack.config.{base,dev,prod}.js`
   - 用户提供的 API 扩展的沙箱执行环境

**修改构建配置时**，可能需要更新多个 webpack 配置文件。

### 路径别名（所有进程通用）

```typescript
'@main'      → 'src/main'
'@renderer'  → 'src/renderer'
'@lyric'     → 'src/renderer-lyric'
'@common'    → 'src/common'
'@static'    → 'src/static'
'@root'      → 'src'
```

### 状态管理：不使用 Vuex/Pinia

本项目**直接使用 Vue 3 组合式 API 原语**，而不是 Vuex 或 Pinia。

**模式** (`src/renderer/store/`)：
```typescript
// src/renderer/store/example.ts
import { reactive, ref, computed } from 'vue'

export const state = reactive({
  count: 0
})

export const increment = () => {
  state.count++
}

// 在组件中使用
import { state, increment } from '@renderer/store/example'
```

状态模块：`index.ts`（全局应用状态）、`setting.ts`、`player/`、`list/`、`download/`

### IPC 通信

**集中式事件注册表**：`src/common/ipcNames.ts`

永远不要硬编码 IPC 事件名称。始终使用以下常量：
- `CMMON_EVENT_NAME` - 全局应用事件
- `PLAYER_EVENT_NAME` - 音乐播放器控制
- `WIN_MAIN_RENDERER_EVENT_NAME` - 主窗口 IPC
- `WIN_LYRIC_RENDERER_EVENT_NAME` - 歌词窗口 IPC
- `HOTKEY_RENDERER_EVENT_NAME` - 快捷键事件

**类型安全的包装器**：
- 主进程：`mainOn`、`mainHandle`、`mainSend` 来自 `@common/mainIpc`
- 渲染进程：`rendererSend`、`rendererInvoke`、`rendererOn` 来自 `@common/rendererIpc`

### Worker 线程架构

**主进程**使用 Node.js `worker_threads` + Comlink：
```typescript
global.lx.worker.dbService  // 数据库操作 worker
```

**渲染进程**使用 Web Workers + Comlink：
```typescript
window.lx.worker.main       // 主 worker
window.lx.worker.download   // 下载 worker
```

**关键提示**：所有数据库操作都通过 worker 线程的 Comlink RPC 进行。永远不要从主线程直接访问数据库。

数据库 worker 模块：`list`、`lyric`、`music_url`、`music_other_source`、`download`、`dislike_list`

类型定义：`src/common/types/main_window.d.ts` → `workerDBSeriveTypes`

### 数据库架构

由 worker 线程中的 better-sqlite3 管理的 **SQLite WAL 模式**数据库。

位置：`{userData}/LxDatas/lx.data.db`

数据表：
- `db_info` - 版本跟踪
- `list` - 用户播放列表
- `list_music` - 播放列表中的歌曲
- `lyric` - 缓存的歌词
- `music_url` - 缓存的音乐 URL
- `music_other_source` - 自定义音乐源
- `download` - 下载队列
- `dislike_list` - 不喜欢的歌曲

**数据库迁移**：`src/main/worker/dbService/migrate.ts`
**架构验证**：`src/main/worker/dbService/verifyDB.ts` - 启动时验证表结构并自动迁移

### 窗口间通信

主窗口 ↔ 歌词窗口通信使用 **MessageChannel**：

1. 歌词窗口向主进程请求通道
2. 主进程创建 MessageChannel
3. 通过 IPC 向每个窗口发送一个端口
4. 窗口通过端口直接通信（无需主进程参与）

模式：`src/main/modules/winLyric/index.ts` 和 `src/renderer-lyric/core/ipc.ts`

### 用户 API 扩展系统

位置：`src/main/modules/userApi/`

允许用户导入 JavaScript 脚本来扩展音乐源功能（获取 URL、歌词、专辑封面）。

**执行环境**：
- 带有预加载脚本的隔离 BrowserWindow（`dist/user-api-preload.js`）
- 通过 contextBridge 提供的沙箱 API：
  - `window.lx.request()` - HTTP 请求
  - `window.lx.utils.crypto.*` - 加密工具
  - `window.lx.utils.buffer.*` - Buffer 工具
  - `window.lx.utils.zlib.*` - 压缩工具

出于安全考虑，没有直接的 Node.js 访问权限。

### 全局状态对象

**主进程**：
```typescript
global.lx = {
  event_app,      // EventEmitter，用于应用生命周期、配置、主题变更
  event_list,     // EventEmitter，用于播放列表 CRUD
  event_dislike,  // EventEmitter，用于不喜欢列表
  worker: { dbService },  // Worker 线程句柄
  player_status,  // 当前播放器状态
  theme,          // 当前主题对象
  // ... 更多
}
```

**渲染进程**：
```typescript
window.lx = {
  worker: { main, download },  // Web Workers
  // 通过预加载脚本暴露
}
```

## 开发工作流

### 启动开发环境

```bash
npm run dev
```

这会启动：
1. 主进程 webpack（监视模式）
2. 渲染进程 webpack 开发服务器（端口 9080）
3. 歌词渲染进程 webpack 开发服务器（端口 9081）
4. 用户 API 脚本 webpack（监视模式）
5. Electron，带有 `--inspect=5858` 用于调试

### 热重载行为

- **渲染进程更改**：热模块替换（HMR）- 即时生效
- **歌词窗口更改**：HMR - 即时生效
- **主进程更改**：完全重启 Electron - 约 5 秒
- **通用代码更改**：可能需要手动重启

### 调试

- 主进程：Chrome DevTools 在 `chrome://inspect` 或 `localhost:5858`
- 渲染进程：内置 DevTools（Ctrl+Shift+I / Cmd+Option+I）
- Worker 线程：`console.log` 输出在相应进程的控制台

### 跨进程修改

添加跨多个进程的功能时：

1. 在 `src/common/ipcNames.ts` 中定义 IPC 事件
2. 在 `src/common/types/` 中添加类型定义
3. 在 `src/main/modules/` 中实现主进程处理器
4. 在 `src/renderer/` 中实现渲染进程逻辑
5. 如果需要数据库更改，更新 worker 方法

### 模块注册

主进程模块按特定顺序注册在 `src/main/modules/index.ts` 中：

```typescript
userApi         // 必须首先注册，用于 API 脚本初始化
commonRenderers // 共享工具
winMain         // 主窗口
hotKey          // 快捷键
tray            // 系统托盘
appMenu         // 应用菜单
winLyric        // 歌词窗口
// ... 更多
```

由于依赖关系，顺序很重要。

## 代码风格

### ESLint 配置

基于 Standard.js 和 TypeScript 扩展。

关键规则：
- `space-before-function-paren: never` - 函数括号前无空格
- `comma-dangle: always-multiline` - 多行时尾随逗号
- `eqeqeq: off` - 允许使用 `==`（谨慎使用）
- `camelcase: off` - 允许使用 snake_case
- `prefer-const: off` - 即使不重新赋值也允许使用 `let`

Vue 特定规则：
- `vue/multi-word-component-names: off` - 允许单词组件名
- 通过 `vue-pug` 插件支持 Pug 模板

### TypeScript

- 目标：ESNext
- 模块：ESNext，Node 解析
- 允许 JavaScript 文件（`allowJs: true`）
- 基础配置：`@tsconfig/recommended`

## 重要的非显而易见的模式

### 1. Windows 便携模式

在 Windows 上，如果可执行文件旁边存在 `portable` 文件夹，应用会使用 `{exeDir}/portable/userData` 而不是 `%APPDATA%/lx-music-desktop`。

检查位置：`src/main/utils/utils.ts` → `getUserDataPath()`

### 2. 单实例强制执行

使用 `app.requestSingleInstanceLock()`。第二个实例通过 `second-instance` 事件将其参数（如深度链接）传递给第一个实例。

位置：`src/main/app.ts`

### 3. 深度链接支持

协议：`lxmusic://`

通过 `app.setAsDefaultProtocolClient('lxmusic')` 注册。浏览器扩展使用此功能将歌曲发送到应用。

解析：`src/main/utils/schemeUrl.ts`

### 4. 主题系统

主题是由 `src/common/theme/createThemes.js` 生成的预构建 CSS 文件。

运行时切换：在 `<html>` 元素上更改主题类，无需重启。

自动主题：通过 `nativeTheme.shouldUseDarkColors` 跟随操作系统深色模式。

### 5. 导航安全

渲染进程通过 `will-navigate` 处理器锁定导航：
- 仅允许白名单 URL（参见 `@common/config.ts` → `ALLOWED_RENDERER_URLS`）
- 防止通过外部导航的 XSS

### 6. 自定义窗口标题栏

主窗口无边框（`frame: false`）。自定义标题栏在 Vue 中实现。

窗口控件：`src/renderer/components/layout/WindowControl.vue`

### 7. 歌词窗口点击穿透

使用 `win.setIgnoreMouseEvents(true)` 实现点击穿透模式。

切换：用户可以启用/禁用点击穿透和拖动。

### 8. 同步架构

两种模式：
- **服务器模式**：启动 WebSocket 服务器供其他设备连接
- **客户端模式**：连接到远程同步服务器

同步数据：播放列表、设置、不喜欢列表

位置：`src/main/modules/sync/`

### 9. 开放 API 服务器

用于外部控制的 HTTP REST API（例如从其他应用）。

在设置中启用 → 在指定端口上打开 HTTP 服务器。

位置：`src/main/modules/sync/`

## 文件位置参考

### 入口点
- 主进程：`src/main/index.ts`
- 主窗口渲染进程：`src/renderer/main.ts`
- 歌词窗口渲染进程：`src/renderer-lyric/main.ts`
- 用户 API 预加载：`src/main/modules/userApi/renderer/preload.js`

### 关键模块
- IPC 注册表：`src/common/ipcNames.ts`
- 类型定义：`src/common/types/`
- 数据库 worker：`src/main/worker/dbService/`
- 主模块：`src/main/modules/index.ts`
- 渲染进程状态：`src/renderer/store/`
- 路由：`src/renderer/router.ts`

### 构建配置
- 开发运行器：`build-config/runner-dev.js`
- 生产构建：`build-config/pack.js`
- Webpack 配置：`build-config/{main,renderer,renderer-lyric,renderer-scripts}/webpack.config.{base,dev,prod}.js`

### 静态资源
- 图标/图片：`src/static/`
- SVG 精灵图：`src/renderer/assets/svgs/`（通过 svg-sprite-loader 自动加载）
- 主题：`src/common/theme/`

## 测试

目前**未配置测试框架**。没有 Jest、Vitest 或其他测试运行器。

如果实现测试，考虑：
- Worker 线程模块的单元测试
- IPC 通信的集成测试
- 使用 Spectron 或 Playwright 的 Electron 应用 E2E 测试

## 常见陷阱

1. **Windows 上的路径分隔符**：始终使用 `path.join()`，永远不要用 `+` 或 `/` 连接
2. **Worker 异步调用**：所有 worker 方法都返回 Promise - 始终 `await` 它们
3. **多个 webpack 配置**：构建更改通常需要更新 3-4 个 webpack 配置
4. **类型上下文分离**：主进程和渲染进程有独立的 TypeScript 上下文 - 共享类型必须放在 `src/common/types/`
5. **渲染进程中的 Node.js**：出于安全考虑默认禁用（预加载脚本除外）
6. **CSS 作用域**：在 Vue 组件中使用 `<style scoped>` 或 CSS 模块以避免全局污染
7. **Pug 空格**：Pug 模板对空格敏感 - 注意缩进
8. **SVG 精灵图加载器**：`src/renderer/assets/svgs/` 中的 SVG 会自动加载 - 不要手动导入
9. **窗口事件**：使用全局事件发射器（`global.lx.event_*`）而不是直接 IPC 进行进程内通信
10. **数据库迁移**：修改数据库结构时，始终更新 `db_info` 表中的架构版本
