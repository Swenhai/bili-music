# 本 Fork 修改说明

> 原项目：[tgymwnet/bili-music](https://github.com/tgymwnet/bili-music)（B 站音乐播放器，Electron + Vue3 + Quasar）
> 本 Fork 由 [Swenhai](https://github.com/Swenhai) 维护，修改时间：2026-11-03

## 修改内容

### 1. 联网修复：绕过系统代理（electron/main.cjs）

**问题**：系统设置了 HTTP 代理（如 127.0.0.1:7821）时，Chromium 渲染进程会把发往本地代理
`127.0.0.1:25588` 的内部请求也交给系统代理转发，导致失败，界面表现为「无法联网」。

**修改**：`app.whenReady()` 中强制会话直连：

```js
await session.defaultSession.setProxy({ mode: 'direct' })
```

### 2. 搜索修复：适配 B 站 wbi 风控（src/api/bilibili.ts）

**问题**：B 站风控升级后，旧搜索接口 `/x/web-interface/search/type` 返回 HTML 拦截页，
搜索永远显示「未找到相关内容」（实际网络是通的）。

**修改**：搜索改走 wbi 路径（代码本身已有 `signParams` wbi 签名，只是 URL 少了 `/wbi`）：

```
/x/web-interface/search/type  →  /x/web-interface/wbi/search/type
```

### 3. 新功能：合集 / 多 P 拆分为单曲

**参考**：[kenmingwang/azusa-player（电梓播放器）](https://github.com/kenmingwang/azusa-player)
的分 P / 合集拆单曲思路，未直接引用其代码，仅借鉴交互设计，在原架构上独立实现。

**改动文件与内容**：

- `src/api/bilibili.ts` — 新增 `getVideoDetail(bvid)`：一次 `view` 请求同时取
  `pages[]`（多 P 视频，每 P 独立 cid）和 `ugc_season.sections[].episodes[]`
  （UGC 合集，每集独立 bvid），新增 `SplitSong` / `VideoDetail` 类型。
- `src/pages/SearchPage.vue` — 搜索结果条新增「拆分合集为单曲」按钮；
  弹出拆分列表对话框，支持单曲播放、单曲/全部加入播放队列、正在播放高亮；
  点击多 P / 合集类结果时自动弹出拆分列表。
- `src/stores/player.ts` — 播放队列去重与定位从 `bvid` 改为 `id`
  （拆分单曲共享同一 bvid，`id = bvid::cid` 才能区分不同分 P）。
- `src/components/PlayerBar.vue` — `playFromQueue` 优先使用曲目自带的 `cid`，
  修复多 P 队列自动连播时永远回到第一 P 的问题。

## 引用清单

| 引用 | 用途 |
|---|---|
| [B 站 Web API](`/x/web-interface/view`、`/x/web-interface/wbi/search/type`、`/x/player/playurl`、`/x/frontend/finger/spi`、`/x/passport-login/web/qrcode/*`) | 视频详情/分P/合集、搜索、音频流、风控 cookie、扫码登录 |
| WBI 签名算法（原项目自带 `src/utils/wbi.ts`） | 搜索接口参数签名 |
| [azusa-player](https://github.com/kenmingwang/azusa-player) 的分 P/合集拆单曲交互思路 | 合集拆分功能的设计参考 |

## 构建

```bash
npm install --registry=https://registry.npmmirror.com   # Electron 二进制建议设 ELECTRON_MIRROR
npx vite build
npx electron-builder --config.npmRebuild=false          # 本机无 VS 编译链时跳过原生重建
```

其余功能与原项目一致，未做改动。
