# CF-Worker-Uptime ☁️

[![Deploy to Cloudflare Workers](https://img.shields.io/badge/Deploy%20to-Cloudflare%20Workers-orange?style=flat-square&logo=cloudflare)](https://workers.cloudflare.com/)

一个运行在 Cloudflare Workers 和 D1 数据库上的高性能、无服务器（Serverless）状态监控工具。
无需购买服务器，利用 Cloudflare 强大的全球边缘网络监控您的网站和 API。

## ✨ 特性 (Features)

- **完全无服务器**: 部署在 Cloudflare Workers 上，依托 Cron Triggers 定时触发，成本极低（个人使用通常免费）。
- **多协议支持**: 支持 HTTP(S) (GET/HEAD) 和 TCP 端口监控。
- **内置状态页**: 自带美观的单页应用 (SPA) 状态页，展示实时可用性和历史趋势。
- **通知系统**:
  - **邮件**: 通过 [Resend](https://resend.com) 发送告警邮件。
  - **Webhook**: 支持自定义回调 URL，可对接飞书、钉钉、Telegram 等。
- **高级配置**:
  - **防抖动 (Grace Period)**: 连续失败多次才报警，避免网络波动导致的误报。
  - **自定义验证**: 支持自定义 HTTP 状态码验证（如 200, 201, 204）。
  - **分组与标签**: 灵活对监控项进行分组和打标签（支持自定义颜色）。
- **故障公告**: 支持在配置文件中手动添加维护公告或故障说明。

---

## 🚀 快速开始 (Quick Start)

小白也能轻松上手的部署指南。

### 1. 准备工作

- 一个 [Cloudflare](https://www.cloudflare.com/) 账号。
- 本地安装了 [Node.js](https://nodejs.org/) 环境。
- 安装并登录 Wrangler CLI:
  ```bash
  npm install -g wrangler
  wrangler login
  ```

### 2. 获取代码

```bash
git clone https://github.com/your-username/CF-Worker-Uptime.git
cd CF-Worker-Uptime
npm install
```

### 3. 创建数据库

我们需要一个 D1 数据库来存储监控历史。运行以下命令：

```bash
npx wrangler d1 create uptime-db
```

**⚠️ 重要步骤**:
执行命令后，控制台会输出一段 JSON 信息。找到 `database_id`，复制它！

打开项目根目录下的 `wrangler.jsonc` 文件，找到 `d1_databases` 部分，替换 `database_id`:

```jsonc
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "uptime-db",
      "database_id": "你的-DATABASE-ID-粘贴在这里" // <--- 修改这里
    }
  ],
```

### 4. 初始化数据表

将数据表结构写入数据库：

```bash
npx wrangler d1 execute uptime-db --file=./schema.sql --remote
```

### 5. 修改配置 (Config)

`config.yaml` 是本项目的核心配置文件。你可以直接修改它来添加你要监控的网站。

```bash
# 编辑配置文件
code config.yaml
```

*(详细配置说明请见下文 [⚙️ 配置详解](#%EF%B8%8F-配置详解-configuration))*

### 6. 部署上线 (Deploy)

```bash
npx wrangler deploy
```

部署完成后，你会获得一个 `https://uptime-monitor.你的子域.workers.dev` 的链接，访问它即可看到你的状态页！

---

## ⚙️ 配置详解 (Configuration)

`config.yaml` 拥有丰富的功能，以下是一些**特色字段**的详细介绍：

### 全局设置 (`settings`)

| 字段 | 说明 | 推荐值 |
|------|------|--------|
| `title` | 状态页的标题 | "我的服务监控" |
| `notification_on_down_only` | **🌟 特色**: 如果设为 `true`，则只在服务**挂掉 (DOWN)** 时发送通知，恢复时不再发送。适合不想被恢复通知打扰的用户。 | `true` |
| `summary_exclusion` | **🌟 特色**: 在这里填入分组 ID，该分组将不会计入页面顶部的"系统整体状态"。适合用于监控一些非核心服务（如测试环境）。 | `["test_group"]` |
| `callback_url` | 通用 Webhook 地址，状态变更时会向此 URL 发送 POST 请求。 | `""` |

### 监控项配置 (`monitors`)

每个监控项都有很多可调参数：

```yaml
    monitors:
      - id: "blog_main"
        name: "个人博客"
        type: "http"       # 支持 'http' 或 'tcp'
        url: "https://blog.example.com"
        method: "HEAD"     # 使用 HEAD 请求可以减少流量消耗
        timeout: 5000      # 超时时间 (毫秒)
        expected_latency: 500 # 期望延迟，虽然目前不影响状态，但用于图表参考
        
        # 🌟 特色: 防抖动机制
        # 只有连续失败 3 次，才会判定为 DOWN 并发送通知
        grace_period: 3    
        
        # 🌟 特色: 自定义验证
        validation:
          status: [200, 301, 302] # 允许的状态码列表
          
        display:
          chart: true      # 是否显示延迟图表
          public_link: true # 点击名称是否跳转
```

### 故障公告 (`incidents`)

你可以在 `config.yaml` 中手动发布故障信息或维护计划，它们会置顶显示在状态页。

```yaml
incidents:
  - id: "incident-001"
    title: "数据库维护"
    type: "maintenance" # maintenance (维护) 或 incident (故障)
    status: "in_progress" # scheduled, in_progress, completed, resolved
    start_time: "2024-01-01T10:00:00Z"
    affected_monitors: ["blog_main"] # 关联受影响的监控项 ID
    updates:
      - timestamp: "2024-01-01T10:00:00Z"
        message: "正在进行数据库迁移..."
        status: "in_progress"
```

---

## 🔔 通知设置 (Notifications)

### 1. 邮件通知 (推荐)

本项目集成了 [Resend](https://resend.com) 邮件服务（免费额度足够个人使用）。

1. 注册 Resend 并获取 API Key。
2. 在 Cloudflare Workers 设置 Secret 环境变量（为了安全，不要写在代码里）：

```bash
# 设置 API Key
npx wrangler secret put RESEND_KEY
# (输入你的 Resend API Key)

# 设置发件人 (必须在 Resend 验证过域名)
npx wrangler secret put RESEND_SEND
# (输入如: alert@yourdomain.com)

# 设置收件人
npx wrangler secret put RESEND_RECEIVE
# (输入你的邮箱)
```

### 2. Webhook

在 `config.yaml` 中配置 `callback_url`。当状态发生变化时，Worker 会向该 URL 发送如下 JSON：

```json
{
  "monitor_id": "google",
  "status": "DOWN",
  "time": 1704364800000,
  "incident_id": "..."
}
```

你可以使用 Serverless 函数或自动化工具（如 n8n, IFTTT）接收此 Webhook 并推送到任意平台。

---

## 🛠️ 开发与调试

- **本地运行**:
  ```bash
  npx wrangler dev
  ```
  这会在本地启动服务，方便调试 UI 和逻辑。

- **查看实时日志**:
  ```bash
  npx wrangler tail
  ```
  如果部署后发现不工作，可以使用此命令查看线上报错。

## 📅 定时任务说明

Cloudflare Workers 的 Cron Triggers 默认配置在 `wrangler.jsonc` 中：

```jsonc
"triggers": {
  "crons": ["* * * * *"] // 默认为每分钟执行一次
}
```

如果需要修改频率，请更改此处的 Cron 表达式。
