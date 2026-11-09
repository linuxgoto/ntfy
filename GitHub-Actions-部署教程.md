---
AIGC:
    Label: "1"
    ContentProducer: 001191440300708461136T1XGW3
    ProduceID: 62af1c42423ad350b2d4da6d04c5c743_e7b07010dc4411f1bff2525400cd780f
    ReservedCode1: 8h4ROFc7bWuBWSuX2rzOcITVKqymW9/mlfICUbfnAgMsPYesGkYudUr/kk2scULYy3l5xC8HYD7tsv1p91WpCPhrT7266JcGgVjphl9XBjXDQokTfJQpa8vj+pX4+IXZBjEc5kUID1LTBT1LLBUQEOqciy3PmiPNruONQOCoPumgzB6qyoO5fiHO70g=
    ContentPropagator: 001191440300708461136T1XGW3
    PropagateID: 62af1c42423ad350b2d4da6d04c5c743_e7b07010dc4411f1bff2525400cd780f
    ReservedCode2: 8h4ROFc7bWuBWSuX2rzOcITVKqymW9/mlfICUbfnAgMsPYesGkYudUr/kk2scULYy3l5xC8HYD7tsv1p91WpCPhrT7266JcGgVjphl9XBjXDQokTfJQpa8vj+pX4+IXZBjEc5kUID1LTBT1LLBUQEOqciy3PmiPNruONQOCoPumgzB6qyoO5fiHO70g=
---

# 用 GitHub Actions 部署（零命令行版）

> 目标：**代码传上 GitHub，填几个密钥，点一次 Run，服务就上线能用。**
> 以后改配置、改代码都在网页上点，不用在本机装 Node、不用碰 PowerShell。
> 全程约 15 分钟。看不懂的名词查文末《附录 C》。

---

## 它是怎么跑起来的

```
你在网页上点 Run workflow
        ↓
GitHub 开一台临时 Linux 机器（跑完就销毁）
        ↓
自动执行：npm ci → npm test（32 项自测）→ 建 R2 桶 → 部署 → 冒烟测试
        ↓
Cloudflare 上出现你的 ntfy 服务（Worker + Durable Object + R2）
        ↓
手机 ntfy App 填这个地址就能收推送
```

**关键点**：Cloudflare 的账号权限不放代码里，而是存在 GitHub 仓库的 **Secrets**（加密，任何人都看不到，包括你自己回看时）里，Actions 运行时临时注入。所以仓库即使公开，密钥也不会泄露。

---

## 你要准备的东西

| 需要 | 是否必须 | 说明 |
|---|---|---|
| GitHub 账号 | 必须 | 免费注册，https://github.com/signup |
| Cloudflare 账号 | 必须 | 已有就用，没有去 https://dash.cloudflare.com/sign-up |
| Cloudflare 已开通 R2 | 必须 | 控制台点一次「启用」，需要绑卡验证，免费额度内不扣费 |
| 自定义域名（托管在 Cloudflare） | 可选 | 有则手机推送更稳；没有也能用 `xxx.workers.dev` |
| 本机装 Git | 可选 | 用「网页上传」方式时不需要 |

---

## 第 1 步：建一个 GitHub 仓库

**要做什么**：给这套代码建个"网盘 + 自动化流水线"的房子。

**为什么**：Actions 只能跑在仓库里；仓库同时充当代码备份。

**怎么做**：

1. 登录 GitHub，右上角 **+** → **New repository**
2. 填写：
   - **Repository name**：`ntfy-cloudflare`（随意，英文）
   - **Description**：随便写或留空
   - 可见性选 **Private**（推荐；私有仓库每月有 2000 分钟 Actions 免费额度，本项目每次跑约 1~2 分钟，够用几百次）
   - **不要**勾 "Add a README file"、**不要**勾 .gitignore 和 license（勾了会和要上传的文件冲突）
3. 点 **Create repository**

**你应该看到**：一个空仓库页面，显示 "Quick setup" 和几条 git 命令。

---

## 第 2 步：把项目文件传上去

项目在这台电脑上：

```
C:\Users\Administrator\AppData\Roaming\Tencent\Marvis\User\oAN1i2XA3P-syKMIdXSbSD10Zjn8\workspace\conv_6584d7454c524546be01ccbced9248ad\output\ntfy-cloudflare
```

下面三种方式**选一种**。

### 方式 A：网页拖拽上传（不装 Git，最简单）

⚠️ 关键：`node_modules` 文件夹**不能传**（几百 MB，网页上传会卡死或失败）。`.gitignore` 在网页上传时不起作用，必须手动避开。

1. 打开仓库页面 → **uploading an existing file** 链接（或 **Add file** → **Upload files**）
2. 打开资源管理器进入项目目录，**只选这些**拖进浏览器：
   - `src`（文件夹）
   - `scripts`（文件夹）
   - `test`（文件夹）
   - `.github`（文件夹，里面就是自动部署的 workflow）
   - `wrangler.toml`
   - `package.json`
   - `package-lock.json`
   - `.gitignore`
   - `README.md`、`小白操作手册.md`、`GitHub-Actions-部署教程.md`（这几份文档可选，但建议传）
3. 页面底部 Commit message 随便写，如 `init`，点 **Commit changes**

> `.github` 是隐藏文件夹（名字以点开头）。在资源管理器「查看」里勾上「隐藏的项目」才看得到。
> 如果网页不让你拖 `.github`，就在 GitHub 上手动建：**Add file** → **Create new file** → 文件名输入 `.github/workflows/deploy.yml`（输入 `/` 会自动变成目录）→ 把本机该文件的内容整段粘进去 → Commit。

### 方式 B：命令行（推荐，`.gitignore` 自动生效）

先装 Git（https://git-scm.com/download/win ，一路下一步）。然后在项目目录打开 PowerShell（方法见《小白操作手册.md》第 2 步），依次执行：

```powershell
git init
git add .
git commit -m "init: ntfy on cloudflare"
git branch -M main
git remote add origin https://github.com/你的用户名/ntfy-cloudflare.git
git push -u origin main
```

第一次 push 会弹浏览器要求登录 GitHub → 点授权即可。

> `.gitignore` 已经排除了 `node_modules/` 和 `.wrangler/`，所以 `git add .` 不会把依赖传上去。

### 方式 C：GitHub Desktop（图形界面）

安装 https://desktop.github.com → File → Add local repository → 选项目目录 → Publish repository（取消勾选 "Keep this code private" 保持私有）→ Publish。

**验证上传成功**：仓库首页能看到 `src`、`wrangler.toml`，并且 **Actions** 标签里已经出现一条运行记录（因为 push 触发了部署，但此刻还没密钥，所以会失败——正常，下一步去配密钥）。

---

## 第 3 步：创建 Cloudflare API Token

**要做什么**：拿到一把只够部署用的钥匙。

**为什么**：Actions 跑在 GitHub 的机器上，要让 Cloudflare 认可"这是你账号授权的操作"，就得用 API Token。**不要用全局 API Key**，权限太大。

**怎么做**：

1. 打开 https://dash.cloudflare.com/profile/api-tokens
2. 点 **Create Token**
3. 拉到最下面 **Custom token** → **Get started**
4. **Token name**：`github-actions-ntfy`
5. **Permissions** 加两行（点 `+ Add more`）：

   | 第一列 | 第二列 | 第三列 |
   |---|---|---|
   | Account | Workers Scripts | **Edit** |
   | Account | Workers R2 Storage | **Edit** |

6. **Account Resources**：`Include` → 选你的账号
7. **TTL**：留 `No expiration`（或设个 1 年，到期再换）
8. 点 **Continue to summary** → **Create Token**
9. 页面显示一串 token（形如 `AbCdEf...` 40 位），**立刻复制**——这串只显示一次，关掉就再也看不到（只能重建一个）

**出问题怎么办**：

| 现象 | 解决 |
|---|---|
| 没有 "Custom token" | 界面改版了，找 **Create Custom Token** 字样；或选模板 **Edit Cloudflare Workers** 再手动加 R2 权限 |
| 保存时提示需要验证邮箱 | 去邮箱点验证链接 |
| Token 忘了复制 | 回列表页 Delete 掉重建一个，不影响已有部署 |

---

## 第 4 步：找到你的 Account ID

**要做什么**：复制 32 位账号 ID。

**为什么**：一个 Cloudflare 账号下可能有多个"账户"（Account），部署时要指名道姓说是哪一个。

**怎么做**（三选一）：

- 登录 https://dash.cloudflare.com 后看浏览器地址栏：`dash.cloudflare.com/`**`<32位十六进制>`**`/...`，那就是 Account ID
- 或打开 **Workers & Pages** 页面，右侧栏有 **Account ID**，点复制图标
- 或命令行 `npx wrangler whoami` 输出里的 Account ID

形如：`a1b2c3d4e5f60718293a4b5c6d7e8f90`

---

## 第 5 步：把密钥填进 GitHub

**要做什么**：把 Token 和 Account ID 存成仓库的 Secrets。

**为什么**：workflow 里用 `${{ secrets.CLOUDFLARE_API_TOKEN }}` 读取；放在 Secrets 里加密存储，日志中会自动打码成 `***`。

**怎么做**：

1. 进你的仓库 → **Settings**（顶部标签栏）
2. 左侧 **Secrets and variables** → **Actions**
3. 停留在 **Secrets** 标签 → 右上 **New repository secret**，逐条添加：

   | Name（必须一字不差） | Secret 值 | 必须？ |
   |---|---|---|
   | `CLOUDFLARE_API_TOKEN` | 第 3 步复制的那串 | ✅ 必须 |
   | `CLOUDFLARE_ACCOUNT_ID` | 第 4 步那 32 位 | ✅ 必须 |
   | `NTFY_AUTH_TOKEN` | 你想设的访问密码（长随机串）。留空/不建 = 不鉴权，topic 名就是密码 | 可选 |

4. 每条点 **Add secret**

**常见坑**：

- 名字必须**全大写、下划线**，写错一个字读数就是空，deploy 会报 `Authentication error` 或 `No account selected`
- 值首尾**不要带空格**（复制时容易带上）

---

## 第 6 步：填写运行参数（Variables）

**要做什么**：把"你要部署到哪个地址、消息留多久"这类非敏感配置填在 Variables 里。

**为什么**：这些值不是密码，但每个人不一样，写死在代码里不利于维护；Variables 改了只需要再点一次 Run，不用改代码。

**怎么做**：同页面（Settings → Secrets and variables → Actions）→ 切到 **Variables** 标签 → **New repository variable**。

| Name | 值示例 | 必须？ | 说明 |
|---|---|---|---|
| `NTFY_BASE_URL` | `https://ntfy.你的域名.com` | 有自定义域名时**必须** | 想用 `xxx.workers.dev` 就**留空不建**，workflow 会自动回填 |
| `MESSAGE_CACHE_DURATION` | `12h` | 可选 | 消息保留时长，`30m`/`12h`/`7d` |
| `ATTACHMENT_EXPIRY` | `3h` | 可选 | 附件保留时长 |
| `ATTACHMENT_MAX_SIZE` | `15728640` | 可选 | 附件上限（字节，15MB） |
| `MESSAGE_LIMIT` | `2048` | 可选 | 单 topic 最多存多少条 |

**注意**：`NTFY_BASE_URL` 一旦填了，就必须是**已经能访问**的域名（末尾不要带 `/`）；域名还没绑就先不填，等第 9 步绑好再回来补。

---

## 第 7 步：点一次 Run，让它部署

**要做什么**：手动触发流水线。

**为什么**：刚才上传代码时已经自动跑过一次，但那时还没密钥，是失败的；现在密钥齐了，重新跑一次就成功。

**怎么做**：

1. 仓库顶部 → **Actions** 标签
2. 左侧列表点 **Deploy ntfy to Cloudflare**
3. 右侧 **Run workflow** 下拉 → 分支选 `main` → 绿色 **Run workflow**
4. 刷新页面，出现一条黄色（运行中）的记录，点进去

---

## 第 8 步：看懂日志（每一步在干什么）

点进运行记录后，左侧是步骤列表，**我们的 11 步各干什么**：

| 步骤名 | 它在干什么 | 失败了说明什么 |
|---|---|---|
| 检出代码 | 把仓库代码下载到临时机器 | 仓库是空的/权限问题 |
| 安装 Node 22 | 装运行环境 | 极少失败 |
| 安装依赖 | `npm ci` 装 wrangler | `package-lock.json` 与 `package.json` 不一致 |
| 检查密钥是否配齐 | 缺 Token/Account ID 直接报错 | 回第 5 步，名字写错了 |
| 离线自测（32 项） | 在离线环境跑全部业务逻辑 | 代码被改坏了，日志里会有 FAIL |
| 创建 R2 桶 | 桶不存在就建，存在就跳过 | Token 缺 R2 权限，或账号没开通 R2 |
| 注入配置 | 把 Variables/Secrets 写进 `wrangler.toml` | 变量格式不对（例如 BASE_URL 忘了写 https://） |
| 部署到 Cloudflare | 真正上传并发布 | 权限不足 / DO 迁移冲突 |
| 未设自定义域名时回填 BASE_URL | 没给域名就自动用 workers.dev 地址再部署一次 | 日志里没解析到地址 |
| 线上冒烟测试（10 项接口） | 打真实接口验发布/拉取/附件/长连接 | 看哪一项红，对《小白操作手册》附录 B |
| 写入部署摘要 | 在页面顶部生成一张结果卡片 | 不影响部署 |

**你应该看到**：全部步骤绿色对勾，页面顶部 **Summary** 区域出现：

```
## ntfy 部署结果
状态：✅ 成功
- 服务地址：`https://ntfy.xxx.workers.dev`
- 健康检查：`.../v1/health`
- 网页调试：`.../`
- 手机 App 服务器地址填：`...`
```

**这个地址就是你的服务器地址，记下来。**

---

## 第 9 步：绑定自己的域名（可选但强烈建议）

**为什么**：`workers.dev` 在国内部分网络下不稳定，手机长连接容易断；自定义域名稳得多。

**怎么做**：

1. Cloudflare 控制台 → **Workers & Pages** → 点 **ntfy-edge**
2. **Settings** → **Domains & Routes** → **Add** → **Custom domain**
3. 填 `ntfy.你的域名.com` → **Add domain** → 等状态变 **Active**（1~3 分钟）

**然后必须回来做这一步**（否则 App 里注册的地址还是 workers.dev）：

1. GitHub 仓库 → **Settings** → **Secrets and variables** → **Actions** → **Variables**
2. 新增 `NTFY_BASE_URL` = `https://ntfy.你的域名.com`
3. 回到 **Actions** → **Deploy ntfy to Cloudflare** → **Run workflow** 再跑一次

**验证**：跑完后浏览器打开 `https://ntfy.你的域名.com/v1/config`，应看到 `"base_url":"https://ntfy.你的域名.com"`。

---

## 第 10 步：确认真的能用

三种验法，任选：

1. **浏览器**：打开 `https://你的地址/v1/health` → 显示 `{"healthy":true}`
2. **调试网页**：打开 `https://你的地址/` → 有内置的发送/接收界面，直接点着玩
3. **发一条再拉回来**（不用装任何东西）：浏览器打开
   `https://你的地址/你的topic/json?poll=1` 能看到历史消息

> 想用命令行发消息见《小白操作手册.md》第 10 步；手机 App 接入见第 14 步。

---

## 第 11 步：以后的日常操作

| 我想做的事 | 怎么做 |
|---|---|
| 改了代码（`src/`）想上线 | push 到 `main` 分支，自动部署，不用点任何按钮 |
| 改配置（缓存时长、附件上限） | 改 Variables → Actions → Run workflow |
| 设/改访问密码 | 改 Secret `NTFY_AUTH_TOKEN` → Run workflow（注意：设了之后所有客户端都要带 token） |
| 换 Cloudflare 账号 | 换两个 Secret → Run workflow |
| 看线上日志 | 本机 `npx wrangler tail`；或在 Cloudflare 控制台 Worker 页面看 Logs（本 workflow 已开 observability） |
| 临时关掉自动部署 | 仓库 Settings → Actions → 关掉 workflow，或改 `.github/workflows/deploy.yml` 里的 `on:` |
| 回滚到上一版 | Cloudflare 控制台 → Worker → Deployments → 找到旧版本 → **Rollback** |

**注意**：只要 `wrangler.toml` 被 `ci-config.mjs` 改过再提交，本机文件里的 `BASE_URL` 会变；这不影响 Actions（每次运行都重新注入），但如果本机也手动部署，记得以 Variables 为准。

---

## 常见报错对照表

| 报错 / 现象 | 原因 | 解决 |
|---|---|---|
| Actions 标签里左边没有 workflow | 文件路径不对，或不在默认分支 | 必须是 `.github/workflows/deploy.yml`（注意开头的点），且分支是 `main` |
| `缺少 Secret CLOUDFLARE_API_TOKEN` | 名字拼错 / 建在 Variables 里了 | 回第 5 步，注意是 **Secrets** 标签不是 Variables |
| `Authentication error [code: 10000]` | Token 无效或权限不够 | 重新建 Token，确认两行 Edit 权限 + Account Resources 选了账号 |
| `No account selected` / `Could not determine account` | 缺 Account ID 或 ID 错 | 回第 4 步复制完整 32 位 |
| `创建 R2 桶失败` | 账号没开通 R2 / Token 缺 R2 权限 | 控制台先启用 R2；Token 加 `Workers R2 Storage:Edit` |
| `bucket already exists` 一类的红 | 极少数情况下返回码不是 0 且文案不同 | 手动删掉桶重建，或忽略——桶存在本就能部署 |
| `npm ci` 报 lock 文件不一致 | 有人改了 package.json 没更新 lock | 本机跑 `npm install` 后提交新的 `package-lock.json` |
| `npm ci` 报 `EBADENGINE` | 本地 Node 与 22 差异 | 我们已固定 runner 用 Node 22，一般不会出现 |
| 冒烟测试某一项红 | 线上真实问题 | 看红的那一项，对《小白操作手册》第 13 步/附录 B |
| `Workflow ... is disabled` / 找不到 Run workflow 按钮 | 仓库很久没活动，Actions 被自动停用 | 点页面上的 **Enable workflow** 按钮，或随便提交一次代码激活 |
| 运行被标成 `cancelled` | 你连续点了几次 Run，并发规则自动取消旧的 | 正常现象，等最新那次跑完 |
| `This workflow has no runs yet` 但已 push | 只 push 了非 main 分支 | push 到 `main`，或手动 Run |
| 私有仓库提示 Actions 分钟数耗尽 | 免费额度 2000 分钟/月用完了 | 等下月重置；或把仓库设为公开（代码公开=额度不限） |
| 部署成功但 `config` 里 base_url 是 workers.dev | 绑了自定义域名但没设 `NTFY_BASE_URL` | 回第 9 步后半段，加 Variable 再 Run |

---

## 附录 A：这个 workflow 每段在干什么（技术向）

| 片段 | 作用 |
|---|---|
| `on: push: branches: [main]` + `workflow_dispatch` | 推代码自动部署；也支持网页手动点 |
| `concurrency` + `cancel-in-progress` | 同一时刻只允许一个部署，避免两版交错发布 |
| `env` 里 `CLOUDFLARE_API_TOKEN` 等 | 把 Secrets/Variables 变成环境变量，供 wrangler 与注入脚本读取 |
| `npm ci` | 按 lock 文件精确安装，比 `npm install` 更适合 CI |
| `npm test` | 32 项离线自测，代码门禁：测不过就不部署 |
| `wrangler r2 bucket create` + 错误分支判断 | 幂等建桶；已存在就跳过，权限错才报红 |
| `node scripts/ci-config.mjs` | 把 Variables/Secrets 写进 `wrangler.toml` 的 `[vars]`，并对格式做校验 |
| `wrangler deploy` + `tee deploy.log` | 部署并落日志，再用正则从日志抓出 `*.workers.dev` 地址 |
| 回填 BASE_URL 再部署 | 没配域名时，让服务自己知道对外地址（附件链接/App 注册都靠它） |
| `smoke-test.mjs` | 打线上真实接口的 10 项端到端校验，失败即红 |
| `$GITHUB_STEP_SUMMARY` | 在运行页面顶部生成中文结果卡片 |

---

## 附录 B：不想用 Git / 不想用网页拖文件

最省事的替代：**在 GitHub 里直接建仓库后，用 GitHub 的 "Import" 或把 zip 解压后逐目录上传**。原则上只要保证仓库根目录结构与项目一致（`wrangler.toml` 在根、`src/` 在根、`.github/workflows/deploy.yml` 在根），Actions 就能跑。

---

## 附录 C：名词解释

| 名词 | 一句话解释 |
|---|---|
| GitHub Actions | GitHub 提供的免费自动化流水线，能在云端跑命令 |
| Workflow | 一条流水线，就是一个 yml 文件，放在 `.github/workflows/` 下 |
| Secret | 加密存储的密钥，日志里自动打码，建好后连你自己也看不到原值 |
| Variable | 明文存储的配置项，建好后可以随时查看和修改 |
| API Token | Cloudflare 的限权钥匙，只给它该有的权限 |
| Account ID | 你的 Cloudflare 账户 32 位标识 |
| `main` 分支 | 主分支，我们的 workflow 只对它触发 |

---

## 一页速通（配好密钥之后）

```
Settings → Secrets and variables → Actions
   Secrets:   CLOUDFLARE_API_TOKEN / CLOUDFLARE_ACCOUNT_ID (/ NTFY_AUTH_TOKEN)
   Variables: NTFY_BASE_URL（有域名就填）
Actions → Deploy ntfy to Cloudflare → Run workflow
→ 等 1~2 分钟 → 看 Summary 里的服务地址 → 手机 App 填这个地址
```
*（内容由AI生成，仅供参考）*
