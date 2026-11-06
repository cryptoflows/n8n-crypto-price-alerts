# n8n crypto price-level alerts (fire once, re-arm)

A free n8n workflow that checks crypto prices every minute and sends **one** Telegram and/or Discord message when a price level is crossed. It does not send the same alert again on every run. After an alert, the level re-arms once price has stayed on the safe side for two scheduled runs, so the next crossing alerts again.

- No exchange API keys. Prices come from Binance's public market-data host `data-api.binance.vision`, which also works in regions where binance.com is blocked.
- No n8n credentials needed. Telegram/Discord settings live in one `Config` node.
- Dry-run by default: if you leave the tokens empty, the workflow returns the messages instead of sending them.

## Who runs this
WalletWatcher Alerts (@WalletWatcherAlertsBot), the Whale Moves feed (@whalemovesalert) and this repo are published by **iMonitoring** (internetmonitoring.nl), Lelystad, the Netherlands.
Contact: info@internetmonitoring.nl

**WalletWatcher Pro:** follow up to 20 wallets for €6 a month, paid inside the bot. On the free plan, each friend you invite with /invite adds 3 extra wallets.

## WalletWatcher Pro: USD alerts + n8n webhook
Wallet move alerts in Telegram with the **$ value** of each transfer. Set a **$ threshold** with `/min 5000` so you only hear about moves above $5k. Pro can also POST every alert to your own n8n (or any) webhook with a `usd` field, via `/webhook <url>`.

**€6, one payment for 30 days of Pro.** It doesn't renew automatically. If it isn't working for you within 14 days of payment, you get a full refund.

1. [Buy on Gumroad](https://pjamour.gumroad.com/l/qxyjv). Your license key is in the receipt.
2. Open [@WalletWatcherAlertsBot](https://t.me/WalletWatcherAlertsBot?start=readme) and send `/activate <your license key>`.

Alerts only. No trading, no signals, no advice.

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
// state.fired[key]: true = alerted (latched), false = armed, absent = never seen
// state.safeRuns[key]: consecutive scheduled runs on the safe side while latched
for (const r of rules) {
  const p = px[r.sym];
  const hit = r.op === '>' ? p > r.lvl : p < r.lvl;
  if (!(r.key in state.fired)) { state.fired[r.key] = hit; state.safeRuns[r.key] = 0; continue; } // cold-start seed: no alert
  if (state.fired[r.key]) { // latched: re-arm only after 2 safe runs in a row
    if (hit) state.safeRuns[r.key] = 0;
    else if (++state.safeRuns[r.key] >= 2) { state.fired[r.key] = false; state.safeRuns[r.key] = 0; }
    continue;
  }
  // armed + hit: alert. NOT marked here. The alert carries a `commit` (fired=true, safeRuns=0);
  // Notify applies it only after Telegram/Discord accepted the message, so a failed send retries next run.
  if (hit) out.push({ json: { text, commit: [{ path: ['fired', r.key], value: true }, { path: ['safeRuns', r.key], value: 0 }] } });
}
// Notify, per alert: send → on success apply(commit); on error report it and leave fired unset
```

## Reliability

A level is only marked as "already alerted" **after** the message was actually delivered. The Check node attaches the state update to the alert, and the Notify node applies it only once Telegram/Discord accepted the message. If a send fails (network error, bad token, Telegram down), the error is shown in the Notify output and the alert is retried on the next scheduled run instead of staying silent until price crosses back. Thanks to u/Full-SpoonFood for reporting this.

**Fresh imports don't fire instantly.** A duplicated or freshly imported workflow starts with empty state. On a level's first run, if price is already past it, the level is seeded as already alerted and no message is sent; if price is on the safe side, it is seeded as armed. Thanks to u/vlotstroom for this one.

**Re-arm needs two runs on the safe side.** After an alert, the level stays latched until price has been on the safe side for two scheduled runs in a row. One noisy print back across can't re-arm it and fire again a minute later, while a held retrace still releases the latch. The safe-run streak is stored next to the fired flag; on the alert path both are written only after a successful send, so a failed send still retries next run. Thanks to u/Fit-Corner2382 for suggesting this.

## Setup (about 5 minutes)

1. Download [`workflows/crypto-price-level-alerts.json`](workflows/crypto-price-level-alerts.json). In n8n, go to **Workflows → Import from File**.
2. Open **Config** and set:
   - `levels`: comma-separated rules such as `BTCUSDT>100000, BTCUSDT<80000, ETHUSDT>4000, SOLUSDT<120`
   - `telegramBotToken` + `telegramChatId` (get a token from @BotFather), and/or `discordWebhookUrl`
3. Click **Execute workflow** to test, then **activate** the workflow.

> **Gotcha:** n8n only saves workflow static data on production (active) executions, not on manual test runs. In the editor every manual run starts with empty state, so a manual run only seeds levels (a level that is already past is treated as alerted and stays silent). That's expected. To see real alerts, test on the active workflow.

Note: after an alert, the workflow re-arms once price has been back on the other side of the exact level for two scheduled runs in a row (two minutes on the default schedule). There is no buffer band. If price chops slowly around a level, you can add a percent band in the latched `!hit` branch, for example only count a run as safe when price is 1% away.

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
