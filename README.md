# Coze（扣子）每日签到自动化

> 基于 GitHub Actions 的 Coze 每日积分自动领取工具，零本地运维：**每天自动点亮联系活跃天数、领取免费积分，失败时微信推送告警，Cookie 自动续期，配置一次全年无忧。**

Fork 自开源项目 [jzcangshu/coze-daily-credit-action](https://github.com/jzcangshu/coze-daily-credit-action)。

---

## 目录

- [项目简介](#项目简介)
- [功能特性](#功能特性)
- [设计原理](#设计原理)
- [技术栈](#技术栈)
- [项目结构](#项目结构)
- [快速开始（小白版）](#快速开始小白版)
  - [第 1 步：Fork 仓库](#第-1-步fork-仓库)
  - [第 2 步：导出 Coze Cookie](#第-2-步导出-coze-cookie)
  - [第 3 步：配置 Secrets](#第-3-步配置-secrets)
  - [第 4 步：启用 Actions](#第-4-步启用-actions)
  - [第 5 步：首次运行验证](#第-5-步首次运行验证)
- [进阶配置](#进阶配置)
  - [Cookie 自动续期（GH_PAT）](#cookie-自动续期gh_pat)
  - [失败微信通知（PUSHPLUS_TOKEN）](#失败微信通知pushplus_token)
  - [失败 iOS 通知（BARK_PUSH_URL，可选）](#失败-ios-通知bark_push_url可选)
- [日常使用](#日常使用)
- [常见问题 FAQ](#常见问题-faq)
- [注意事项](#注意事项)
- [开发说明](#开发说明)
- [更新日志](#更新日志)

---

## 项目简介

Coze（扣子，https://www.coze.cn/ ）要求**每天和它聊天至少一次**才能点亮「联系活跃天数」并领取每日免费积分。手动签到容易忘、难坚持。

本项目把签到任务搬到 **GitHub Actions 云端**：

- 不需要自己的电脑开机；
- 每天定时用保存的登录 Cookie 打开 Coze 页面，自动完成签到/领积分；
- Cookie 在签到过程中会被服务端刷新，脚本会把**最新的 Cookie 自动写回**仓库 Secret（自动续期），避免几天后失效；
- 签到**失败时通过 pushplus 推送微信消息**，让你第一时间知道。

## 功能特性

| 功能 | 说明 |
|------|------|
| ⏰ 定时自动签到 | 每天**北京时间约 05:30**（UTC 21:30）自动运行，云端执行，本地无需开机 |
| 🍪 Cookie 登录 | 使用浏览器导出的完整 Cookie（含 httpOnly 登录凭证）保持登录态 |
| 🔄 Cookie 自动续期 | 每次签到后把刷新的 Cookie 自动写回仓库 Secret，长期免维护 |
| 📱 失败微信通知 | 运行失败时通过 pushplus 推送微信消息，附仓库与运行记录链接 |
| 🍎 失败 iOS 通知（可选） | 支持 Bark 渠道，iPhone 用户可选用 |
| 🔒 最小权限令牌 | 自动续期使用细粒度 PAT，仅授权本仓库 + Secrets 读写权限 |
| 🖥️ 无界面运行 | 基于 Playwright 无头浏览器，模拟真实用户点击「领取/签到」按钮 |

## 设计原理

```
┌─────────────────────────────────────────────────────────┐
│                    GitHub Actions 云端                    │
│                                                          │
│  定时触发 (cron: 30 21 * * * UTC = 北京 05:30)            │
│        │                                                 │
│        ▼                                                 │
│  ┌──────────────┐    读取     ┌───────────────────────┐  │
│  │ Playwright   │◄───────────│ Secret:                │  │
│  │ 无头浏览器    │            │ COZE_COOKIES_JSON      │  │
│  └──────┬───────┘            └───────────────────────┘  │
│         │ 注入 Cookie 打开 coze.cn/home                  │
│         ▼                                                │
│  自动点击「签到 / 领取免费积分」按钮                         │
│         │                                                │
│         ├── 成功 → 结束 ✅                                │
│         │                                                │
│         ▼                                                │
│  保存刷新后的最新 Cookie                                   │
│         │                                                │
│         ▼                                                │
│  用 GH_PAT 调用 GitHub API ──► 回写 Secret                │
│  （"Updated repository secret COZE_COOKIES_JSON."）       │
│         │                                                │
│         ▼ 仅失败时                                        │
│  notify-failure.mjs ──► pushplus ──► 📱 微信推送告警       │
└─────────────────────────────────────────────────────────┘
```

**核心思路：**

1. **登录态即 Cookie**：不存密码，只保存浏览器登录后的 Cookie。Coze 的登录凭证主要在 `sessionid` 等 httpOnly Cookie 中，必须用能读取 httpOnly Cookie 的方式导出（浏览器插件或 CDP 协议）。
2. **云端模拟真人**：Playwright 打开真实页面并注入 Cookie，像人一样点按钮，比裸调 API 更稳、更不容易被风控。
3. **自愈式续期**：签到过程中 Coze 会滚动刷新会话，脚本抓住这个机会把新 Cookie 写回 Secret，形成「用一次、续一次」的闭环。只要每个月至少成功运行一次，Cookie 理论上不会过期。
4. **失败可观测**：成功时安静无声，失败时立刻微信告知，避免「悄悄断了半个月」才发现。

## 技术栈

| 组件 | 技术 | 用途 |
|------|------|------|
| 运行平台 | GitHub Actions | 云端定时任务，免费额度足够 |
| 浏览器自动化 | Playwright + Chromium（无头模式） | 注入 Cookie、模拟点击签到 |
| 运行时 | Node.js（ESM，`*.mjs`） | 签到与通知脚本 |
| 配置管理 | GitHub Encrypted Secrets | 安全存放 Cookie、令牌、推送 token |
| 自动续期 | GitHub REST API（细粒度 PAT） | 回写 `COZE_COOKIES_JSON` Secret |
| 消息推送 | pushplus（微信）/ Bark（iOS） | 失败告警 |
| 定时调度 | GitHub Actions `schedule`（cron） | 每天 UTC 21:30 触发 |

## 项目结构

```
coze-daily-credit-action/
├── .github/
│   └── workflows/
│       └── coze-daily.yml        # 定时工作流：签到 → 上传失败产物 → 失败通知
├── scripts/
│   ├── coze-daily.mjs            # 主脚本：读 Cookie → Playwright 打开 Coze → 签到
│   ├── update-github-secret.mjs  # 用 GH_PAT 回写最新 Cookie 到 Secret
│   ├── notify-failure.mjs        # 失败通知入口（组装消息后调 notifications）
│   └── notifications.mjs         # 推送实现：pushplus + Bark 双通道
├── package.json                  # 依赖声明（playwright 等）
├── package-lock.json
└── README.md                     # 本文档
```

### 工作流关键步骤（coze-daily.yml）

```yaml
- name: Notify on failure
  if: failure()                    # 仅失败时触发
  env:
    BARK_PUSH_URL:  ${{ secrets.BARK_PUSH_URL }}
    PUSHPLUS_TOKEN: ${{ secrets.PUSHPLUS_TOKEN }}
    PUSHPLUS_TOPIC: ${{ secrets.PUSHPLUS_TOPIC }}
    RUN_URL: ...               # 自动拼接本次运行记录链接
```

---

## 快速开始（小白版）

> 全程约 15 分钟，只需完成一次。

### 第 1 步：Fork 仓库

1. 注册/登录 [GitHub](https://github.com)；
2. 打开 [jzcangshu/coze-daily-credit-action](https://github.com/jzcangshu/coze-daily-credit-action)；
3. 点右上角 **Fork** → **Create fork**，得到你自己的仓库，例如 `你的用户名/coze-daily-credit-action`。

### 第 2 步：导出 Coze Cookie

**方式 A：Cookie-Editor 浏览器插件（推荐小白）**

1. 用 Chrome/Edge 打开 https://www.coze.cn/ 并登录；
2. 安装浏览器插件 **Cookie-Editor**（Chrome 应用商店搜索即可）；
3. 在 coze.cn 页面上点 Cookie-Editor 图标 → 右下角 **Export**（导出）→ 自动复制 JSON；
4. 把剪贴板内容粘贴到记事本备用（下一步要用）。

> ⚠️ 必须在**已登录**的 coze.cn 页面上导出。导出的内容是一长串 JSON 数组，包含 `sessionid` 等登录凭证，**不要泄露给任何人**。

**方式 B：开发者工具 / 浏览器自动化（进阶）**

Cookie-Editor 无法导出部分 httpOnly Cookie 时，可用 CDP（Chrome DevTools Protocol）的 `Storage.getCookies` 读取 coze.cn 的**全部** Cookie（含 httpOnly），整理为如下格式的 JSON 数组：

```json
[
  {
    "name": "sessionid",
    "value": "xxxxxxxx",
    "domain": ".coze.cn",
    "path": "/",
    "expires": 1790000000.123,
    "httpOnly": true,
    "secure": true,
    "sameSite": "Lax"
  }
]
```

### 第 3 步：配置 Secrets

1. 进入**你自己 Fork 的仓库**页面 → **Settings** → 左侧 **Secrets and variables** → **Actions**；
2. 点 **New repository secret**，逐个添加：

| Name（名称） | Secret（值） | 必填 |
|--------------|--------------|------|
| `COZE_COOKIES_JSON` | 第 2 步导出的完整 Cookie JSON 数组 | ✅ 必填 |
| `GH_PAT` | 细粒度个人令牌（用于 Cookie 自动续期，见下文） | ⭕ 推荐 |
| `PUSHPLUS_TOKEN` | pushplus 的 token（失败微信通知，见下文） | ⭕ 可选 |
| `BARK_PUSH_URL` | Bark 推送地址（失败 iOS 通知） | ⭕ 可选 |

> 💡 填写时：Name 精确一致（区分大小写），Value 粘贴时确保完整（Cookie JSON 有几 KB 长）。

### 第 4 步：启用 Actions

> ⚠️ **Fork 仓库的 Actions 默认是禁用的，且要启用两次**——这是最容易遗漏的一步！

1. **启用仓库 Actions**：进入你仓库的 **Actions** 标签页，会看到提示 "Workflows aren't being run on this forked repository"，点击绿色按钮 **I understand my workflows, go ahead and enable them**；
2. **启用定时工作流**：进入 Actions → 左侧选 **Coze daily credit** 工作流 → 点 **Enable workflow** 按钮（启用后才会按 cron 定时运行，并出现手动运行按钮）。

### 第 5 步：首次运行验证

1. 在工作流页面右侧点 **Run workflow** ▼ → **Run workflow** 确认按钮；
2. 等待 2~5 分钟（首次运行需安装依赖和浏览器内核）；
3. 运行结束后：
   - **绿色 ✓ Success** → 签到成功，配置完成！🎉
   - **红色 ✗ Failure** → 点进运行详情查看日志，常见原因见 [FAQ](#常见问题-faq)。

---

## 进阶配置

### Cookie 自动续期（GH_PAT）

不配置此项也能用，但 Cookie 过期后（通常数周）需要重新手动导出。配置后脚本每次签到会自动回写最新 Cookie，**一年内基本免维护**。

**创建细粒度令牌（Fine-grained personal access token）：**

1. 登录 GitHub → 右上角头像 → **Settings** → 左下角 **Developer settings** → **Personal access tokens** → **Fine-grained tokens** → **Generate new token**；
2. 按下表填写：

| 配置项 | 值 |
|--------|-----|
| Token name | `coze-daily-credit-refresh`（随意） |
| Expiration | Custom，建议设 1 年后 |
| Repository access | **Only select repositories** → 只勾选 `你的用户名/coze-daily-credit-action` |
| Repository permissions | **Secrets** → **Read and write**（Metadata 会自动附带 Read-only） |

3. 点 **Generate token**，**立即复制**生成的 `github_pat_` 开头的令牌（**只显示这一次**！）；
4. 回到仓库 Settings → Secrets → Actions，新建 Secret：Name 填 `GH_PAT`，Value 粘贴令牌。

**验证续期生效**：手动触发一次运行，成功后打开运行日志，`Update Coze cookie secret` 步骤出现：

```
Updated repository secret COZE_COOKIES_JSON.
```

即续期链路打通。

### 失败微信通知（PUSHPLUS_TOKEN）

1. 打开 [pushplus 官网](https://www.pushplus.plus/)，**微信扫码登录**；
2. 登录后首页即可看到你的 **token**（一串 32 位字符），复制；
3. ⚠️ **实名认证**：pushplus 现在要求实名认证后才能发消息（否则接口返回 code 905）。前往 https://verify.pushplus.plus ，填写姓名、身份证号、手机验证码并支付认证费（一次认证终生有效）；
4. 回到仓库 Settings → Secrets → Actions，新建 Secret：Name 填 `PUSHPLUS_TOKEN`，Value 粘贴 token；
5. **验证**：可先用 curl 测试 token 是否能推送到微信：

```bash
curl -X POST "https://www.pushplus.plus/send" \
  -H "Content-Type: application/json" \
  -d '{"token":"你的token","title":"测试","content":"Coze签到通知测试","template":"txt"}'
```

返回 `{"code":200,...}` 且微信收到消息即成功。

> 之后**签到失败时**，微信会收到「Coze每日积分失败」推送，内含仓库与运行记录链接。

### 失败 iOS 通知（BARK_PUSH_URL，可选）

iPhone 用户可改用/并用 [Bark](https://apps.apple.com/app/bark-customed-notifications/id1403753865)：

1. App Store 安装 Bark，打开后复制推送 URL（形如 `https://api.day.app/你的key`）；
2. 新建 Secret：Name 填 `BARK_PUSH_URL`，Value 粘贴该地址（末尾不要带 `/`）。

---

## 日常使用

配置完成后**无需任何操作**：

- **每天北京时间约 05:30** 自动签到（你电脑不用开机）；
- 查看**运行记录**：仓库 → **Actions** 标签页，每天一条，绿色 ✓ 表示成功；
- 查看**当前 Secret 列表**：Settings → Secrets and variables → Actions；
- 签到失败 → 微信收到推送 → 点运行记录链接 → 看日志排查。

**手动补签**：任何时候进入 Actions → Coze daily credit → Run workflow 手动触发即可。

## 常见问题 FAQ

**Q1：运行失败，日志提示 401 / 未登录 / Cookie 无效？**
Cookie 已过期或导出不完整。重新执行[第 2 步](#第-2-步导出-coze-cookie)导出，然后到 Secrets 里**更新** `COZE_COOKIES_JSON`（点该 Secret 的铅笔 ✏️ 图标更新，不要新建重名）。配置了 GH_PAT 自动续期的话，这种情况很少出现。

**Q2：Fork 后 Actions 页面是空的 / 定时任务不跑？**
Fork 仓库默认禁用 Actions，且要**启用两次**：①Actions 页点 "I understand... enable them"；②工作流详情页点 "Enable workflow"。详见[第 4 步](#第-4-步启用-actions)。

**Q3：pushplus 推送报 code 905「未实名认证」？**
pushplus 平台要求实名认证后才能发消息。到 https://verify.pushplus.plus 完成认证（姓名+身份证+手机号，一次性支付认证费）。

**Q4：GitHub 令牌（GH_PAT）过期了怎么办？**
微信收不到续期相关错误时检查 Actions 日志。令牌到期后重新创建一个（同[上文步骤](#cookie-自动续期gh_pat)），更新 `GH_PAT` Secret 即可。

**Q5：Secret 名字填错了 / 值贴漏了？**
Secret 名字**区分大小写**且必须精确为 `COZE_COOKIES_JSON` 等；Cookie JSON 很长，粘贴后建议核对首尾字符完整。

**Q6：一天能签到多次吗？**
没必要。活跃天数按天计算，每天成功一次即可。

**Q7：GitHub Actions 收费吗？**
公共仓库 Actions 完全免费；Fork 出来的仓库默认是公共的，每月 2000 分钟免费额度对每天一次、每次几分钟的任务绰绰有余。

**Q8：换电脑 / 重装浏览器后要重新配置吗？**
不需要。Cookie 和令牌都存在 GitHub 云端，与本地设备无关。

## 注意事项

- 🔐 **Cookie = 账号钥匙**：`COZE_COOKIES_JSON` 等同于登录态，绝不要粘贴给他人、不要提交到代码里，只放在 GitHub Encrypted Secrets 中；
- 🕐 **定时时间可能有延迟**：GitHub cron 在整点高峰期可能延迟几分钟到几十分钟，属正常现象，不影响签到；
- 🍪 **长期不用会失效**：如果账号长期在别处退出登录、或 Coze 更换会话机制，Cookie 可能失效，重新导出即可；
- ⚖️ **合规使用**：仅供个人学习与自动化个人签到，请勿用于刷量等违规用途；Coze 风控策略变化可能导致脚本需要更新；
- 🚫 **不要把 Secret 提交进 git**：任何含 Cookie/token 的文件不要 push 到仓库（本项目的 Secret 均通过加密变量注入，代码中不含任何凭证）。

## 开发说明

### 本地运行

```bash
# 1. 克隆仓库
git clone https://github.com/你的用户名/coze-daily-credit-action.git
cd coze-daily-credit-action

# 2. 安装依赖（含 Playwright）
npm install
npx playwright install chromium

# 3. 设置环境变量后运行签到脚本
export COZE_COOKIES_JSON='[{"name":"sessionid","value":"...","domain":".coze.cn", ...}]'
export GH_PAT='github_pat_xxx'          # 可选：启用自动回写
export PUSHPLUS_TOKEN='你的token'        # 可选：启用失败通知
node scripts/coze-daily.mjs
```

### 各脚本职责

| 文件 | 职责 |
|------|------|
| `scripts/coze-daily.mjs` | 解析 `COZE_COOKIES_JSON` → 每次使用**新的 Playwright context**（避免状态残留冲突）→ 打开 `https://www.coze.cn/home` → 自动点击签到/领取按钮 → 输出刷新后的 Cookie |
| `scripts/update-github-secret.mjs` | 用 `GH_PAT` 调 GitHub REST API，把最新 Cookie 加密回写到 `COZE_COOKIES_JSON` Secret |
| `scripts/notify-failure.mjs` | 组装失败消息（标题、仓库、运行记录 URL），调用通知模块 |
| `scripts/notifications.mjs` | `sendPushPlus()`：POST `https://www.pushplus.plus/send`（支持 `PUSHPLUS_TOPIC` 群组、`PUSHPLUS_TEMPLATE` 模板）；`sendBark()`：GET Bark URL 推送；两者并行，任一成功即视为通知成功 |

### 环境变量一览

| 变量 | 作用 | 来源 Secret |
|------|------|-------------|
| `COZE_COOKIES_JSON` | Coze 登录 Cookie 数组 | 同名 |
| `GH_PAT` | 回写 Secret 的细粒度令牌 | 同名 |
| `PUSHPLUS_TOKEN` | pushplus 推送令牌 | 同名 |
| `PUSHPLUS_TOPIC` | pushplus 群组编码（可选） | 同名 |
| `PUSHPLUS_TEMPLATE` | 消息模板，默认 `txt` | 同名 |
| `BARK_PUSH_URL` | Bark 推送基础地址 | 同名 |
| `RUN_URL` / `REPOSITORY` | 失败消息中的运行链接与仓库名 | 工作流自动注入 |

### 修改定时时间

编辑 `.github/workflows/coze-daily.yml` 中的 cron（注意 GitHub cron 使用 **UTC 时间**，北京时间 = UTC + 8）：

```yaml
schedule:
  - cron: '30 21 * * *'   # UTC 21:30 = 北京时间 05:30
```

---

## 更新日志

### v1.2（2026-09-25）
- ✨ 新增失败微信通知（pushplus）：适配 pushplus 实名认证要求（未认证时接口返回 code 905）
- ✨ 新增失败 iOS 通知（Bark）双通道支持

### v1.1（2026-09-25）
- ✨ 新增 Cookie 自动续期：签到后用细粒度 PAT（`GH_PAT`，仅授权本仓库 + Secrets 读写）自动回写最新 Cookie，日志输出 `Updated repository secret COZE_COOKIES_JSON.`
- 🐛 修复 Playwright context 状态残留问题：每次运行使用全新 context

### v1.0（2026-09-25）
- 🎉 首次发布：GitHub Actions 定时签到，Cookie 登录，手动触发验证通过
- 📋 完成 Fork → Cookie 导出 → Secret 配置 → 启用 Actions → 首次运行全链路

---

## 致谢

- 上游项目：[jzcangshu/coze-daily-credit-action](https://github.com/jzcangshu/coze-daily-credit-action)
- 消息推送：[pushplus 推送加](https://www.pushplus.plus/)、[Bark](https://bark.day.app/)

> 如果本项目帮到了你，欢迎点个 ⭐ Star！
