# CF-Worker-Uptime

<img width="1083" height="1064" alt="image" src="https://github.com/user-attachments/assets/06ec0336-66e4-4f3a-bd28-6b46a49fa180" />


A high-performance, serverless uptime monitor running on Cloudflare Workers and D1 Database.

## Features

- **Multi-Protocol Monitoring**: Supports HTTP(S) and TCP checks.
- **Serverless Architecture**: Runs entirely on Cloudflare Workers (Cron Triggers) + D1.
- **Declarative Configuration**: Manage all monitors via a single `config.yaml` file.
- **Status Page**: Built-in status page (SPA) served directly from the Worker.
- **Notification System**: Built-in Resend Email integration and customizable Webhook support.
- **Advanced Logic**:
  - Configurable timeouts and expected latency.
  - "Grace Period" to prevent false alarms (flapping).
  - "Down Only" notification mode.
  - Custom User-Agent to avoid blocking.

## Quick Start

### 1. Prerequisites

- Node.js & npm
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/install-and-update/) authenticated with your Cloudflare account.

### 2. Setup

1. **Clone the repository**:
   ```bash
   git clone https://github.com/your-username/CF-Worker-Uptime.git
   cd CF-Worker-Uptime
   npm install
   ```

2. **Create D1 Database**:
   ```bash
   npx wrangler d1 create uptime-db
   ```
   Copy the `database_id` from the output and update `wrangler.toml`:
   ```toml
   [[d1_databases]]
   binding = "DB"
   database_name = "uptime-db"
   database_id = "YOUR_DATABASE_ID_HERE"
   ```

3. **Initialize Database Schema**:
   ```bash
   npx wrangler d1 execute uptime-db --file=./schema.sql --remote
   ```

4. **Configure Monitors**:
   Edit `config.yaml` to add your services. See [Configuration](#configuration) below.

5. **Deploy**:
   ```bash
   npx wrangler deploy
   ```

### 3. Notifications (Optional)

To enable email notifications via **Resend**:

1. Get your API Key from [Resend](https://resend.com).
2. Set secrets in Cloudflare:
   ```bash
   npx wrangler secret put RESEND_KEY     # e.g., re_12345678
   npx wrangler secret put RESEND_SEND    # e.g., uptime@yourdomain.com
   npx wrangler secret put RESEND_RECEIVE # e.g., admin@yourdomain.com
   ```

## Configuration

The `config.yaml` file controls everything.

```yaml
settings:
  title: "My Status Page"
  logo: "https://example.com/logo.png" # Optional
  notification_on_down_only: true # If true, only notify on DOWN events (ignore recovery/degraded)
  callback_url: "" # Optional generic webhook

groups:
  - id: "websites"
    name: "Websites"
    monitors:
      - id: "google"
        name: "Google Search"
        type: "http" # or 'tcp'
        url: "https://www.google.com"
        method: "HEAD"
        timeout: 5000
        expected_latency: 500
        grace_period: 3 # Wait for 3 consecutive failures before alerting
        display:
          chart: true
          public_link: true
```

## Development

- **Local Development**:
  ```bash
  npx wrangler dev
  ```
- **View Logs**:
  ```bash
  npx wrangler tail
  ```

## License

MIT
