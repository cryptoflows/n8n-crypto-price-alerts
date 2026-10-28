# n8n crypto price-level alerts (fire once, re-arm)

A free n8n workflow that checks crypto prices every minute and sends **one** Telegram and/or Discord message when a price level is crossed. It does not send the same alert again on every run. Once price crosses back, the level re-arms, so the next crossing alerts again.

- No exchange API keys. Prices come from Binance's public market-data host `data-api.binance.vision`, which also works in regions where binance.com is blocked.
- No n8n credentials needed. Telegram/Discord settings live in one `Config` node.
- Dry-run by default: if you leave the tokens empty, the workflow returns the messages instead of sending them.

## How it works

`Schedule (every minute) / Run manually → Config → Check → Notify`

| Node | Type | What it does |
|---|---|---|
| Schedule | Schedule Trigger | Cron `* * * * *` |
| Run manually | Manual Trigger | For testing in the editor |
| Config | Set | Telegram/Discord settings and the list of levels |
| Check | Code | Fetches prices, evaluates rules, keeps state |
| Notify | Code | Sends each alert to Telegram and/or Discord |

The core of `Check`:

```js
const state = $getWorkflowStaticData('global'); // persists between active (scheduled) runs
state.fired = state.fired || {};
for (const r of rules) {
  const p = px[r.sym];
  const hit = r.op === '>' ? p > r.lvl : p < r.lvl;
  if (hit && !state.fired[r.key]) { state.fired[r.key] = true; out.push(/* alert */); } // fire once
  if (!hit) state.fired[r.key] = false;                                                  // re-arm
}
```

## Reliability

A level is only marked as "already alerted" **after** the message was actually delivered. The Check node attaches the state update to the alert, and the Notify node applies it only once Telegram/Discord accepted the message. If a send fails (network error, bad token, Telegram down), the error is shown in the Notify output and the alert is retried on the next scheduled run instead of staying silent until price crosses back. Thanks to u/Full-SpoonFood for reporting this.

## Setup (about 5 minutes)

1. Download [`workflows/crypto-price-level-alerts.json`](workflows/crypto-price-level-alerts.json). In n8n, go to **Workflows → Import from File**.
2. Open **Config** and set:
   - `levels`: comma-separated rules such as `BTCUSDT>100000, BTCUSDT<80000, ETHUSDT>4000, SOLUSDT<120`
   - `telegramBotToken` + `telegramChatId` (get a token from @BotFather), and/or `discordWebhookUrl`
3. Click **Execute workflow** to test, then **activate** the workflow.

> **Gotcha:** n8n only saves workflow static data on production (active) executions, not on manual test runs. In the editor every manual run starts with empty state. That's expected.

Note: the workflow re-arms as soon as price is back on the other side of the exact level, with no buffer band. If price chops around a level, you can add hysteresis in the `!hit` branch, for example re-arm only when price is 1% away.

Tested on n8n 1.x. The JSON contains no credentials.

## Write-up

The full logic, line by line: **[Building crypto price-level alerts in n8n that fire once and re-arm](https://cryptoflows.hashnode.dev/building-crypto-price-level-alerts-in-n8n-that-fire-once-and-re-arm)**

## License

MIT

---

### More from me (paid / optional, clearly labelled)

- **Crypto alerts pack for n8n** v1.4 (10 workflows: listings, funding extremes, liquidations, BTC whale transfers, portfolio digest, DCA, market movers, EVM wallet watch incl. ERC-20 + exchange labels, Solana SOL+SPL wallet alerts, and this one): https://pjamour.gumroad.com/l/sfkvkp
- **@WalletWatcherAlertsBot** on Telegram: alerts when a wallet you watch sends or receives funds. Free for 1 wallet, Pro €6/month. https://t.me/WalletWatcherAlertsBot
- Free public whale-move feed: https://t.me/whalemovesalert
