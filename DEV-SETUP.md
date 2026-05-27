# 开发环境搭建与问题记录

## 快速启动

```bash
npm install          # 安装依赖
npm run dev          # 编译并启动开发模式
```

启动后会同时运行：
- 主窗口渲染进程 webpack-dev-server：`http://localhost:9080/`
- 歌词窗口渲染进程 webpack-dev-server：`http://localhost:9081/`
- 主进程 webpack watch（文件修改后自动重启 Electron）
- 用户 API 脚本 webpack watch
- Electron（带 `--inspect=5858` 调试端口）

首次编译耗时约 60 秒，后续增量编译约 1 秒。

## 已知问题与解决方案

### 1. Electron 二进制文件安装失败

**症状**：运行 `npm run dev` 报错：
```
Error: Electron failed to install correctly, please delete node_modules/electron and try installing again
```

**原因**：`node_modules/electron/path.txt` 文件缺失或内容异常（如尾部带换行符），导致 Electron 找不到 `electron.exe`。

**排查**：
```bash
# 检查 path.txt 是否存在
cat node_modules/electron/path.txt

# 检查是否有尾部换行符
xxd node_modules/electron/path.txt
# 正确输出应为: 656c 6563 7472 6f6e 2e65 7865 (electron.exe)
# 带换行符输出: 656c 6563 7472 6f6e 2e65 7865 0a (尾部多 0a)

# 检查二进制文件是否存在
ls node_modules/electron/dist/electron.exe
```

**修复方案一**：重新安装 Electron
```bash
rm -rf node_modules/electron
npm install
# 如果 install.js 未自动下载二进制，手动执行：
node node_modules/electron/install.js
```

**修复方案二**：手动解压（当 install.js 静默失败时）
```bash
# 1. 找到缓存的 zip 文件
ls "$LOCALAPPDATA/electron/Cache/"
# 在子目录中查找 electron-v*-win32-x64.zip

# 2. 手动解压
mkdir -p node_modules/electron/dist
cd node_modules/electron/dist
unzip "<缓存路径>/electron-v*-win32-x64.zip"

# 3. 创建 path.txt（注意不要有尾部换行）
printf 'electron.exe' > node_modules/electron/path.txt
```

**修复方案三**：仅修复 path.txt 换行问题
```bash
printf 'electron.exe' > node_modules/electron/path.txt
```

### 2. SQLite 数据库打开失败

**症状**：Electron 启动后控制台报错：
```
SqliteError: unable to open database file
code: 'SQLITE_CANTOPEN'
```

**原因**：首次运行时用户数据目录（`LxDatas`）尚不存在，better-sqlite3 无法创建数据库文件。

**说明**：此错误通常不影响应用启动，应用会在后续操作中自动创建所需目录和数据库。如果持续报错，可手动创建数据目录：
- Windows: `%APPDATA%/lx-music-desktop/LxDatas/`
- 便携模式: `{exeDir}/portable/userData/LxDatas/`

## 调试方式

| 目标 | 方法 |
|------|------|
| 主进程 | Chrome DevTools 访问 `chrome://inspect` 或 `localhost:5858` |
| 渲染进程 | 应用内按 `Ctrl+Shift+I` 打开 DevTools |
| Worker 线程 | `console.log` 输出在对应进程的控制台 |

## 注意事项

- Electron 版本：`^40.10.1`（当前安装 40.10.1）
- 项目使用 Node.js `worker_threads` + Comlink 处理数据库操作，不要从主线程直接访问数据库
- 修改 `src/common/` 下的代码可能需要重启所有进程
- Windows 上始终使用 `path.join()` 拼接路径
