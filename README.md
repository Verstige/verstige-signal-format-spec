# Verstige Trading Engine — Signal Format Specification

**Version 1.0 — July 30, 2026**

An open specification for the text format the trading engine parses. Write a
scanner that emits signals in this format and the engine tracks them, reports
TP/SL hits, and logs the trade — exactly as it does for its own signals.

## Who this is for

Anyone building a scanner, indicator, or signal source that should feed the
Verstige trading engine. If you have ever wanted a third-party tool to post
into the same pipeline as the built-in engines, this is the contract.

## The format

Every signal uses one template. Only the symbol tag and the decimal precision
change.

```
➖XAUUSD | NITRO ➖
☑️ ENTRY PRICE @ 4019.25
├ Order: BUY
├ 🔑 TP1: 4024.25 (+50 pips)
├ 🔑 TP2: 4034.25 (+150 pips)
└ 🔑 TP3: 4039.25 (+200 pips)
⛔️ Stop Loss: 4009.25 (100 pips)
```

The parser reads `direction`, `entry`, `sl`, and `tp1`–`tp4` from the text.
Three rules are load-bearing:

1. The header must use `➖` (U+27D6) on both sides — that is how the engine
   recognises a message as its own.
2. Every TP line must end with `(+{N} pips)` — the pips value is extracted
   from that parenthetical, not computed from the prices.
3. Direction must be uppercase `BUY` or `SELL`.

Supported symbols: `XAUUSD`, `BTCUSD`, `GBPJPY`, `EURUSD`, `US30`.
Supported engine tags: `NITRO`, `CIPHER`, `DRAGON`, `TITAN`, `FLUENCE`.

## How to use it

1. Read [`index.md`](./index.md) — the full specification, including the
   optional entry zone, per-symbol decimal precision, the reply format for
   TP/SL events, and the validation checklist.
2. Emit signals in that format to the engine's channel.
3. The engine replies in thread on the original message: `TP1 HIT`, `TP2 HIT`,
   `TP3 HIT`, `TP4 HIT`, `SL HIT`, `trade closed`.

## Terms

MIT licensed. See [LICENSE](./LICENSE).

The specification is open so that independent scanners can integrate. The
engine that consumes it is a separate, closed implementation — the format is
the interoperable surface, not the engine.
