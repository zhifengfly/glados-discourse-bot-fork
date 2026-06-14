# GLaDOS 签到 & Discourse 多站自动阅读 Bot ☁️

Telegram Bot，自动签到 GLaDOS，静默阅读 NodeLoc / NodeSeek / LinuxDO Discourse 论坛攒升级经验。

## 功能

- ✅ **GLaDOS 多账号签到** — 支持任意数量的 GLaDOS 类账号
- ✅ **Discourse 多站自动阅读** — NodeLoc / NodeSeek / LinuxDO
- ✅ **Telegram 管理** — 绑定/解绑账号，查看统计（帖数/时长/Cookie状态），实时通知
- ✅ **定时执行** — Cloudflare Workers Cron 每小时触发一次，每次读 5 帖
- ✅ **行为仿真** — 随机阅读时长 + 随机休息，避免风控

## 部署

### 1. 用 Deploy with Workers 按钮（推荐）

[![Deploy to Cloudflare Workers](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/Linsars/glados-discourse-bot)

点击后授权 GitHub 和 Cloudflare，填入 `BOT_TOKEN` 和 `ADMIN_ID` 即可。

### 2. 手动部署

```bash
# 1. 克隆仓库
git clone https://github.com/Linsars/glados-discourse-bot.git
cd glados-discourse-bot

# 2. 安装 wrangler
npm install -g wrangler

# 3. 创建 KV namespace
wrangler kv:namespace create GLADOS_DB

# 4. 配置 wrangler.toml
#    将 kv_namespaces.id 改成上一步创建的 ID
#    在 Cloudflare Dashboard 添加 secret:
#    - BOT_TOKEN: Telegram Bot Token
#    - ADMIN_ID: 你的 Telegram User ID

# 5. 部署
wrangler deploy
```

### 3. 激活 Bot

部署后访问 Worker 域名（例如 `https://glados-bot.xxx.workers.dev/`），会自动激活 Webhook。

如果显示 `{"webhook":"✅ 已激活","commands":"✅ 已注册"}` 则说明激活成功。

## 绑定账号

在 Telegram 中与 Bot 对话，通过菜单操作：

### Discourse 论坛（NodeLoc / NodeSeek / LinuxDO）
点「添加账号」→ 选择对应站点 → **发送 Cookie 即可**（格式：`_forum_session=xxx; _t=yyy`）

Bot 自动调 Discourse API 提取用户名和邮箱，无需手动取名。

### GLaDOS 账号
点「绑定账号」→ 发送 GLaDOS Cookie 即可（格式：`connect.sid=xxx; ...`）

Bot 自动调 GLaDOS API 提取邮箱和对应域名，无需指定。

## 获取 Cookie（Surge / Loon / QX / Egern 模块）

**安装链接（点击即可添加）：**
```
https://raw.githubusercontent.com/Linsars/Surge/main/sg/glados.yaml
```

> 💡 本模块为 Surge 模块格式，在 **Egern** 中亦可使用（用法相同）。

**支持的站点：**
- `glados.network` / `glados.rocks` 等 GLaDOS 变体
- `www.nodeloc.com`（NodeLoc）
- `nodeseek.cc`（NodeSeek）
- `linux.do`（Linux DO）

安装后，访问对应站点的 **头像 → 设置 → 账户** 页面，模块会自动捕获 Cookie。

### 手动获取 Cookie

所有 Discourse 站点步骤相同：

1. 登录网站
2. 浏览器 F12 → Application → Cookies
3. 复制 `_forum_session` 和 `_t` 的值
4. 发送给 Bot

## 升级条件参考

### NodeLoc [会员等级系统](https://www.nodeloc.com/t/topic/47018)

| 等级 | 关键条件 | Bot 能跑的 |
|------|---------|-----------|
| 🥉 TL1 白银 | 600 分钟阅读，100 篇帖子 | ✅ 全部 |
| 🥈 TL2 黄金 | 3000 分钟，30 天，赞/回复 | ✅ 阅读 + 访问 |
| 🥇 TL3 钻石 | 100 天考察期，赞/回复 | ✅ 阅读 + 访问 |
| 👑 TL4 王者 | 满足 TL3 + 申请投票 | ❌ 手动 |

### NodeSeek [信任等级说明](https://nodeseek.cc/n/topic/283)

| 等级 | 关键条件 | Bot 能跑的 |
|------|---------|-----------|
| 🥉 TL1 基础 | 30 篇帖子，10 分钟 | ✅ 全部 |
| 🥈 TL2 成员 | 100 篇，60 分钟，15 天 | ✅ 阅读 + 访问 |
| 🥇 TL3 常规 | 200 话题 + 500 帖累计，100 天考察 | ✅ 阅读 + 访问 |
| 👑 TL4 领袖 | 手动授予 | ❌ |

### LinuxDO [Discourse 信任度](https://linux.do/t/topic/2460)

| 等级 | 关键条件 | Bot 能跑的 |
|------|---------|-----------|
| 🥉 TL1 基础 | 10 分钟阅读，15 篇帖子 | ✅ 全部 |
| 🥈 TL2 成员 | 300 分钟，80 篇，30 天 | ✅ 阅读 + 访问 |
| 🥇 TL3 常规 | 2000 分钟，500 篇，100 天 | ✅ 阅读 + 访问 |
| 👑 TL4 领袖 | 手动，隐藏 | ❌ |

## 技术架构

```
┌──────────────────────────────────────────────────────────┐
│                    handleScheduled (cron)                 │
│  ┌────────────────┐  ┌──────────────────────────┐        │
│  │ GLaDOS          │  │ 自动阅读引擎（慢速模式）  │        │
│  │ 签到引擎        │  │ runNodelocBatch()        │        │
│  │                 │  │  队列式，60-120s/帖      │        │
│  │ getAccount      │  │  15% 概率休息 20-40 分   │        │
│  │ DataObj()       │  │  NodeLoc / NodeSeek /   │        │
│  └────────────────┘  │  LinuxDO 复用            │        │
│                       └──────────────────────────┘        │
│                                                           │
│                    handleUpdate (webhook manual)           │
│  ┌──────────────────────────────────────────────────┐     │
│  │ 直读模式（fast）                                  │     │
│  │  latest.json → for loop → 3-7s sleep → timing    │     │
│  │  NodeLoc / NodeSeek / LinuxDO 各独立 inline 实现  │     │
│  │  坏帖 try/catch 跳过，实时 editMessage 更新进度   │     │
│  └──────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

### 阅读引擎说明

代码中有两套阅读逻辑：

| 模式 | 触发 | 节奏 | 状态管理 | 实现方式 |
|------|------|------|---------|---------|
| **直读（fast）** | 手动阅读（webhook callback） | 3-7 秒/帖 | 仅记录累计数 | inline，三站独立 |
| **慢读（cron）** | `handleScheduled` | 60-120 秒/帖 | 队列+休息+累计 | `runNodelocBatch` 共用 |

### 新增 Discourse 站注意事项

每个新论坛配合前需要验证：
- 话题 URL 路径：标准 `/t/{id}` 还是 `/n/topic/{id}`（如 NodeSeek）
- 是否包含 Cloudflare 防护
- Cookie 中是否含 `_t`（CSRF token fallback）
- `latest.json` 返回格式是否一致
- **`ctx.waitUntil()` 30 秒壁钟限制对 webhook 触发的所有操作生效**

## 环境变量 / Secrets

| 变量 | 说明 | 获取方式 |
|------|------|---------|
| `BOT_TOKEN` | Telegram Bot Token | [@BotFather](https://t.me/BotFather) |
| `ADMIN_ID` | 你的 Telegram User ID | [@userinfobot](https://t.me/userinfobot) |
| `GLADOS_DB` | KV Namespace ID | `wrangler kv:namespace create` |

## 本地开发

```bash
# 克隆
git clone https://github.com/Linsars/glados-discourse-bot.git
cd glados-discourse-bot

# 安装依赖
npm install

# 本地测试
wrangler dev --remote
```

## License

MIT
