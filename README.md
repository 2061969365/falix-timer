# Falix Timer 自动续期

> 针对 `https://client.falixnodes.net/timer?id=<SERVER_ID>` 的 GitHub Actions 自动化续期脚本  
> 2026-11 新版：登录页是 Next.js 受控表单（脚本会等 React hydration 完成后用原生 setter 填表）；计时器页显示 `167 小时 xx 分 xx 秒`，点击「添加时间」直接调用 `/timer/extend`，返回 `Timer has been extended.`，不再播放广告。

## 功能

- **计时器续期**：打开 timer 页 → 处理 Turnstile → 点击 `Add Time/添加时间` → 调用 `/timer/extend` → 回到 timer 页读取新读数，读数增加即成功；单次最多补 3 次，计时器达到 168h 上限时自动停止
- **服务器保活**：登录后访问 `/server/<ID>/console`，离线时点击 `Start/启动`，轮询状态直到 `starting/online`，最多 3 轮
- **真人浏览器**：`puppeteer-real-browser` + `turnstile:true` 自动处理 Cloudflare Turnstile；登录表单在 hydration 后直接写值，避免被 React 覆盖或焦点被 solver 抢走
- **WARP 加速**：`fscarmen/warp-on-actions` 解决广告/IMA 加载问题
- **定时调度**：`schedule: 17 2 * * *` 每天 02:17 UTC 运行一次
- **Cookie 缓存**：按账号缓存到 `cookies.json`（key 为 serverId），下次复用免登录；切账号前清 cookie/localStorage/IndexedDB，避免串号
- **多账号**：`FALIX_ACCOUNTS` JSON 数组按顺序处理，单个账号失败不影响其他账号；全部结束后有失败则 exit 1
- **权限最小化**：job 显式声明 `contents: read` + `actions: write`，旧 run 清理不会再 403

## 快速开始

### 1. Fork 本仓库

Fork 到你自己的 GitHub 账号。

### 2. 配置 Secrets

`Settings` → `Secrets and variables` → `Actions` → `New repository secret`：

| Secret | 必填 | 说明 | 示例 |
|---|---|---|---|
| `FALIX_EMAIL` | 单账号 | Falix 登录邮箱 | `you@example.com` |
| `FALIX_PASSWORD` | 单账号 | Falix 密码 | `xxx` |
| `FALIX_SERVER_ID` | 单账号 | 服务器 ID，即 timer 链接的 `id` 参数 | `YOUR_SERVER_ID` |
| `FALIX_ACCOUNTS` | 多账号 | JSON 数组，配置后优先于上面三个 | 见下方 |
| `TG_TOKEN` | 可选 | Telegram Bot Token（用于通知） | `123456:ABC...` |
| `TG_CHAT_ID` | 可选 | Telegram Chat ID | `123456789` |

> 单账号只配 `FALIX_EMAIL` / `FALIX_PASSWORD` / `FALIX_SERVER_ID` 即可；多账号改用 `FALIX_ACCOUNTS`，单账号 secrets 可留作 fallback。

#### 多账号

`FALIX_ACCOUNTS` 的值为 JSON 数组：

```json
[
  {"email":"a@example.com","password":"pass1","serverId":"srv1","label":"主号"},
  {"email":"b@example.com","password":"pass2","serverId":"srv2","label":"小号"}
]
```

- 配置后优先使用；未配置或解析失败时回退到 `FALIX_EMAIL` / `FALIX_PASSWORD` / `FALIX_SERVER_ID`
- 账号按数组顺序串行处理，共用同一个浏览器；每个账号独立 cookie（key = `serverId`），切换前清 cookie/localStorage/IndexedDB
- 某个账号失败只记录并继续下一个；全部结束后有失败则 job 失败（exit 1），日志和 TG 会带 `[序号/总数] label`
- `label` 只用于日志/通知，不要填邮箱等敏感信息；Telegram 里的 `<>&` 会被去掉
- 账号多时把 `timeout-minutes` 调大（默认 60 分钟，约每账号 1.5–2 分钟）

### 3. 触发运行

- **自动**：每天 `02:17 UTC` 定时触发（`schedule: 17 2 * * *`）
- **手动**：`Actions` → `Falix Auto Start` → `Run workflow` → `Run workflow`

### 4. 查看结果

- `Actions` 日志查看 `parse time` / `timer before/after` / `timer extend success` / `status`
- 续期 API 会打印 `POST .../timer/extend` 与 `{"success":true,"message":"Timer has been extended."}`
- `Artifacts` 下载 `screenshots`（保留 3 天）排查失败界面
- Telegram 接收续期/状态通知

## 工作流说明

- 每天一次 `schedule` + `workflow_dispatch` 手动触发
- 多账号：读取 `FALIX_ACCOUNTS` 后串行执行；每个账号独立 cookie，失败隔离，最后汇总 exit code
- WARP dual 栈 + DNS 1.1.1.1，保证广告与 Cloudflare 正常加载
- 登录表单：等待 `__reactProps$`（React hydration）后用原生 setter + `input/change` 事件填值，再回读校验
- 计时器解析：兼容 `167 小时 46 分钟 9 秒`、`167:46:09`、`167 hours 46 minutes 9 seconds`
- `Keepalive.yml`：每 3 天自动提交 `keep-alive.txt` 防止仓库 60 天休眠

## 本地调试

```bash
# 安装依赖
npm init -y && npm install puppeteer-real-browser@1.4.4 puppeteer-core@25.7.0

# 单账号
set FALIX_EMAIL=you@example.com
set FALIX_PASSWORD=xxx
set FALIX_SERVER_ID=YOUR_SERVER_ID

# 或多账号（JSON 一行）
set FALIX_ACCOUNTS=[{"email":"a@example.com","password":"pass1","serverId":"srv1"},{"email":"b@example.com","password":"pass2","serverId":"srv2"}]

node run.mjs
```

`run.mjs` 由 workflow 的 `Write script` 步骤动态生成，本地可直接复制该段 `script` 保存为 `run.mjs`。

## 常见问题

- **登录失败**：检查日志是否出现 `React ready` 与 `email/password OK (js)`；若为 `login failed: ...`，检查 Secrets、Turnstile 是否超时、WARP 是否生效
- **账号启用 2FA**：脚本会输出 `login requires 2FA`，该账号记为失败并继续其他账号；需要关闭 2FA 或改用 cookie 方案
- **多账号里某个账号失败**：日志/TG 显示 `[i/N] label 失败: ...`，其他账号继续；最后 job 为红，按提示单独修该账号
- **no extend button**：计时器可能已满或页面未加载，查看截图 `timer_01_loaded.png`
- **计时器时间未增加**：168h 封顶时属正常；日志会打印 `timer near cap`，此时 `timer/extend` 仍然返回成功
- **登录后 cookie 很快失效**：站点会话较短，脚本每次会重新登录；`cookies.json` 仅作加速

## 验证记录

- 2026-11-03：定位到登录失败根因——旧脚本在 hydration 前输入，且 `turnstile:true` 的 solver 每秒点击验证码区域抢走焦点。改为 hydration 后原生 setter 填表；run 41672488560 / 41676789538 通过（`email/password OK (js)`、`login ok`、`status: online`）
- 2026-11-03：新版计时器格式与加时流程。run 41677473480 日志显示 `POST .../timer/extend` → `Timer has been extended.`，读数 `604567s → 604791s`（168h 封顶），判定成功
- 2026-11-27：多账号改造（`FALIX_ACCOUNTS`）合入。单账号 fallback E2E：run 46018064503 登录后续期 `591173s → 604789s`；run 46018486807 复用 6 个 cookie，`status: offline` 时点击「启动」→ `server already online without ad`，续期至 `604788s`
