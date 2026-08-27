# Falix Timer 自动续期

> 针对 `https://client.falixnodes.net/timer?id=<SERVER_ID>` 的 GitHub Actions 自动化续期脚本  
> 已通过浏览器实测：页面显示 `71 小时 xx 分` + `添加时间`按钮，点击需登录后弹出 `Watch Ad` 广告视频，播放完成后提示 `Timer has been extended`。

## 功能

- **计时器续期**：自动打开 `https://client.falixnodes.net/timer?id=<SERVER_ID>` → 处理 Turnstile → 点击 `Add Time/添加时间` → 点击 `Watch Ad/观看广告` → 自动播放 `video` / `vjs-big-play-button` → 等待 90s 直到 `Timer has been extended/计时器已`，重试 3 轮
- **服务器保活**：登录后先访问 `/server/<ID>/console`，若状态为 `offline` 则自动点击 `Start` → `Watch Ad` → 播放广告等待 30s，最多 3 轮
- **真人浏览器**：`puppeteer-real-browser` + `turnstile:true` 自动过 Cloudflare Turnstile，CDP 坐标点击规避 React 受控输入
- **WARP 加速**：`fscarmen/warp-on-actions` 解决 IMA 广告不加载
- **自触发续命**：20 分钟强制 `repository_dispatch` 触发下一轮，避免 6 小时 Actions 限制；`schedule: */15 * * * *` 作为兜底
- **Cookie 缓存**：登录成功后缓存 `cookies.json`，下次复用免登录

## 快速开始

### 1. Fork 本仓库

Fork 到你自己的 GitHub 账号。

### 2. 配置 Secrets

`Settings` → `Secrets and variables` → `Actions` → `New repository secret`：

| Secret | 必填 | 说明 | 示例 |
|---|---|---|---|
| `FALIX_EMAIL` | ✅ | Falix 登录邮箱 | `you@example.com` |
| `FALIX_PASSWORD` | ✅ | Falix 密码 | `xxx` |
| `FALIX_SERVER_ID` | ✅ | 服务器 ID，即 timer 链接的 `id` 参数 | `YOUR_SERVER_ID` |
| `TG_TOKEN` | 可选 | Telegram Bot Token（用于通知） | `123456:ABC...` |
| `TG_CHAT_ID` | 可选 | Telegram Chat ID | `123456789` |

> 需在 Secrets 中配置 `FALIX_SERVER_ID`（从 `timer?id=` 链接获取）。

### 3. 触发运行

- **自动**：每 15 分钟 `schedule` 触发 + 每次运行结束自触发 `falix-start`
- **手动**：`Actions` → `Falix Auto Start` → `Run workflow` → `Run workflow`
- **API 触发**：
```bash
curl -X POST https://api.github.com/repos/<用户名>/<仓库名>/dispatches \
  -H "Authorization: Bearer <PAT>" \
  -H "Accept: application/vnd.github+json" \
  -d '{"event_type":"falix-start"}'
```

### 4. 查看结果

- `Actions` 日志查看 `time before extend / time after extend` 和 `timer extended ok`
- `Artifacts` 下载 `screenshots`（保留 3 天）排查失败界面
- Telegram 接收 `✅ 计时器已续期: 259xxx s → 262xxx s (+3600s)` 通知

## 工作流说明

- `falix-auto-start.yml:4`：`push / schedule / workflow_dispatch / repository_dispatch` 四种触发
- `falix-auto-start.yml:59`：WARP dual 栈
- `falix-auto-start.yml:82` / `698`：仅重新登录时保存 cookie，避免无效覆盖
- `falix-auto-start.yml:112`：20 分钟 `LIMIT` 强制触发下一轮，保持常驻
- `Keepalive.yml:4`：每 3 天 `17 3 */3 * *` 自动提交 `keep-alive.txt` 防止仓库 60 天休眠

## 本地调试

```bash
# 安装依赖
npm init -y && npm install puppeteer-real-browser@1.4.4 puppeteer-core@25.7.0

# 设置环境变量后运行（需本机 Chrome）
set FALIX_EMAIL=you@example.com
set FALIX_PASSWORD=xxx
set FALIX_SERVER_ID=YOUR_SERVER_ID
node run.mjs
```

`run.mjs` 由 workflow 的 `Write script` 步骤动态生成，本地可直接复制该段 `script` 保存为 `run.mjs`。

## 常见问题

- **登录失败**：检查邮箱/密码、Turnstile 是否超时（日志 `login turnstile unconfirmed`），WARP 是否生效
- **no extend button**：计时器可能已满或页面未加载，查看截图 `timer_01_loaded.png`
- **video never got a src**：IMA 广告未加载，确认 WARP 步骤 `warp=on`，DNS 已切 `1.1.1.1`
- **计时器时间未增加**：已改用通用校验 `timeAfter > timeBefore`，71 小时大计时器也会正确判定为成功

## 验证记录

- 2026-08-27 浏览器实测：未登录访问 `timer?id=<SERVER_ID>` 显示 `71 小时 xx 分` + `添加时间`，点击后重定向至 `/auth/login`（需登录态才能续期），已在脚本中实现完整登录→续期闭环。
