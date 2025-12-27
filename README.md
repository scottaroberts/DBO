# DBO_Prod (Range Breakout + Recoup Add) — Business Rules & Feature Inventory

This repo contains the **current TradingView strategy** used for the DBO bot (“DBO_Prod”). The focus of this README is **business rules** (what the system does and why), plus a **feature inventory** you can use to plan a migration to a new platform.

> Historical docs that inspired parts of this README: the bracket-cycle spec and the MVP simplification spec. fileciteturn0file1 fileciteturn0file2

---

## What the bot does (1‑minute mental model)

1. **Build a range** over a fixed lookback window ending at a configurable “range end” time.
2. Once the range is formed and quality filters pass, **place both breakout brackets**:
   - Long stop above the range
   - Short stop below the range
3. When one side fills, optionally place a **Recoup (Opposite Add)** in the *opposite direction* (often routed to **YM** as a proxy).
4. Manage the day with **news blocks**, **daily P/L halts**, a **trade-limit**, and strict **end-of-session cleanup**.
5. If the Recoup hits its TP, the bot performs a **paired exit** to close the micro leg too.

---

## Key concepts / glossary

- **Range window:** the period used to measure `dayHigh/dayLow` (the “box”).
- **Breakout window:** the period after the range ends where trading is allowed (until `exitTime`).
- **Bracket set:** a pair of pending orders (Long + Short) with attached TP/SL.
- **Cycle:** a full sequence from “brackets armed” → (fill) → (profit/stop) → (rearm or stand-down).
- **Recoup (Opposite Add):** an additional opposite-direction trade intended to recover loss / stabilize outcomes.
- **Proxy ticker:** when enabled, Recoup orders/exits are emitted as alerts for a different symbol (typically **YM**).

---

## Time & session rules

### Time zones
- **`tzInput`**: primary session time zone (defaults to `America/Los_Angeles`).
- **`newsTzInput`**: interpretation time zone for *news HHMM* times (defaults to Pacific).

### Session construction (single daily session)
Inputs:
- **Range End**: `rangeEndHour`, `rangeEndMin` (default ~03:34 in `tzInput`)
- **Lookback**: `lookbackH`, `lookbackM` (default 4:00)
- **Exit Duration**: `exitDurH`, `exitDurM` (default 9:35)

Business rule:
- `rangeEnd` is today’s “Range End” timestamp in `tzInput` (DST-safe).
- `rangeStart = rangeEnd − lookbackDuration`.
- `exitTime = rangeEnd + exitDuration`.
- **Early-exit window opens at** `earlyExitTime = exitTime − earlyExitMinutes`.

### Daily boundary for P/L halts
- The “trading day” for halt math resets at **15:00 (3:00pm) in `tzInput`**.
- If the current bar is earlier than today’s 15:00, the day-start rolls back to yesterday’s 15:00.

---

## Range formation & levels

During the **range window**:
- Track `dayHigh` = highest high; `dayLow` = lowest low.

After the range ends:
- Compute breakout levels from offsets:
  - `longEntry  = dayHigh + entryOff`
  - `shortEntry = dayLow  − entryOff`
  - `longStop   = dayLow  − stopOff`
  - `shortStop  = dayHigh + stopOff`
- Range width and TP:
  - `rangeSize = |longEntry − longStop|`
  - `tpSize = rangeSize × tpRatio`
  - `longTP  = longEntry  + tpSize`
  - `shortTP = shortEntry − tpSize`

Stop-loss type:
- `slType = RR` → stop size derived from TP (`rrRatio`)
- `slType = Points` → fixed `ptsSL`

---

## Quality filters (when we do **not** trade)

The bot can invalidate the session (for the whole breakout window) based on:

1. **Min/Max range width**
   - `minRangePoints` (disable with -1)
   - `maxRangePoints` (disable with -1)

2. **Midpoint start proximity (deferred start)**
   - `midpointStartTolerancePct > 0` means: **do not start** unless price returns near midpoint during the breakout window.
   - A “deferred start” is visualized as a red box that extends until the official start is latched.

3. **News-based blocking** (see next section)

Snapshot rule (important):
- On the first *eligible* breakout moment, the bot freezes a “sessionValid” decision so it doesn’t oscillate later during the day (the system treats “this is a tradable day” as a yes/no latch).

---

## News blocking & intraday delay policy

News events are stored internally as `(date, HHMM, TypeId)` rows.

### Severity toggles
- The bot supports three severities (1/2/3) with enable toggles:
  - `useSev3Filter`, `useSev2Filter`, `useSev1Filter`

### Per-type policy
For each TypeId, a dropdown selects one policy:
- **Inherit**: blocks only if that severity’s toggle is enabled.
- **Ignore**: never blocks.
- **Delay then trade**: blocks until event time + configured delay minutes.
- **Block entire day**: blocks the full day.

### Intraday Delay Mode (all severities)
- `useSev3TimeWindow = true` enables **Intraday Delay Mode**.
- In this mode, the bot:
  - Blocks the day **only** if there is at least one “Block entire day” event **or**
  - Blocks **until** the latest “timed” event today plus `sev3PostDelayMin`.

Important detail:
- Timed events require a valid HHMM (> 0). Rows with `HHMM = 0000` only matter if their policy is **Block entire day**.

### How this affects trading
- When news is blocking:
  - New bracket placements are prevented.
  - “Halt” state is shown in telemetry.
- When the last timed event + delay passes, trading can resume **within the same session** (if other gates allow).

---

## Risk management & trade limits

### Base sizing
- `riskDollars` is the **target $ risk** for the initial micro trade sizing.

### Daily risk halts (P/L caps)
- `maxDailyLoss` (caps new risk; also used to cap Recoup additions)
- `maxDailyProfit` (halts new trading once reached)

PnL computation:
- Uses **closed trades only**, summed since the 15:00 day boundary:
  - `points = (exit − entry)` (sign-aware for long/short)
  - `dollars = points × syminfo.pointvalue × |qty|`
  - `net = dollars − strategy.closedtrades.commission(i)`

Note:
- If you configure commissions/slippage in the strategy settings UI, those values will be reflected via TradingView’s closed-trade prices and `commission(i)`.

### Trade limit
- `tradeLimit` counts **sets** (Long+Short bracket placements), not individual fills.
- The bot will not arm a new set once `lastIssuedTradeNumber >= tradeLimit`.

### Price gate before re-arming
After the first bracket set has been issued:
- The bot requires price to return **inside the range** before it will place the next set.

### Midpoint quality after Trade 1
If `requireMidpointReturn = true`:
- After the first completed trade, re-arming can require a midpoint “return” confirmation (bar close near midpoint), depending on context.

---

## Bracket cycle & stateflows (business logic)

### Placement
When all entry preconditions pass (session valid, not blocked, not halted, no open position, locks clear, tradeLimit ok):
- Place **both** `Long` and `Short` brackets as a set.

### On profit (TP hit)
- Perform centralized “profit cleanup”:
  - Cancel remaining bracket inventory (opposite side + Recoup).
  - Reset cycle state.
  - Optionally **re-arm** (immediately or deferred until midpoint gate passes).
- Schedule a **one-bar-later “Emergency” sweep** to reduce the odds of a broker / webhook desync.

### On stop (SL hit)
- **Keep the opposite side** resident (so it can still enter).
- Clear only the side that stopped out.
- If the opposite later stops as well, the bot re-arms a new set.
- Special case: inside the Early Exit window, on stop the bot cancels everything and stands down (no “flip” risk near the end).

---

## Recoup (Opposite Add) & YM proxy routing

### Trigger
- When a micro position opens (Long or Short), the bot evaluates a Recoup add in the **opposite direction**.

### Entry/TP logic
- Recoup entry is offset from the stopped side’s stop level by `oppositeReissueEntryOffsetPts`.
  - If this input is **-1**, the bot uses a “legacy match” offset (`entryOff − stopOff`).
- Recoup TP can be overridden with `oppositeTpOverride` (if set ≥ 0).

### Stop distance scaling (Recoup only)
- `oppAddStopPct` scales the Recoup stop distance vs the original micro bracket distance.
  - Example: 80% means the Recoup stop is closer than the original stop distance.

### Sizing (risk-delta targeting)
- The bot targets a Recoup that increases total risk toward:
  - `baseRisk × oppositeQtyMultiplier`
- It computes the **additional** quantity needed (“risk delta”) rather than mirroring the micro size.

### Proxy routing (YM)
If `useProxyRealtimeOnly = true` and the script is running in realtime:
- Recoup orders are emitted as proxy alerts for the YM ticker (derived from the chart symbol).
- Quantity conversion uses `proxyQtyMul` (default ~0.10, i.e., YM ≈ 10× MYM).

Backtesting behavior:
- In historical bars (non-realtime), Recoup trades are simulated on the chart symbol so Strategy Tester can model the logic.

### Paired exit on Recoup TP
If the Recoup TP level is touched:
- The bot issues a **paired exit** to close the MYM leg as well, to lock the combined outcome.

---

## Exits & safety behavior

The strategy has multiple exit “doors,” but the intent is consistent:

1. **ProfitTP cleanup** — cancel opposite inventory and reset cycle; may re-arm.
2. **Early Exit** — near session end: if the current position is profitable (close vs avg), exit early.
3. **End of Session (EOS)** — at or after `exitTime`: cancel orders, flatten positions, and sweep proxy exits.
4. **Emergency exit (scheduled)** — often triggered one bar after profit to re-emit exit/cancel logic and reduce desync risk.

---

## Visuals & telemetry

Key visuals you’ll see on-chart:
- **Range box** during formation.
- **Deferred-start red box** if midpoint gating delays the official start.
- **Active (yellow) box** during the trading window.
- Optional debug labels:
  - L1: trade labels
  - L2: opposite side / news labels
  - L3: low-level trace

Data-window / panel concepts:
- Trading status (Trading vs Blocked)
- Next block / resume time (when intraday delay mode is on)
- Day P/L and halt reason

---

## Feature inventory (for migration roadmap)

### Session & time handling
- [x] DST-safe time anchors via `timestamp(tz, ...)`
- [x] Range window built from RangeEnd + Lookback duration
- [x] Breakout window built from RangeEnd + ExitDuration
- [x] Daily reset boundary at 15:00 local

### Core trading engine
- [x] Dual bracket placement (Long + Short)
- [x] Trade-limit by “sets”
- [x] Price-inside-range gate before re-arming
- [x] Midpoint deferred-start & post-Trade1 quality gate
- [x] Min/Max range width filters
- [x] Cooldowns / per-bar de-dupe latches

### Recoup / Opposite Add
- [x] Immediate opposite-direction add on entry
- [x] Risk-delta sizing to a multiplier target
- [x] Stop distance scaling (percent of original)
- [x] Opposite TP override
- [x] Paired exit when Recoup TP is hit

### News & blocking
- [x] Embedded news rows (date, HHMM, TypeId)
- [x] Severity toggles (1/2/3)
- [x] Per-type policy dropdown (Inherit / Ignore / Delay / All-day)
- [x] Intraday Delay Mode + post-event delay minutes
- [x] “All-day” hard blocks

### Risk & halts
- [x] Daily loss cap (halts new trading)
- [x] Daily profit cap (halts new trading)
- [x] P/L computed from closed trades since 15:00 boundary

### Safety exits & cleanup
- [x] Early exit window (profit-only exit)
- [x] EOS cancel/flatten
- [x] Scheduled emergency re-sweep for desync reduction
- [x] Proxy cancel-sweep / proxy exit support

### Integrations
- [x] TradersPostDeluxe alert/JSON integration (proxy orders + proxy exits)

---

## Suggested roadmap for the “new platform” port

1. **Session engine module**
   - Time-zone anchors, DST-safe window creation, day-boundary.
2. **Range builder**
   - Rolling extrema, range validation, midpoint measurement.
3. **Order engine (state machine)**
   - Bracket sets, trade-limit sets, cooldowns, price-inside-range gate.
4. **News calendar service**
   - Data ingestion (CSV/API), policy resolution, intraday delay math, “resume at” timestamps.
5. **Risk service**
   - Daily P/L aggregation, caps, projected-risk gate for new orders.
6. **Recoup/Proxy module**
   - Risk-delta sizing, proxy mapping (MYM↔YM), paired exits.
7. **Execution & reliability**
   - Idempotent exit/cancel sweeps, webhook retry policy, audit log.
8. **Telemetry/UI**
   - Single coherent “Status + Why” model shared by UI, logs, and gates.

---

## Repo notes

- Script name: **DBO_Prod** (Pine v6).
- External dependency: **`adam_overton/TradersPostDeluxe`** TradingView library for advanced order JSON/alerts.

---

## Disclaimer

This strategy is provided “as-is” for research/automation. Futures trading is risky; use strict risk controls and validate behavior in a paper environment before live deployment.
