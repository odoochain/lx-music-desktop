# QQ会员·自备Cookie 音源 — 开发说明

> 适用脚本：`全豆要-聚合音源-V4.1.js`（LX Music 自定义音源 / userApi）
> 音源 ID：`qqvip`　音源名称：`QQ会员·自备Cookie`
> 文档日期：2026-07-08

---

## 1. 功能概述

利用**用户本人已登录的 QQ 音乐会员 Cookie**，直接调用 QQ 官方 `musicu.fcg` 接口获取
VIP 音质（128k / 320k / flac / flac24bit 臻品母带）的真实播放地址。
意义：在不依赖第三方代理/破解服务的前提下，用本人合法权益拉取无损音源。

- 该音源只负责**播放取链（`musicUrl`）**，没有 `musicSearch` / `lyric` 能力
  （见脚本 `sourceConfig[QQ_VIP_SOURCE_ID].actions = ["musicUrl"]`）。
- 因此**不能用来搜索歌曲**，只能作为「换源播放」：先在其它音源搜到歌曲，再切到本音源放。

---

## 2. 取链原理与流程

```
歌曲 songmid
   │
   ▼  getQQMediaMid()   请求 music.pf_song_detail_svr
   │                    拿到 file.media_mid（媒体文件 mid）
   ▼
拼 vkey 请求文件名字段  M500<mediaMid>.mp3 / F000<mediaMid>.flac / RS01<mediaMid>.flac
   │
   ▼  qqVipGetUrl()     请求 vkey.GetVkeyServer (CgiGetVkey)，带本人 Cookie + 签名
   │                    返回 purl + sip（CDN 域名）
   ▼
拼接最终地址  sip[0] + purl
   │
   ▼  validateUrl()     探测可播放（206）后返回
```

关键接口：

| 步骤 | 接口 | 作用 |
|---|---|---|
| 取 media_mid | `https://u.y.qq.com/cgi-bin/musicu.fcg?data={music.pf_song_detail_svr}` | 拿到文件真正使用的 media_mid |
| 取 vkey | `https://u.y.qq.com/cgi-bin/musicu.fcg?...`（`module=vkey.GetVkeyServer`） | 用 Cookie 换取 VIP 播放 purl |

---

## 3. 关键实现要点

### 3.1 签名相关

- `md5()`：标准 MD5 实现，用于 `musicu.fcg` 的请求签名（与官方 Web 端一致）。
- `getQQVipUin()`：从 `QQ_COOKIE` 解析 `uin=`。
- `getQQVipPskey()`：依次尝试 `p_skey` / `pskey` / `qm_keyst` / `qqmusic_key`
  解析登录票据（**优先用 `p_skey`/`pskey`，但实测网页端 `qm_keyst`/`qqmusic_key` 也能算通 g_tk**）。
- `getQQVipGTK(pskey)`：经典 `g_tk` 算法（`hash = 5381; hash += (hash<<5)+c`）。
- `qqMusicuSign(params)`：除 `sign` 外的所有参数按 key 升序以 `&` 连接 `k=v`，
  两端加盐 `QQ_VIP_SIGN_SALT` 后 `md5`。盐值见脚本常量 `QQ_VIP_SIGN_SALT`。

### 3.2 `getQQMediaMid(songmid)` —— 本次修复的核心

免费（无 Cookie、`uin=0`）请求 `music.pf_song_detail_svr`，从返回的
`track_info.file` 中取：

- `mediaMid = file.media_mid`（**vkey 文件名必须用此值**）
- `sizeFlac` / `sizeHires`（用于 flac24bit 可用性判断）

### 3.3 `qqVipGetUrl(songInfo, quality)`

1. 校验 `QQ_COOKIE` 非空、`getQQSongId(songInfo).type === "mid"`。
2. 调 `getQQMediaMid` 拿到 `mediaMid` / `sizeHires`。
3. 映射音质：

   | quality | 文件名前缀 | 扩展名 |
   |---|---|---|
   | 128k | `M500` | mp3 |
   | 320k | `M800` | mp3 |
   | flac | `F000` | flac |
   | flac24bit / 24bit | `RS01` | flac |

4. **flac24bit 前置校验**：若 `sizeHires === 0`（歌曲本身无 hi-res 母带），
   直接抛出「该歌曲无臻品母带（hi-res）资源，请选择 flac 音质」，避免无谓的 -46628。
5. 组装 `vkey.GetVkeyServer` 请求，带 `loginflag=1`、`platform="20"`、本人 Cookie。
6. 取 `sip[0] + purl` 作为最终地址，`validateUrl` 探测通过后返回。

---

## 4. 本次修复根因（-46628 "file not exist"）

**现象**：vkey 请求 `code:0` 成功返回，但所有 CDN 域名（`aqqmusic.tc.qq.com`、
`ws.stream.qqmusic.qq.com`、`isure.stream.qqmusic.qq.com` 等）探测均返回
`-46628 file not exist`，免费/登录/VIP 任意形态都失败。

**根因**：vkey 的 `filename` 字段错误地用了歌曲 `songmid`，
而 QQ 要求使用**媒体文件 mid `file.media_mid`**——两者**通常是不同的字符串**。
例：《晴天》`songmid=0039MnYb0qxYhV`，但其媒体文件 id 是 `003Qui1q2u1Zho`。
`GetVkeyServer` 对签名较宽松（错用 songmid 也照发 vkey），真正在 **CDN 取字节时才校验文件是否存在**，
于是暴露 -46628。这与签名、Cookie、`g_tk`、VIP 权益均无关（那些一直是对的）。

**修复**：新增 `getQQMediaMid()`，在取链前用免费接口拿到 `file.media_mid`，
并以它拼 `filename`（如 `M500003Qui1q2u1Zho.mp3`）。

---

## 5. 配置方法

脚本顶部常量（示例，非真实值）：

```js
const QQ_COOKIE = "<浏览器登录QQ音乐后复制的完整 Cookie>";
```

- 浏览器登录 https://y.qq.com/ → F12 → Network/Application 复制完整 Cookie。
- 需含 `uin`、`qqmusic_key` / `qm_keyst`（或 `p_skey` / `pskey`）。
- `QQ_COOKIE` 留空则本音源不可用（会报「未配置Cookie」）。

> ⚠️ 安全提醒：`QQ_COOKIE` 是明文高敏感凭证，**不要提交进 Git / 外传**。

---

## 6. 使用方式

1. 设置 → 自定义源 → **导入脚本**，选择 `全豆要-聚合音源-V4.1.js`。
2. 顶部音源列表出现「QQ会员·自备Cookie」。
3. 在其它音源（如汽水、QQ 免费）搜到歌曲 → 播放/换源时选择「QQ会员·自备Cookie」
   → 即可拉取本人 VIP 的 flac 等音质。

**限制**：本音源无搜索能力，必须「借其它源搜、用本源放」。

---

## 7. 本机实测结果（用户本人会员 Cookie + 修复后逻辑）

| 音质 | 结果 |
|---|---|
| 128k | ✅ 206 可播放 |
| 320k | ✅ 206 可播放 |
| flac | ✅ 206 可播放（**VIP 权益生效**） |
| flac24bit（晴天） | 46628 —— 因该曲本身无 hi-res 母带，属正常限制 |

结论：QQ会员·自备Cookie 音源已可用，flac 无损可经本人会员权益拉取；
flac24bit 仅对确实带臻品母带的歌曲有效。

---

## 8. 注意事项

- **Cookie 会过期**：会员到期或登录失效后 flac 会报「Cookie可能过期 / 该音质无VIP权限」，
  需重新从浏览器复制最新 Cookie 并重新导入脚本。
- 修复前的临时诊断脚本（`qq_check.js` / `qq_diag3~7.js` 及其 `*_out.txt`）为调试产物，
  与功能无关，确认无误后可删除。
- 本音源为「本人权益使用」，请遵守 QQ 音乐服务条款与版权规定。
