# ntfy on Cloudflare Workers — 完整搭建与使用教程

> 一个用 **Cloudflare Workers + Durable Objects(SQLite) + R2** 原生重写的 ntfy 兼容服务端。
> 官方 ntfy Android / iOS / Web 客户端可以直接连过来用，接口路径、头字段、返回格式都按 ntfy v1 规范实现。
>
> **第一次上手、没接触过 Cloudflare？** 直接看同目录的 **[小白操作手册.md](./小白操作手册.md)**：16 步，每步都写了「做什么 / 为什么 / 输入什么 / 该看到什么 / 出问题怎么办」。本文档偏技术参考，手册偏手把手。
>
> **不想在本机装 Node、不想碰命令行？** 用 GitHub Actions 全自动部署：见 **[GitHub-Actions-部署教程.md](./GitHub-Actions-部署教程.md)**。代码推到 GitHub、填 2 个密钥、点一次 Run，部署 + 冒烟测试全自动完成。本节第 3~7 步（本机装环境、手动 deploy）可以整段跳过。

---

## 目录

- [0. 先看结论：这套东西能干什么、不能干什么](#0-先看结论这套东西能干什么不能干什么)
- [1. 前置条件](#1-前置条件)
- [2. 项目结构](#2-项目结构)
- [3. 第一步：装环境并登录 Cloudflare](#3-第一步装环境并登录-cloudflare)
- [4. 第二步：创建 R2 存储桶](#4-第二步创建-r2-存储桶)
- [5. 第三步：本地跑起来验证](#5-第三步本地跑起来验证)
- [6. 第四步：跑冒烟测试](#6-第四步跑冒烟测试)
- [7. 第五步：正式部署](#7-第五步正式部署)
- [8. 第六步：绑定自己的域名](#8-第六步绑定自己的域名)
- [9. 第七步：客户端接入（手机 App / curl / 你的挪车系统）](#9-第七步客户端接入手机-app--curl--你的挪车系统)
- [10. 第八步：可选开启访问密码](#10-第八步可选开启访问密码)
- [11. 配置项速查表](#11-配置项速查表)
- [12. API 兼容清单](#12-api-兼容清单)
- [13. 常用命令速查（curl / PowerShell）](#13-常用命令速查curl--powershell)
- [14. 费用与额度](#14-费用与额度)
- [15. 已知限制（务必先读）](#15-已知限制务必先读)
- [16. 排错手册](#16-排错手册)
- [17. 后续可扩展的方向](#17-后续可扩展的方向)
- [18. 部署方式二：GitHub Actions 全自动部署](#18-部署方式二github-actions-全自动部署)

---

## 0. 先看结论：这套东西能干什么、不能干什么

### 能干的

| 能力 | 说明 |
| --- | --- |
| 发布消息 | `POST/PUT /{topic}`，支持 `Message/Title/Priority/Tags/Click/Actions/Markdown/Cache` 等头 |
| JSON 发布 | `POST /`，body 里写 `{"topic":"xxx","message":"..."}` |
| 实时订阅 | `/{topic}/json`（NDJSON 长连接）、`/{topic}/sse`（Server-Sent Events）、`/{topic}/raw`（纯文本流） |
| WebSocket | `/{topic}/ws`，用 Hibernation API，手机锁屏挂着几乎不烧额度 |
| 历史补拉 | `?since=all / latest / 10m / 1720000000 / 消息ID`，`?poll=1` 一次性拉取 |
| 消息过滤 | `?id= / message= / title= / priority= / tags=` |
| 附件 | multipart 上传、`Filename:` 头原始二进制上传、`Attach:` 远程 URL 抓取，存 R2，返回 `/file/xxx` 链接 |
| 消息缓存 | 每个 topic 独立 SQLite，默认缓存 12 小时，过期自动清理（含 R2 附件） |
| 内置网页 | 浏览器打开 `/` 或 `/{topic}`，可视化收发测试 |
| 元信息 | `/v1/health`、`/v1/config`（App 注册服务器时读这个） |

### 不能干的（刻意的取舍）

| 缺口 | 原因 | 影响 |
| --- | --- | --- |
| iOS 后台推送 | 没有 APNs 证书链路 | iOS 端 App 前台/短时后台能收，**杀进程后收不到**。Android 靠自建长连接，不受影响 |
| 用户登录 / 注册 / 付费 | YAGNI，个人用不上 | `/v1/config` 里 `enable_login=false` |
| Web Push / 邮件通知 / 定时投递 / 短信 | 同上 | App 里这些开关会灰掉或无效 |
| 多 topic 合并订阅（`/a,b/json`） | 一个连接只服务一个 DO | 返回 400 |
| 消息持久期超长 | 免费额度与存储成本 | 默认 12h，可调 |

---

## 1. 前置条件

清单如下，缺一项都会卡住：

1. **Cloudflare 账号**（免费版即可，DO 的 SQLite 后端免费计划已开放）。
2. **一个域名托管在 Cloudflare**（推荐，自定义域名需要；只用 `*.workers.dev` 也能跑，但有些网络环境下 workers.dev 不稳定，手机端体验略差）。
3. **Node.js 18+**（建议 20 或 22）。本机检查：
   ```powershell
   node -v
   npm -v
   ```
4. **R2 已开通**。第一次用 R2 需要在控制台点一次开通（会要求绑定付款方式，但 10GB 存储内不收费）。

> 项目已在本机生成好，路径：
> `...\workspace\conv_6584d7454c524546be01ccbced9248ad\output\ntfy-cloudflare`
> 后文所有命令都在这个目录里执行。

---

## 2. 项目结构

```
ntfy-cloudflare/
├── wrangler.toml          # 部署配置：DO 绑定、R2 绑定、环境变量
├── package.json           # dev / deploy / tail / smoke / test 脚本
├── src/
│   ├── index.js           # Worker 入口：路由、发布解析、附件、鉴权、元信息
│   ├── topic-do.js        # TopicDO：每 topic 一个 DO，SQLite 消息缓存 + 广播 + alarm 清理
│   ├── util.js            # 公共函数：ID、时长解析、优先级、CORS、错误响应
│   └── ui.js              # 内置调试网页（无框架，纯原生 JS）
├── scripts/
│   └── smoke-test.mjs     # 冒烟测试：打真实/本地服务的端到端自检
└── test/
    └── offline-test.mjs   # 离线自测：node:sqlite + 内存 R2 打桩，32 项断言，无需网络
```

请求流向：

```
手机/Browser/curl
      │
      ▼
Worker (index.js)  ── 解析路由、头字段、附件落 R2
      │
      ▼
TopicDO (id = topic 名)
   ├── SQLite  : messages 表（id/t/expires/body/title/priority/tags/attach…）
   ├── WebSocket: Hibernation API，广播 + 自动保活
   └── alarm   : 定时删过期消息 + 删对应 R2 附件
```

**为什么一个 topic 一个 DO**：消息天然按 topic 分组，DO 是单线程串行执行、天然保序，还自带 SQLite 和 alarm 定时器。一个 DO 扛几万条消息轻轻松松。

---

## 3. 第一步：装环境并登录 Cloudflare

### 3.1 安装依赖

```powershell
cd "C:\Users\Administrator\AppData\Roaming\Tencent\Marvis\User\oAN1i2XA3P-syKMIdXSbSD10Zjn8\workspace\conv_6584d7454c524546be01ccbced9248ad\output\ntfy-cloudflare"
npm install
```

装完会看到 `node_modules\wrangler`。验证：

```powershell
npx wrangler --version
```

### 3.2 登录

```powershell
npx wrangler login
```

会弹浏览器，点 **Allow** 授权。终端显示 `Successfully logged in` 即完成。

检查当前账号：

```powershell
npx wrangler whoami
```

> 如果你想用 API Token 而不是浏览器登录（例如在服务器上），设两个环境变量：
> ```powershell
> $env:CLOUDFLARE_API_TOKEN="你的Token"
> $env:CLOUDFLARE_ACCOUNT_ID="你的AccountID"
> ```
> Token 权限至少给：Workers Scripts:Edit、Durable Objects:Edit、Workers R2 Storage:Edit、Account Settings:Read。

---

## 4. 第二步：创建 R2 存储桶

在控制台 **R2 → Overview** 点一次开通（只需一次），然后：

```powershell
npx wrangler r2 bucket create ntfy-attachments
```

看到 `Created bucket 'ntfy-attachments'` 就成了。

> 桶名可以自己改，但必须同步改 `wrangler.toml` 里的 `bucket_name`，两边要一致。

确认桶存在：

```powershell
npx wrangler r2 bucket list
```

---

## 5. 第三步：本地验证（两种方式，至少跑通 A）

### 方式 A：离线自测（推荐，100% 可跑，不需要网络）

这套自测用 Node 自带的 `node:sqlite` 打桩 Durable Object 的 `storage.sql`、用内存 Map 打桩 R2，然后**真实调用 `src/index.js` 的 fetch**，覆盖路由、鉴权、发布、拉取、过滤、SSE、附件上传下载等 32 项断言。

```powershell
cd "...\output\ntfy-cloudflare"
npm test
```

预期结尾：

```
结果：通过 32 项，失败 0 项
```

（等价命令：`node --experimental-sqlite test\offline-test.mjs`，需要 Node 22+）

> 我在这里踩过一个真 bug：`topic-do.js` 里取上限时写成了未定义变量 `maxRows`，一旦发布就会 500。
> 就是靠这套自测暴露出来的，现在已经修好。**所以请务必先跑一次 `npm test`**。

### 方式 B：`wrangler dev` 本地联调（可选）

```powershell
npx wrangler dev
```

看到 `Ready on http://127.0.0.1:8787` 后，浏览器打开 `http://127.0.0.1:8787`，填 topic（如 `mytest`）→ 点「订阅」→ 输入内容 → 点「发布」。

> ⚠️ 实测提醒：在本机（Windows 10 + wrangler 4.158.0）上，`wrangler dev` 的 **miniflare Durable Object 会报 internal error**，连一个最简单的 DO 样例都跑不通，这不是本项目代码的问题（健康检查 `/v1/health`、配置 `/v1/config`、静态页面都正常，只有 DO 转发失效）。
> 遇到这种情况别怀疑代码：跳过方式 B，直接跑方式 A 自测，然后往下部署，到线上再用**第 6 步的 smoke-test** 做端到端验证。
> 本地数据在 `.wrangler\state`，想清空就删该目录；端口冲突加 `--port 8788`。

---

## 6. 第四步：跑冒烟测试（打线上地址）

`scripts\smoke-test.mjs` 是**对着真实服务器**跑的端到端脚本，本地 dev 和线上部署后都能用。

部署完成后（见第 7 步），执行：

```powershell
cd "...\output\ntfy-cloudflare"
node scripts\smoke-test.mjs https://ntfy.你的域名.com
```

也可以对本地 dev 用：`node scripts\smoke-test.mjs http://127.0.0.1:8787`

预期输出（10 项全绿）：

```
目标服务器：http://127.0.0.1:8787
测试 topic：smoke-xxxxxx

  ✅ health  {"healthy":true}
  ✅ config  base_url=http://127.0.0.1:8787
  ✅ publish  id=xxxxxxxxxxxx priority=4
  ✅ poll  1 条，命中目标消息
  ✅ filter by id
  ✅ publish json
  ✅ upload + download  23 bytes
  ✅ sse  412 字节
  ✅ websocket  {"id":"...","event":"open",...
  ✅ 404 handling

结果：通过 10 项，失败 0 项
```

有红项就停下排查，别急着部署（见[第 16 节](#16-排错手册)）。

---

## 7. 第五步：正式部署

```powershell
npx wrangler deploy
```

输出里会给出你的 Worker 地址，形如：

```
https://ntfy-edge.<你的账号子域>.workers.dev
```

先直接验证线上：

```powershell
curl.exe https://ntfy-edge.xxx.workers.dev/v1/health
```

返回 `{"healthy":true}` 就成功了。

### 7.1 立刻补一步：设置 BASE_URL

`BASE_URL` 决定了两件事：**附件下载链接的域名**、以及 `/v1/config` 返回给 App 的 `base_url`。留空时用请求的 origin 兜底，能用但不完美（自定义域名后建议显式设置）。

```
wrangler.toml → [vars] → BASE_URL = "https://ntfy.你的域名.com"
```

在最终域名确定后改这一行，再 `npx wrangler deploy` 一次即可。

### 7.2 查看实时日志

```powershell
npx wrangler tail
```

发一条消息，日志里能看到请求路径与错误堆栈。排查线上问题时这个最有用。

---

## 8. 第六步：绑定自己的域名

强烈建议做，理由：`workers.dev` 在部分网络下会抽风；而且手机上填一个干净的子域更省心。

### 情况 A：域名已托管在 Cloudflare（最常见）

1. 控制台 → **Workers & Pages** → 点进 `ntfy-edge`
2. **Settings → Domains & Routes → Add → Custom domain**
3. 填 `ntfy.你的域名.com`，点添加
4. Cloudflare 会自动创建 CNAME 并签发证书，一般 1～3 分钟生效

也可以命令行一次搞定：

```powershell
npx wrangler deploy --route "ntfy.你的域名.com/*" --zone-name "你的域名.com"
```

### 情况 B：域名不在 Cloudflare

把该子域的 NS 或 CNAME 指到 Cloudflare（CNAME 目标形如 `ntfy-edge.xxx.workers.dev`，需开代理橙云），再按情况 A 添加自定义域。

### 绑定后必做

1. 修改 `wrangler.toml` 的 `BASE_URL` 为新域名 → `npx wrangler deploy`
2. 浏览器访问 `https://ntfy.你的域名.com/v1/config`，确认 `base_url` 已是新域名
3. 跑一遍线上冒烟测试：
   ```powershell
   node scripts\smoke-test.mjs https://ntfy.你的域名.com
   ```

---

## 9. 第七步：客户端接入（手机 App / curl / 你的挪车系统）

### 9.1 浏览器（最快验证）

打开 `https://ntfy.你的域名.com` → 输入 topic → 订阅 → 发布。手机浏览器也能用。

### 9.2 Android 官方 ntfy App

1. 应用商店（或 F-Droid / GitHub Releases）装 **ntfy**。
2. 打开 App → 右上角 **Settings → Manage users / Default server**，把服务器地址填成 `https://ntfy.你的域名.com`
   （不同版本入口略有差异：有的在首次订阅时点「Use another server」直接输入地址）
3. 回到主界面 → **+** → 输入 topic 名（例如 `chepai-京A12345`） → Subscribe
4. 从网页/curl 往这个 topic 发一条，手机应在 1 秒内弹出通知

> topic 相当于密码：**知道 topic 名的人都能订阅**。所以别用 `test`、`notice` 这种。建议 `前缀-随机串`，例如 `car-a8f3k2n9`。

### 9.3 iOS 官方 ntfy App

同样把服务器地址改成你的域名、订阅 topic 即可。但注意第 0 节的限制：**iOS 收推送依赖 APNs，本项目没有，所以 App 被杀掉后收不到**。前台或刚切后台时能收到。

### 9.4 curl 发布（最常用）

```powershell
# 纯文本
curl.exe -d "车主要挪车，车牌 京A12345" https://ntfy.你的域名.com/car-a8f3k2n9

# 带标题/优先级/标签/点击跳转
curl.exe -H "Title: 挪车通知" -H "Priority: urgent" -H "Tags: car,warning" -H "Click: https://example.com/detail" -d "请尽快挪车" https://ntfy.你的域名.com/car-a8f3k2n9

# 带附件
curl.exe -F "file=@C:\temp\photo.jpg" -F "message=现场照片" https://ntfy.你的域名.com/car-a8f3k2n9
```

### 9.5 接进你现有的挪车 Worker

建议直接用**JSON 发布**（`POST /`，topic 放在 body 里）：中文标题塞 HTTP 头会乱码，JSON 体没有这个限制，返回值里还带 `id` 方便记账。

```js
async function sendNtfyPush(env, { title, message, priority = 3, tags = ["car"] }) {
  const base = (env.NTFY_BASE_URL || "https://ntfy.sh").replace(/\/+$/, "");
  const host = new URL(base).hostname;
  const topic = env.NTFY_TOPIC || "chepai";

  const headers = { "Content-Type": "application/json; charset=utf-8" };
  // 只有自建服务且配了 token 才带；对官方 ntfy.sh 带了会 401
  if (env.NTFY_TOKEN && host !== "ntfy.sh") headers["Authorization"] = `Bearer ${env.NTFY_TOKEN}`;

  const body = JSON.stringify({
    topic,
    message: String(message ?? ""),
    title: title ? String(title) : undefined,
    priority: Number(priority) || 3,
    tags: Array.isArray(tags) ? tags : [String(tags)],
  });

  // JSON 发布打根路径 /
  const res = await fetch(`${base}/`, { method: "POST", headers, body });
  if (!res.ok) return { ok: false, status: res.status, detail: `${res.status} ${await res.text()}`.slice(0, 300) };
  const data = await res.json().catch(() => ({}));
  return { ok: true, status: 200, id: data.id || "", detail: "ok" };
}
```

对应的变量（Worker 的 Settings → Variables and Secrets）：`NTFY_BASE_URL` = 你的自建域名、`NTFY_TOPIC` = 车牌 topic、`NTFY_TOKEN` = 可选（不鉴权就留空）。

日志状态按返回值记账，别写死 `success`：`ok` → `success`，失败 → `fail` 并把 `detail` 存进 `notify_logs`（邮件 + 推送一成一败记 `partial`，都没配记 `none`）。

**两个坑（你之前踩过，这里再钉一遍）**：

- 对**官方 `ntfy.sh`** 无条件下发 `Authorization` 会 401 —— 要按域名判断该不该带 token。
- 官方 `ntfy.sh` 有**按 IP 的每日配额**，超了返回 42908（`daily message quota reached`）。自建之后就彻底没这个问题了。

### 9.6 其他客户端

任何遵循 ntfy 协议的工具都能接：ntfy CLI（`ntfy publish --server https://你的域名 ...`）、Home Assistant 的 ntfy 集成（填 server + topic）、Node-RED、各种语言的 HTTP 客户端。

---

## 10. 第八步：可选开启访问密码

默认是「topic 即密码」。如果你想再加一层（例如这个域名会被别人看到），设置 `AUTH_TOKEN`：

方式一，改 `wrangler.toml`：

```toml
[vars]
AUTH_TOKEN = "换个长的随机串"
```

方式二，用 secret（推荐，不进 Git）：

```powershell
npx wrangler secret put AUTH_TOKEN
```

开启后所有**发布和订阅**都要带：

```powershell
curl.exe -H "Authorization: Bearer 你的token" -d "hello" https://ntfy.你的域名.com/mytopic
```

客户端侧：Android App 里填用户名 + 密码（用户名随便写，密码填 token）即可，因为代码同时兼容 `Basic` 与 `Bearer`。

> `/v1/health`、`/v1/config`、附件下载 `/file/*` 不校验 token，避免客户端接入握手失败。

---

## 11. 配置项速查表

全部在 `wrangler.toml` 的 `[vars]` 里，改完 `npx wrangler deploy` 生效。

| 变量 | 默认值 | 作用 |
| --- | --- | --- |
| `BASE_URL` | 空 | 对外域名。空则用请求 origin。影响附件链接与 `/v1/config` |
| `MESSAGE_CACHE_DURATION` | `12h` | 消息缓存时长，支持 `30m/12h/7d` |
| `MESSAGE_LIMIT` | `2048` | 单 topic 最多保留条数，超出丢最旧 |
| `ATTACHMENT_EXPIRY` | `3h` | 附件保留时长（写入消息的 `expires`，由 alarm 清理 R2） |
| `ATTACHMENT_MAX_SIZE` | `15728640`（15MB） | 单附件上限（字节） |
| `AUTH_TOKEN` | 空 | 留空=不鉴权；设置后需 Bearer/Basic |

绑定项（不是变量，不能删）：

| 绑定 | 类型 | 说明 |
| --- | --- | --- |
| `TOPIC` | Durable Object | 类名 `TopicDO`，`idFromName(topic)` 定位 |
| `ATTACHMENTS` | R2 bucket | 附件存储 |

---

## 12. API 兼容清单

| 端点 | 方法 | 说明 |
| --- | --- | --- |
| `/` | GET | 内置网页 |
| `/` | POST/PUT | JSON 发布，body 需含 `topic` |
| `/{topic}` | GET | 内置网页（预填 topic） |
| `/{topic}` | POST/PUT | 文本发布；`Filename:` 头→原始二进制上传；multipart→表单上传 |
| `/{topic}/json` | GET | NDJSON 流；`?poll=1` 一次性；`?since=` 补拉 |
| `/{topic}/sse` | GET | Server-Sent Events 流 |
| `/{topic}/raw` | GET | 纯文本流，每行一条消息正文 |
| `/{topic}/ws` | GET | WebSocket（Hibernation） |
| `/file/{id}.{ext}` | GET | 附件下载，带 `Content-Disposition` |
| `/v1/health` | GET | `{"healthy":true}` |
| `/v1/config` | GET | App 注册服务器时读取的配置 |

**发布请求头（大小写不敏感，含官方别名）**：

`Message`(`x-message`)、`Title`(`ti`/`t`)、`Priority`(`prio`/`p`，支持 `min/low/default/high/max/urgent` 或 1-5)、`Tags`(`tag`/`ta`，逗号分隔)、`Click`、`Actions`、`Attach`(远程 URL)、`Filename`、`Markdown`(`md`)、`Cache`(`x-cache`，`no` 表示不落库)、`Sequence-Id`。

**Actions 语法**（与官方一致，分号分隔多个动作）：

```
Actions: view, 打开详情, https://example.com/x; http, 我知道了, https://example.com/ack, method=POST, clear=true
```

**订阅查询参数**：`since`、`poll`、`id`、`message`、`title`、`priority`、`tags`。

---

## 13. 常用命令速查（curl / PowerShell）

PowerShell 里 `curl` 是 `Invoke-WebRequest` 的别名，**必须写 `curl.exe`** 才能用真 curl。

```powershell
$S = "https://ntfy.你的域名.com"
$T = "car-a8f3k2n9"

# 发一条
curl.exe -d "hello" $S/$T

# 高优先级 + 标题 + 标签
curl.exe -H "Title: 挪车通知" -H "Priority: 5" -H "Tags: car" -d "请挪车" $S/$T

# 实时看流（Ctrl+C 退出）
curl.exe -N $S/$T/json

# 拉最近 1 小时
curl.exe "$S/$T/json?poll=1&since=1h"

# 只要某一条
curl.exe "$S/$T/json?poll=1&id=xxxxxxxxxxxx"

# SSE
curl.exe -N $S/$T/sse

# 上传附件
curl.exe -F "file=@C:\temp\a.jpg" -F "message=照片" $S/$T

# 健康检查
curl.exe $S/v1/health
```

PowerShell 原生写法（不想依赖 curl）：

```powershell
Invoke-RestMethod -Method Post -Uri "$S/$T" -Headers @{ Title = "标题"; Priority = "4" } -Body "内容"
```

---

## 14. 费用与额度

| 项目 | 量级（免费计划，具体以 Cloudflare 官网为准） |
| --- | --- |
| Workers 请求 | 10 万次/天 |
| Durable Objects | 免费计划可用（**SQLite 后端**，本项目正是为此设计）；有每日读写行数额度 |
| DO 存储 | 每账号有免费额度，个人消息缓存 12h 很难接近 |
| R2 | 10GB 存储免费，**出网流量免费**，A 类/B 类操作有免费额度 |

个人挪车通知这种量级（一天几百条消息、附件少量）**基本永远在免费额度内**。

想再省：

- 把 `MESSAGE_CACHE_DURATION` 降到 `3h`
- 把 `ATTACHMENT_EXPIRY` 降到 `1h`
- 不用附件就把 `ATTACHMENT_MAX_SIZE` 调成 `0`（上传直接报 413）

---

## 15. 已知限制（务必先读）

1. **iOS 无后台推送**。没有 APNs 就没法在 App 被杀后唤醒。要彻底解决只有两条路：接 APNs（需要 Apple 开发者账号、证书、token 管理），或换回官方 ntfy.sh。
2. **`/{topic}/ws` 长连接有单条连接时长上限**。Hibernation 让连接在休眠时不计活跃时长，但 Cloudflare 仍可能周期性断开，客户端会自动重连（官方 App 就是这样设计的，无感）。
3. **SSE / NDJSON 流式订阅是「活着的 Worker」**。客户端挂着不断开时，Worker 会持续计时，成本高于 WS。手机端请用 App 默认的 WS 或轮询，别拿浏览器 SSE 长时间挂着。
4. **多 topic 合并订阅不支持**。`/a,b/json` 会返回 400。一个 App 里订阅两个 topic 就是两条连接，没问题。
5. **消息按 topic 分片，不跨 topic 查询**。没有全局「所有消息」视图。
6. **topic 名即密码**。不要用可猜的名字；对安全有更高要求就开 `AUTH_TOKEN`。
7. **无消息编辑/删除接口**（官方也没有对外暴露的删除 API，只能等过期）。

---

## 16. 排错手册

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 部署报 `Cannot find module ...` | `main` 路径或 import 写错 | 确认 `wrangler.toml` 的 `main = "src/index.js"`，文件名大小写一致 |
| 部署报 DO 迁移错误 | 用了 `new_classes` 但配了 SQLite | 必须 `new_sqlite_classes = ["TopicDO"]`，见 `wrangler.toml` |
| 报 `env.ATTACHMENTS is undefined` | R2 绑定缺失或桶名错 | 检查 `[[r2_buckets]]` 与 `bucket_name`，`npx wrangler r2 bucket list` 确认 |
| 上传附件 413 | 超过 `ATTACHMENT_MAX_SIZE` | 调大该变量，或压图 |
| 网页能发不能收 | 订阅请求打到了另一个 DO | 确认 topic 完全一致（含大小写；本项目按名字精确匹配） |
| 手机 App 注册服务器失败 | `/v1/config` 字段缺失或 `base_url` 不对 | `curl.exe $S/v1/config` 看返回；确认 `BASE_URL` 已设成 https 域名 |
| 手机收不到通知 | topic 拼错 / 手机端后台限制 / iOS 无 APNs | 先用网页订阅同一 topic 验证服务端；Android 关掉电池优化 |
| 消息过一会自己消失 | 正常行为，缓存到期 | 调大 `MESSAGE_CACHE_DURATION` |
| 404 `attachment not found` | 附件已过期被 alarm 清掉 | 正常，附件默认 3h |
| `npx wrangler tail` 看不到日志 | 没重新部署或过滤条件太窄 | `npx wrangler deploy` 后重试，`tail` 不加过滤先看全量 |
| 本地 dev 启动失败端口占用 | 8787 被占 | `npx wrangler dev --port 8788` |
| 本地 `wrangler dev` 发布 500、日志只有 `internal error` | 本机 miniflare 的 DO 环境问题，连最小 DO 样例也失败 | 跳过本地 dev：`npm test` 打桩自测 + 部署后 `scripts\smoke-test.mjs` 打线上 |
| 500 且日志报 `maxRows is not defined` | 1.0.0 早期 bug，已修复 | 确认 `src/topic-do.js` 里是 `const maxRows = Math.max(1, parseInt(this.env.MESSAGE_LIMIT \|\| "2048", 10) \|\| 2048);` |
| `Actions` 头写中文标签直接报错 | HTTP 头只能是 Latin-1 | 中文先做 URL 编码（`%E6%89%93%E5%BC%80`），或用 JSON 发布体传 `actions` 数组 |

排查顺序建议：**先 `npm test` 确认业务逻辑 → 再跑 `smoke-test.mjs` 看哪一项红 → 最后线上 `wrangler tail` 看实际请求**。

---

## 17. 后续可扩展的方向

按「性价比」排序，需要时可以一个个加：

1. **消息保留策略按 topic**：把 `MESSAGE_CACHE_DURATION` 改成「按 topic 前缀」的映射（`car-*` 保 7 天，其他 4h）。
2. **简单管理后台**：加一个 `/admin` 页 + `ADMIN_TOKEN`，能列出所有 topic、看条数、手动清空。
3. **一键诊断**：`/admin/diag` 返回 DO 状态、消息数、最近错误，复刻你挪车项目里那套「一键诊断快照」的做法。
4. **Web Push**：给浏览器加 `pushManager.subscribe` + VAPID，能省掉 WS 常驻成本。
5. **APNs 转发**：只对 `platform=ios` 的订阅者向 APNs 发一条静默通知 + 拉取，能真正解决 iOS 后台问题，但需要 Apple 开发者账号。
6. **接入 D1 做全局索引**：需要跨 topic 检索时，给每个发布同时写一条摘要进 D1。

---

## 18. 部署方式二：GitHub Actions 全自动部署

上面第 3~7 步是「本机装环境 + 手动 deploy」。如果不想在本机折腾，全部交给 GitHub Actions 即可。**完整的零基础点选教程见 [GitHub-Actions-部署教程.md](./GitHub-Actions-部署教程.md)**，这里只列技术要点。

### 18.1 新增的文件

| 文件 | 作用 |
|---|---|
| `.github/workflows/deploy.yml` | 流水线：`npm ci` → `npm test` → 建 R2 桶 → 注入配置 → `wrangler deploy` → 冒烟测试 → 写摘要 |
| `scripts/ci-config.mjs` | 把 GitHub 的 Variables/Secrets 写进 `wrangler.toml` 的 `[vars]`，并做格式校验（BASE_URL 必须带协议、时长必须形如 `12h`、上限必须纯数字） |

### 18.2 在 GitHub 里配什么

**Secrets**（Settings → Secrets and variables → Actions → Secrets）：

| 名称 | 必填 | 说明 |
|---|---|---|
| `CLOUDFLARE_API_TOKEN` | 是 | 自定义 Token，权限：`Account/Workers Scripts/Edit` + `Account/Workers R2 Storage/Edit` |
| `CLOUDFLARE_ACCOUNT_ID` | 是 | 控制台 32 位 Account ID |
| `NTFY_AUTH_TOKEN` | 否 | 访问密码；设了之后所有客户端必须带 `Authorization: Bearer xxx` |

**Variables**（同页面 Variables 标签，全部可选）：

| 名称 | 映射到 `[vars]` | 默认 |
|---|---|---|
| `NTFY_BASE_URL` | `BASE_URL` | 空（workflow 会自动回填 `*.workers.dev`） |
| `MESSAGE_CACHE_DURATION` | 同名 | `12h` |
| `ATTACHMENT_EXPIRY` | 同名 | `3h` |
| `ATTACHMENT_MAX_SIZE` | 同名 | `15728640` |
| `MESSAGE_LIMIT` | 同名 | `2048` |

### 18.3 流水线行为

1. **触发**：push 到 `main` 自动跑；也可在 Actions 页手动 `Run workflow`。
2. **并发**：`concurrency: deploy-ntfy` + `cancel-in-progress`，连点多次只会保留最新一次。
3. **代码门禁**：`npm test` 32 项离线自测不通过就不部署。
4. **建桶幂等**：`wrangler r2 bucket create ntfy-attachments` 已存在则跳过；只有权限类错误才报红。
5. **BASE_URL 自动回填**：没配 `NTFY_BASE_URL` 时，从 `deploy.log` 抓出 `https://xxx.workers.dev`，写回 toml 后**再部署一次**，保证 `/v1/config` 的 `base_url` 与附件链接正确。
6. **线上验收**：跑 `scripts/smoke-test.mjs`（10 项），失败即 workflow 失败。
7. **结果摘要**：写进 `$GITHUB_STEP_SUMMARY`，运行页顶部直接看到服务地址。

### 18.4 与手动部署的差异

| 场景 | 手动部署 | GitHub Actions |
|---|---|---|
| 改 `[vars]` | 改 toml → `wrangler deploy` | 改 Variables → Run workflow（toml 由 CI 注入，别在本地提交改过的值） |
| 部署凭据 | `wrangler login`（OAuth） | `CLOUDFLARE_API_TOKEN`（建议设 TTL，定期轮换） |
| 谁来验收 | 自己跑 `smoke` | 流水线自动跑，红叉即失败 |
| 本机依赖 | 要 Node + npm | 不需要，浏览器即可 |

> 注意：`ci-config.mjs` 会就地改写 `wrangler.toml`。如果本机也跑过它，`git status` 会看到 `wrangler.toml` 被修改——**不要把这个改动提交**（配置以 GitHub Variables 为准）。想避免误提交，可在本地用完 `git checkout -- wrangler.toml` 还原。

---

## 附：一页速通（懒人版）

```powershell
cd "...\output\ntfy-cloudflare"
npm install
npx wrangler login
npx wrangler r2 bucket create ntfy-attachments
npm test                             # 离线自测 32 项，必跑
npx wrangler deploy                  # 拿到 xxx.workers.dev 地址
# 控制台绑自定义域 ntfy.example.com → 改 wrangler.toml 的 BASE_URL → 再 deploy
node scripts\smoke-test.mjs https://ntfy.example.com
```

然后手机装 ntfy App，服务器填 `https://ntfy.example.com`，订阅一个随机 topic，收工。
