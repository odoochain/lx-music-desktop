# LX Music Desktop 编译打包指南

## 环境要求

- **Node.js** (推荐 v18+)
- **npm**
- **Windows** (打包 exe 需要)
- **proxychains** (可选，网络不畅时使用)

## 编译步骤

整个过程分为两步：**构建 (Build)** → **打包 (Pack)**

### 第一步：构建所有进程

```bash
npm run build
```

该命令并行编译 4 个 webpack 进程：

| 进程 | 入口 | 输出 |
|---|---|---|
| 主进程 | `src/main/index.ts` | `dist/main.js`、`dist/dbService.worker.js` |
| 渲染进程（主窗口） | `src/renderer/main.ts` | `dist/renderer.js`、`dist/index.html` |
| 渲染进程（歌词窗口） | `src/renderer-lyric/main.ts` | `dist/renderer-lyric.js`、`dist/lyric.html` |
| 用户 API 脚本 | `src/main/modules/userApi/renderer/preload.js` | `dist/user-api-preload.js` |

耗时约 **40 秒**，产物输出到 `dist/` 目录。

### 第二步：打包为 exe

#### 安装包（推荐）

```bash
npm run pack:win:setup:x64
```

生成 NSIS 安装程序：`build/lx-music-desktop-v{version}-x64-Setup.exe`

#### 其他打包方式

| 命令 | 说明 |
|---|---|
| `npm run pack` | 等同于 build + setup:x64（一步到位） |
| `npm run pack:win` | 打包所有 Windows 架构（x64/x86/arm64/x86_64） |
| `npm run pack:win:7z:x64` | 绿色版 7z 压缩包 |
| `npm run pack:win:portable:x64` | 便携版 exe |
| `npm run pack:linux` | 打包所有 Linux 版本 |
| `npm run pack:mac` | 打包 macOS dmg |

打包耗时约 **1-2 分钟**（首次需下载 NSIS 等工具，后续会缓存）。

## 网络问题处理

### 症状

打包时卡在下载步骤，报错类似：

```
Get "https://github.com/electron-userland/electron-builder-binaries/releases/download/winCodeSign-2.6.0/winCodeSign-2.6.0.7z": 
read tcp ... wsarecv: A connection attempt failed because the connected party did not properly respond
```

electron-builder 需要从 GitHub 下载以下工具（首次打包时）：

- `winCodeSign-2.6.0.7z` (~5.6 MB) — exe 资源编辑和签名
- `nsis-3.0.4.1.7z` (~1.3 MB) — NSIS 安装程序制作
- `nsis-resources-3.4.1.7z` (~731 KB) — NSIS 资源文件

### 解决方案：使用 proxychains

当无法直接访问 GitHub 时，在命令前加 `proxychains`：

```bash
proxychains npm run pack:win:setup:x64
```

`proxychains` 会将网络请求通过本地代理（通常是 `localhost:10808`）转发，顺利下载所需工具。

下载完成后，工具会缓存在 `%LOCALAPPDATA%/electron-builder/Cache/` 目录，后续打包无需重复下载。

### 注意事项

- `proxychains` 需要本地有代理服务运行（如 Clash、V2Ray 等）
- 安装方式：`scoop install proxychains-ng`（Windows）
- 下载的工具缓存在 `%LOCALAPPDATA%/electron-builder/Cache/winCodeSign/` 和 `Cache/nsis/` 等目录

## 常见问题

### Q: 构建失败但没报具体错误？

检查 `dist/` 目录是否完整。可单独构建某个进程排查：

```bash
npm run build:main
npm run build:renderer
npm run build:renderer-lyric
npm run build:renderer-scripts
```

### Q: 打包后 exe 无法运行？

1. 确认 `dist/` 目录已先执行过 `npm run build`
2. 检查 `build/win-unpacked/lx-music-desktop.exe` 是否可直接运行
3. 检查 native 模块是否正确编译（better-sqlite3、bufferutil 等）

### Q: 如何清理重新构建？

```bash
# 清理构建产物
rm -rf dist/ build/

# 重新完整构建 + 打包
npm run pack
```

### Q: 如何只打包不安装？

使用 `pack:dir` 生成免安装的目录：

```bash
npm run pack:dir
```

产物在 `build/win-unpacked/` 目录，直接运行 `lx-music-desktop.exe` 即可。

## 一键编译打包命令

```bash
# 正常网络
npm run pack

# 网络不畅（需要代理）
proxychains npm run pack
```
