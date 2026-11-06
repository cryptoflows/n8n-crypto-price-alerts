# n8n crypto price-level alerts (fire once, re-arm)

A free n8n workflow that checks crypto prices every minute and sends **one** Telegram and/or Discord message when a price level is crossed. It does not send the same alert again on every run. After an alert, the level re-arms once price has stayed on the safe side and outside a percent-of-level band (default 0.1%) for two consecutive scheduled runs, so the next crossing alerts again.

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
| Config | Set | Telegram/Discord settings, levels, max price age and re-arm band |
| Check | Code | Fetches timestamped prices, aborts stale/failed data, evaluates rules, keeps state |
| Notify | Code | Sends each alert to Telegram and/or Discord |

The core of `Check`: one public `GET /api/v3/ticker/24hr?symbols=...` request supplies `lastPrice` and `closeTime` (milliseconds) for all configured symbols. The old `ticker/price` endpoint has no timestamp. Before evaluating **any** rule, Check verifies every used symbol has a valid price/time and is no older than `maxPriceAgeSec` (default 180). A failed/timed-out request or missing/stale data returns a visible `error` item in Check, not “No level crossed”; no alerts or state writes occur.

```js
// Simplified state transitions after all prices pass freshness validation:
const hit = r.op === '>' ? p > r.lvl : p < r.lvl;
const distance = r.op === '>' ? r.lvl - p : p - r.lvl;
const safe = !hit && distance >= r.lvl * (rearmBandPct / 100);
// unseen key: seed fired=hit, safeRuns=0; no alert
// latched + !safe (including inside band): safeRuns=0; stay latched
// latched + safe: increment streak; at 2 clear fired and reset streak
// armed + hit: emit alert with commit [fired=true, safeRuns=0]
// Notify applies that commit only after successful delivery / dry-run
```

`decide()` is pure: it returns alerts and a copy of quiet-path state. Check persists the quiet updates per level; Notify applies alert commits only after delivery.

## Reliability

A level is only marked as "already alerted" **after** the message was actually delivered. The Check node attaches the state update to the alert, and the Notify node applies it only once Telegram/Discord accepted the message. If a send fails (network error, bad token, Telegram down), the error is shown in the Notify output and the alert is retried on the next scheduled run instead of staying silent until price crosses back. Thanks to u/Full-SpoonFood for reporting this.

**Fresh imports don't fire instantly.** A duplicated or freshly imported workflow starts with empty state. On a level's first run, if price is already past it, the level is seeded as already alerted and no message is sent; if price is on the safe side, it is seeded as armed. Thanks to u/vlotstroom for the cold-start seed and percent-band suggestions.

**Re-arm needs depth AND two runs.** After an alert, price must be on the safe side and at least `level * (rearmBandPct / 100)` away for two consecutive scheduled runs. Default `rearmBandPct` is **0.1** (0.1%): for `BTCUSDT>100000`, a price at or below 99900 qualifies; for `ETHUSDT<4000`, at or above 4004 qualifies. A hit or a safe-side price inside the band resets the streak to zero. Set the band to 0 to disable the depth requirement, not the two-run timing. Thanks to u/Fit-Corner2382 (duration) and u/vlotstroom (depth). Quiet seed/streak/reset/re-arm writes happen in Check per level, independently of another level's failed send. Alert commits still wait for successful delivery / dry-run.

**Stale data is an error, not silence.** `maxPriceAgeSec` defaults to **180** seconds. Missing/invalid `lastPrice` or `closeTime`, any used symbol older than the limit, or an HTTP failure/20-second timeout aborts the whole run before seeding, alerting or changing `fired`/`safeRuns`. Inspect the visible Check error (symbol, age and limit for bad data); Notify may have no output when there is no alert. Thanks to u/AICompanionz for the stale-data check.

**Tune the band offline, not in the minute run.** Following u/vlotstroom's method, pull a few weeks of 1-minute klines for the pair from `data-api.binance.vision`, count re-crosses at different `rearmBandPct` values, and pick the smallest band that kills most churn. The band filters slow chop; two consecutive runs filter one-minute wicks. This workflow does not fetch klines or run a live calibrator.

## Setup (about 5 minutes)

1. Download [`workflows/crypto-price-level-alerts.json`](workflows/crypto-price-level-alerts.json). In n8n, go to **Workflows → Import from File**.
2. Open **Config** and set:
   - `levels`: comma-separated rules such as `BTCUSDT>100000, BTCUSDT<80000, ETHUSDT>4000, SOLUSDT<120`
   - `telegramBotToken` + `telegramChatId` (get a token from @BotFather), and/or `discordWebhookUrl`
3. Click **Execute workflow** to test, then **activate** the workflow.

> **Gotcha:** n8n only saves workflow static data on production (active) executions, not on manual test runs. In the editor every manual run starts with empty state, so a manual run only seeds levels (a level that is already past is treated as alerted and stays silent). That's expected. To see real alerts, test on the active workflow.

**Additional Config fields** (numeric assignments in the existing Config node):

| Field | Type | Default | Meaning |
|---|---|---|---|
| `maxPriceAgeSec` | number | `180` | Maximum age in seconds of each used symbol's `closeTime`; missing/NaN setting falls back to 180. |
| `rearmBandPct` | number | `0.1` | Percent of the level required on the safe side for each of the two re-arm runs; 0 disables depth only. Missing/NaN setting falls back to 0.1. |

Changing a level string creates a new state key; its next fresh observation seeds silently rather than alerting. Failed/stale data never seeds a new key.

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
