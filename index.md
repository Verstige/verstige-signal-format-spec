# Verstige Trading Engine — Signal Format Specification
**Version 1.0 — July 30, 2026**
**Purpose:** External scanners must format their alert text to match the engine's expected parse format so trade management (TP/SL tracking, notifications, DB logging) works identically to internally-generated signals.

---

## 1. Architecture Overview

When a signal message is received in the Telegram channel, the engine:

1. Parses the message text to extract `direction`, `entry`, `sl`, `tp1–tp4`
2. Saves the trade to the local SQLite DB (`verstige_trades.db`)
3. Tracks the trade in memory — monitors price for TP/SL hits
4. Replies to the signal message (threaded) on each event: TP1 HIT, TP2 HIT, TP3 HIT, TP4 HIT, SL HIT, trade closed

**All updates are sent as threaded replies to the original signal message.** Your parser must produce a message that the engine can uniquely identify as its own signal, and the engine must be able to extract all price levels from the text.

---

## 2. Signal Message Format

Every engine uses the same structural template. The only differences are the **symbol tag** (line 1) and **decimal precision**.

### 2.1 Required Header
```
➖{SYMBOL} | {ENGINE} ➖
```
- `SYMBOL` must be one of: `XAUUSD`, `BTCUSD`, `GBPJPY`, `EURUSD`, `US30`
- `ENGINE` must be one of: `NITRO`, `CIPHER`, `DRAGON`, `TITAN`, `FLUENCE`
- These exact Unicode characters must be used: `➖` (U+27D6) on both sides

### 2.2 Required Body Fields (all engines)

| Field | Line Prefix | Example | Notes |
|-------|-------------|---------|-------|
| Entry price | `☑️ ENTRY PRICE @` | `☑️ ENTRY PRICE @ 4019.25` | Price with correct decimal precision per symbol |
| Direction | `├ Order:` | `├ Order: BUY` | Must be `BUY` or `SELL` (uppercase) |
| Stop Loss | `⛔️ Stop Loss:` | `⛔️ Stop Loss: 4009.25 (100 pips)` | Must include pips in parentheses |
| TP1 | `├ 🔑 TP1:` | `├ 🔑 TP1: 4024.25 (+50 pips)` | Must include pips in parentheses |
| TP2 | `├ 🔑 TP2:` | `├ 🔑 TP2: 4034.25 (+150 pips)` | Must include pips in parentheses |
| TP3 | `├ 🔑 TP3:` | `├ 🔑 TP3: 4039.25 (+200 pips)` | Must include pips in parentheses |
| TP4 | `└ 🔑 TP4:` | `└ 🔑 TP4: 4069.25 (+500 pips)` | Must include pips in parentheses |

> ⚠️ **Critical:** Each TP line must end with `(+{N} pips)` — the parser extracts pips from this parenthetical. Format must be exactly `(+{integer} pips)` with a space before "pips".

### 2.3 Entry Zone (optional — NITRO only)
If the signal has an entry range, include this line after the Order line:
```
└ ZONE: {low:.2f} - {high:.2f}
```
Example: `└ ZONE: 4019.00 - 4019.50`

### 2.4 Risk-Reward Line (optional — all engines)
```
R:R: {rr_string}
```
Example: `R:R: 1:0.5 / 1:1.5`

### 2.5 Closing Disclaimer
Every message must end with:
```
⚠️ Trading involves risk. Only trade funds you can afford to lose.
```

---

## 3. Complete Examples by Engine

### 3.1 NITRO — XAUUSD (Gold)

```
➖XAUUSD | NITRO ➖

☑️ ENTRY PRICE @ 4019.25
├ Order: BUY
└ ZONE: 4019.00 - 4019.50

⛔️ Stop Loss: 4009.25 (100 pips)

├ 🔑 TP1: 4024.25 (+50 pips)
├ 🔑 TP2: 4034.25 (+150 pips)
├ 🔑 TP3: 4039.25 (+200 pips)
└ 🔑 TP4: 4069.25 (+500 pips)

R:R: 1:0.5 / 1:1.5 / 1:2.0 / 1:5.0

⚠️ Trading involves risk. Only trade funds you can afford to lose.
```

**Precision:** 2 decimal places for all prices.

---

### 3.2 CIPHER — BTCUSD (Bitcoin)

```
➖BTCUSD | CIPHER➖

☑️ ENTRY PRICE @ 67250.0
├ Order: BUY

⛔️ Stop Loss: 67050.0 (200 pips)

├ 🔑 TP1: 67450.0 (+200 pips)
├ 🔑 TP2: 67850.0 (+600 pips)
└ 🔑 TP3: 68250.0 (+1000 pips)

R:R: 1:1.0 / 1:3.0 / 1:5.0

Confirmations:
  • Bias (5m): BULLISH
  • Confluence: HIGH | Score 6/8
  • KDJ bull cross
  • MACD bullish divergence
  • Volume above average

⚠️ Trading involves risk. Only trade with funds you can afford to lose.
```

**Precision:** 1 decimal place for all prices.

---

### 3.3 DRAGON — GBPJPY (British Pound / Japanese Yen)

```
➖GBPJPY | DRAGON➖

☑️ ENTRY PRICE @ 197.85000
├ Order: SELL

⛔️ Stop Loss: 197.90000 (50 pips)

├ 🔑 TP1: 197.80000 (+50 pips)
├ 🔑 TP2: 197.75000 (+100 pips)
├ 🔑 TP3: 197.70000 (+150 pips)
└ 🔑 TP4: 197.60000 (+250 pips)

R:R: 1:1.0 / 1:2.0 / 1:3.0 / 1:5.0

⚠️ Trading involves risk. Only trade with funds you can afford to lose.
```

**Precision:** 5 decimal places for all prices.

---

### 3.4 TITAN — EURUSD

```
➖EURUSD | TITAN➖

☑️ ENTRY PRICE @ 1.08500
├ Order: BUY

⛔️ Stop Loss: 1.08450 (50 pips)

├ 🔑 TP1: 1.08550 (+50 pips)
├ 🔑 TP2: 1.08600 (+100 pips)
├ 🔑 TP3: 1.08650 (+150 pips)
└ 🔑 TP4: 1.08750 (+250 pips)

R:R: 1:1.0 / 1:2.0 / 1:3.0 / 1:5.0

⚠️ Trading involves risk. Only trade with funds you can afford to lose.
```

**Precision:** 5 decimal places for all prices.

---

### 3.5 FLUENCE — US30 (Wall Street 30)

```
➖US30 | FLUENCE➖

☑️ ENTRY PRICE @ 35250
├ Order: BUY

⛔️ Stop Loss: 35150 (100 pips)

├ 🔑 TP1: 35250 (+50 pips)
├ 🔑 TP2: 35350 (+100 pips)
├ 🔑 TP3: 35550 (+250 pips)
└ 🔑 TP4: 35550 (+350 pips)

R:R: 1:0.5 / 1:1.0 / 1:2.5 / 1:3.5

⚠️ Trading involves risk. Only trade funds you can afford to lose.
```

**Precision:** Integer (no decimal places) for all prices.

---

## 4. Trade Update Messages (Threaded Replies)

When the engine detects a TP or SL hit, it sends a threaded reply to the original signal message. External scanners must produce messages in these exact formats for the engine to log them correctly.

### 4.1 Entry Activated
```
Trade Activated ✅
```
No further content needed. Engine sends this when price reaches the entry level.

---

### 4.2 TP1 Hit
```
TP1 HIT 🔐 +{N} PIPS LOCKED IN
⭕️ Move SL to Entry — ride to TP2
```
Engine moves SL to breakeven (+0 pips) when TP1 is hit.

**Example:**
```
TP1 HIT 🔐 +50 PIPS LOCKED IN
⭕️ Move SL to Entry — ride to TP2
```

---

### 4.3 TP2 Hit
```
TP2 HIT 🔐 +{N} PIPS SECURED
⭕️ Move SL to TP1 — stay in for TP3
```
Engine moves SL to TP1 price level.

**Example:**
```
TP2 HIT 🔐 +100 PIPS SECURED
⭕️ Move SL to TP1 — stay in for TP3
```

---

### 4.4 TP3 Hit
```
TP3 HIT 🔐 +{N} PIPS SECURED
⭕️ Move SL to TP2 — finish strong
```
Engine moves SL to TP2 price level.

**Example:**
```
TP3 HIT 🔐 +250 PIPS SECURED
⭕️ Move SL to TP2 — finish strong
```

---

### 4.5 TP4 Hit (Trade Closed)
```
TP4 HIT 🎯 +{N} PIPS TOTAL
Trade Closed ✅
```
Engine closes the trade with maximum profit.

**Example:**
```
TP4 HIT 🎯 +500 PIPS TOTAL
Trade Closed ✅
```

---

### 4.6 SL Hit (Trade Closed — Loss)
```
SL HIT ❌ {PIPS} PIPS LOST
Trade Closed ❌
```
Engine closes the trade at stop loss.

**Example:**
```
SL HIT ❌ -100 PIPS LOST
Trade Closed ❌
```

---

### 4.7 Trade Voided (Entry Not Triggered)
```
➖US30 | FLUENCE➖

Trade voided ❌ Reason: Entry not triggered

Entry was: 35250.0

⚠️ Trading involves risk. Only trade funds you can afford to lose.
```
(FLUENCE/NEXUS format; other engines use similar structure)

---

### 4.8 Session End (DRAGON only)
```
➖GBPJPY | DRAGON➖

⭕️ TRADE CLOSED
Reason: SESSION END
Exit: 197.82500
TPs Secured: TP1 | TP2
P&L: +150 pips
```

---

## 5. Parsing Rules

The engine parses messages using line-prefix matching. Each line must begin with the exact prefix below, with a single space after the prefix.

| Field | Parsed From Line | Extraction Method |
|-------|-----------------|-------------------|
| Symbol | `➖{S} | {E}➖` | Split on ` \| `, take first segment, strip `➖` |
| Engine | `➖{S} \| {E}➖` | Split on ` \| `, take second segment, strip `➖` |
| Direction | `├ Order:` | Split on `: `, take second value, strip whitespace |
| Entry | `☑️ ENTRY PRICE @` | Split on `@ `, take second value, strip whitespace |
| SL | `⛔️ Stop Loss:` | Split on `: `, take second value, strip pips annotation |
| TP1 | `├ 🔑 TP1:` | Split on `: `, take second value, extract price + pips |
| TP2 | `├ 🔑 TP2:` | Split on `: `, take second value |
| TP3 | `├ 🔑 TP3:` | Split on `: `, take second value |
| TP4 | `└ 🔑 TP4:` | Split on `: `, take second value |
| Pips (per TP) | `TP line parenthetical` | Regex: `(+\d+ pips)` from end of line |

### Decimal Precision by Symbol
| Symbol | Precision | Example |
|--------|-----------|---------|
| XAUUSD | 2 dp | `4019.25` |
| BTCUSD | 1 dp | `67250.0` |
| GBPJPY | 5 dp | `197.85000` |
| EURUSD | 5 dp | `1.08500` |
| US30 | Integer | `35250` |

---

## 6. Message ID Requirement

The engine sends all TP/SL updates as **threaded replies** to the original signal message using `reply_to_message_id`. This means:

1. The signal message must be sent by the same bot that will manage the trade
2. The bot must capture the `message_id` of the sent signal
3. All subsequent updates must pass that `message_id` as `reply_to_message_id`

If your external scanner sends the signal and captures the `message_id`, you can hand off the `message_id` to the engine alongside the parsed signal data. Alternatively, if you are building the scanner to parse incoming external alerts, the engine will send the signal itself — the external alert triggers the engine, which then formats and sends the message and captures the ID.

---

## 7. Summary — What an External Scanner Must Produce

| Requirement | Detail |
|-------------|--------|
| Header | `➖{SYMBOL} \| {ENGINE} ➖` on its own line |
| Entry line | `☑️ ENTRY PRICE @ {price}` |
| Direction | `BUY` or `SELL` (uppercase) on `Order:` line |
| SL line | `⛔️ Stop Loss: {price} ({N} pips)` |
| TP1–TP4 lines | `├ 🔑 TP1: {price} (+{N} pips)` format |
| Threaded replies | Must include `reply_to_message_id` of signal |
| DB logging | Engine handles this automatically |

---

## 8. Contact

For integration support, reach out to the Verstige trading operations team.
Provide a sample of your scanner's current output format and the Telegram channel ID.
