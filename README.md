# ExitAnt Connors RSI2 — OKX

Deploy this automated trading bot to **your own Render account** as a background worker. The bot runs from a prebuilt Docker image;
this repository contains **only the Render Blueprint (`render.yaml`)** — no source code.

The bot trades **Larry Connors RSI2**, **Turtle only** or **Turtle + Connors RSI2 merge** on OKX USDT perpetual swaps with exactly the rules of the
TanQuant backtest engine, on the exchange's own candles — daily (the original rule), 4-hour or 1-hour, chosen in the bot; UTC-aligned, stocks/ETFs only US trading days and, on
4-hour/1-hour, only NYSE regular-session candles — for every indicator. On daily candles, Yahoo daily candles are used only to complete
the 200-day moving average of symbols with fewer than 200 exchange candles (recently listed).
Copy a TanQuant result with **“Copy bot setup”** and paste it into the bot to set everything up at once.

## Deploy to Render

[![Deploy to Render](https://render.com/images/deploy-to-render-button.svg)](https://render.com/deploy?repo=https://github.com/ExitAnt-dev/connors-rsi2-okx)

```
https://render.com/deploy?repo=https://github.com/ExitAnt-dev/connors-rsi2-okx
```

1. Click the button (or open the link) and sign in to Render.
2. Render reads `render.yaml` and proposes **one background worker** with a **1 GB persistent disk** mounted at `/data`
   (settings, books and positions survive restarts and redeploys).
3. Enter the environment variables listed below. They are marked `sync: false`, so Render asks you for them on the deploy screen
   and stores them **only in your own Render account**.
4. Click **Deploy Blueprint** and wait for the first deploy to finish.
5. Open Telegram and send **`/start`** to your bot to begin setup.

## Before you deploy

- **OKX account registered through ExitAnt.** At every start the bot checks your OKX UID with the ExitAnt server; only the UID is sent, never your keys.
- **API key** with the **Read** and **Trade** permissions, and the key's **passphrase**.
  **Never enable the Withdraw permission** — the bot never withdraws, and a key without it cannot move funds out of your account even if it leaks.
  *For the Turtle merge's cash vault* the bot moves USDT between your Trading and Funding accounts; per the OKX API docs this transfer is covered by
  the Trade permission. If OKX still refuses it (`50120`), the bot tells you and tries again an hour later — trading continues.
- **Futures mode or multi-currency margin** account mode, and the **long/short (hedge) position mode**. The bot switches the position mode when no
  position or order is open; otherwise set it yourself in OKX → Trade settings.
- If the API key has an **IP allowlist**, add your Render service's outbound IP addresses (Render → the service → Settings → Outbound IP addresses).
  Otherwise OKX rejects the bot's calls (`50110`), and the bot tells you the server's current IP.
- USDT in your OKX **Trading** account. USDT you deposit on OKX usually lands in the **Funding** account — move it to Trading
  (OKX app → Assets → Transfer); when the Trading account is empty the bot tells you how much is waiting in Funding.

## Environment variables

| Variable | What to enter |
|---|---|
| `API_KEY` | OKX API key |
| `SECRET_KEY` | OKX API secret |
| `PASSPHRASE` | OKX API passphrase (set when you created the key) |
| `TELEGRAM_BOT_TOKEN` | Bot token issued by [@BotFather](https://t.me/BotFather) |
| `TELEGRAM_CHAT_ID` | Your own Telegram chat ID — the bot ignores every other sender |

Nothing else to configure: the exchange, trading mode and state-file location are built into the image.

## What the bot does

- Decides once per candle of the chosen length — daily at UTC midnight, 4-hour/1-hour right after each candle closes (stocks: session candles only) —
  and sends market orders at once (the backtest's “fill at the close”). On 4-hour/1-hour candles every indicator length is counted in candles, as in TanQuant
  (200-day MA → 200-bar MA).
- Turtle only (bot-only — not a TanQuant Connors strategy): the Turtle side of the merge alone, always long, on one candle length, with no
  Connors entries. In settings you can switch between the three, except that the side with an open position cannot be switched off
  (a Turtle position blocks Connors only, a Connors position blocks Turtle only) — the bot says why and what you can choose instead.
- In the Turtle + Connors merge the Turtle side can run on its own candle length (e.g. Turtle 4-hour, Connors 1-hour) — chosen on the first Turtle
  screen of the wizard and under 🐢 Turtle rules in settings. Keeping both the same is the TanQuant backtest setup; a different pair is a bot-only
  setting that TanQuant cannot backtest (the Turtle channel and its ATR stop come from the Turtle candles, each side decides at its own candle close).
- Margin-saving entry (bot-only, not in the TanQuant backtest): when a new lot enters while the symbol already holds another lot (Turtle long,
  Connors long or Connors short — same direction or not), it uses the set leverage × 2 and ½ of the held lot's entry size; if the exchange still
  refuses for insufficient margin, × 4 and ¼ once more. The leverage goes back to the setting when that lot closes. The wizard's leverage step and final
  confirmation, the settings summary and the entry notice explain it.
- Places an exchange stop-loss order per symbol and direction (the farthest stop of the open lots) for **the bot's own quantity only** — a quantity
  you open yourself on the same side is never included. The bot watches the closer stops and the exit rules itself around the clock — **keep the bot running**.
- Keeps one account per symbol (trading pocket + vault), like TanQuant's multi-asset backtest. The Turtle merge moves part of the profit to the
  vault (your OKX Funding account) on a trend exit and reinjects it after a deep drawdown, by the USA Turtle rules.
- Never stops trading because of an exchange answer — a refused or failed request skips that symbol for that cycle and is retried.

⚠️ Futures trading carries a very high risk of loss. Backtest results do not guarantee future returns.
