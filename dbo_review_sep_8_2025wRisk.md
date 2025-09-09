# DBO Migration Review — Unified Spec Compliance & State Machine Map

**Date:** 2025-09-08 (America/Los_Angeles)

**Artifacts Reviewed**
- **Implementation:** `Sep 8_working`
- **Original Reference:** `GoldenCode`
- **Specs:** `dbo_simplification_spec_mvp_v_2.md` (MVP), `Readme.md`

> Goal: Provide a single, durable reference that consolidates the gap/spec analysis, a policy matrix with call‑site mapping, and a cycle timeline/state dictionary. No code changes are proposed here.

---

## 1) Executive Summary

**High confidence spec alignment**
- **Stop types simplified** to only `RR` and `Points` (ATR/VWAP/trailing removed per MVP).
- **Opposite-side policy** simplified to a **single mode**: *Keep Opposite (Scaled Qty + Adjusted Entry)*; issuance on **Bar N** (no N+1 pend queues).
- **Central sinks** in place: all entries flow through `f_placeBracket(...)`; all exits through `f_exit_atomic(...)` (cancel → close → emit), including EOS and Early-Exit.
- **Proxy routing for OppAdd** supported (e.g., MYM → YM) with sizing multiplier; paired cleanup via atomic exit.
- **Breakout freeze snapshots** implemented for session validity and TP/SL levels.
- **Early-Exit window** enforces *no new risk* and *cancel pending* before any emit.

**Items to verify or tighten**
- **OppAdd single-shot per cycle**: add/confirm an explicit latch (e.g., `oppAddIssuedThisCycle`) independent of TP-line handle state; preserve a small **opposite‑outstanding counter** and decrement on opposite completions.
- **Profit path cancellation policy**: ensure the Profit handler passes *cancel both opposing orders (original opposite + OppAdd)* into `f_exit_atomic(...)` before re‑arm.
- **OppAdd gate scope**: make intent explicit that OppAdd **ignores MidReturn by default**, via a clear `gates_for_oppadd()` (or equivalent) feeding the send wrapper.

---

## 2) State Dictionary (Spec → Implementation)

### 2.1 Gates & Timing
- **Midpoint gate (deferred start, and re‑arm after first trade profit)** → `requireMidpointReturn`, `midpointStartTolerancePct`.
- **Range width** → `minRangePoints`, `maxRangePoints`.
- **Sev-3 day block** → `useSev3Filter` anchored via session date (derived from `rangeEnd`).
- **Early-Exit window (no new risk + cancel pending)** → `earlyExitMinutes` + `f_exit_atomic(...)` invoked with cancel‑first semantics.

### 2.2 Session Window & Snapshots
- **Anchors** → `rangeStart`, `rangeEnd`, `exitTime`.
- **Validity snapshot** → `sessionSnapTaken`, `sessionValidFrozen`, `sessionInvalidReasonFrozen` (taken on first breakout bar).
- **Sizing snapshot** → `sizingSnapTaken`, `sLongEntry`, `sShortEntry`, `sLongStop`, `sShortStop`, `sLongTP`, `sShortTP` (frozen at breakout).

### 2.3 Central Sinks
- **Entry sink** → `f_placeBracket(entryId, side, qty, entryPx, slPx, tpPx, ...)` (internally calls `f_issueBracket(...)`).
- **Exit sink** → `f_exit_atomic(reason, comment, price, timeMS, tz, chartTkr, proxyTkr, ..., doCancelAllBefore, emitLabel, thisEntryId, thisExitId, oppPrimaryId, oppPrimaryExitId, allowChart, allowProxy)`.

### 2.4 Opposite‑Side (Single Policy; Bar N)
- **Scaling & placement** → `oppositeQtyMultiplier`, `oppositeReissueEntryOffsetPts`, `oppositeTpOverride`.
- **Proxy OppAdd** → `useProxyRealtimeOnly`, `proxyQtyMul`; implemented via `f_send_opposite_on_proxy(...)` and `f_get_proxy_ticker(...)`.
- **Deterministic IDs** → `OppAddL_<bar_index>`, `OppAddS_<bar_index>` (for cancels/traceability).
- **Visual overlay** → `oppTPLineActive`, `oppTPLineIsLong`, `oppTPLinePrice`.

### 2.5 Limits, Halts, Risk Checks
- **Trade/set limit** → *UI and counters differ from GoldenCode*: limit is interpreted as **sets** (pair issuance) rather than individual trades; confirm product requirements and UI copy.
- **Projected‑risk guard** → daily max loss gating via projected‑risk calculation before any new bracket/OppAdd.
- **Profit‑halt** → `haltAfterProfit`.

### 2.6 Stops & TP (Simplified per MVP)
- **Stop types** → `slType` ∈ {`RR`, `Points`} only.
- **RR/Points math** → computed at placement; then enforced via sizing snapshot.

### 2.7 Data & Plots
- **Keep VWAP/VWMA visuals** (no user config inputs per MVP cleanup).
- **Range box / midpoint** visuals & data‑window diagnostics preserved.

---

## 3) Policy Matrix → Call‑Site Map

> Legend: **Spec** describes required behavior; **Call‑site** shows how `Sep 8_working` should (or does) invoke `f_exit_atomic(...)` and related helpers. *Observed* reflects what’s present; *To verify* marks areas that need confirmation.

### 3.1 PROFIT (First trade hits TP)
**Spec**
1) Cancel **both** opposing orders (original opposite + any OppAdd).
2) Close filled side if still open (some brokers require explicit close even after TP fill event).
3) Emit exit alert(s) once flat.
4) Rearm per gates (often requires MidReturn for the *next* set).

**Call‑site (expected)**
```text
f_exit_atomic(
  reason = "PROFIT",
  comment = "Profit cleanup",
  price = <tp fill price or last>,
  timeMS = time_close,
  tz = <tzInput>,
  chartTkr = syminfo.tickerid,
  proxyTkr = <proxy symbol if active>,
  ...,
  doCancelAllBefore = true,      // cancel opposite primary + OppAdd children
  emitLabel = true,
  thisEntryId = <active side entry id>,
  thisExitId  = <exit id>,
  oppPrimaryId = <opp entry id>,
  oppPrimaryExitId = <opp exit id>,
  allowChart = true,
  allowProxy = true
)
```
**Observed**: Early‑Exit/EOS pass cancel‑first; Profit path shows resets + rearm, but explicit `doCancelAllBefore = true` at the Profit call‑site is **not conclusively visible** in current notes.

**To verify**: Profit handler uses `doCancelAllBefore = true` and includes **both** `oppPrimaryId` and any **OppAdd IDs** in cancel scope before re‑arm.

---

### 3.2 STOP (First trade hits SL)
**Spec**
1) Keep the opposite side (original opposite) **working** per single‑policy design.
2) If an OppAdd was issued earlier, keep it if still valid; otherwise, ensure cancel semantics match the decision tree.
3) Do **not** rearm a new set immediately; rearm only when **opposite outstanding** drains to 0 (i.e., when those orders exit or are canceled by rules), and gates allow.

**Call‑site (expected fragments)**
```text
// No global cancel‑all. Close the stopped side if needed, emit exit.
f_exit_atomic("STOP", "Stopped side cleanup", <price>, time_close, tz, chartTkr, proxyTkr, ..., false, true, thisEntryId, thisExitId, na, na, true, true)

// Maintain counters:
oppOutstanding := oppOutstanding   // unchanged here; decremented when opposite legs resolve
// Rearm: only when oppOutstanding == 0 and gates_ok_for_rearm == true
```
**Observed**: Variables exist (`oppOutstanding`, OppAdd IDs); decrement/zero‑test **not explicitly confirmed** at opposite completion sites.

**To verify**: Ensure decrement on opposite TP/SL completion, and rearm gate tied to `oppOutstanding == 0`.

---

### 3.3 EARLY‑EXIT window
**Spec**
1) **No new risk** during Early-Exit window.
2) **Cancel pending** orders.
3) Close any open positions and then **emit exit**.
4) Do **not** immediately rearm; follow gates.

**Call‑site (observed)**
```text
f_exit_atomic(
  reason = "EARLY_EXIT",
  comment = "Early window cleanup",
  price = <last>,
  timeMS = time_close,
  tz = <tzInput>,
  chartTkr = syminfo.tickerid,
  proxyTkr = <proxy>,
  ...,
  doCancelAllBefore = true,   // ✓ required by spec
  emitLabel = true,
  ...
)
```
**Observed**: Matches spec; gates check prevents new risk.

---

### 3.4 EOS (End‑of‑Session)
**Spec**
1) Cancel all working orders.
2) Flatten positions.
3) Emit exit.
4) Reset OppAdd visuals/handles and cycle state.

**Call‑site (observed)**
```text
f_exit_atomic("EOS", "End of session", <last>, time_close, tz, chartTkr, proxyTkr, ..., true, true, thisEntryId, thisExitId, oppPrimaryId, oppPrimaryExitId, true, true)
```
**Observed**: Implemented via atomic exit with cancel‑first; OppAdd / proxy cleanup performed.

---

### 3.5 OppAdd (Opposite Add on Bar N)
**Spec**
- Issue **once per cycle** on fill bar; ID as `OppAddL_<bar_index>` / `OppAddS_<bar_index>`.
- **Proxy** allowed in realtime with `proxyQtyMul`.
- **Gates**: OppAdd has its own scope; **MidReturn disabled by default**.

**Call‑site (observed/expected)**
```text
// Gate result passed in (_blocked)
f_send_opposite_on_proxy(
  _blocked = <gates_for_oppadd()>,
  side = <opposite side>,
  id   = OppAdd[LS]_fmt(<bar_index>),
  qty  = f_calc_additional_proxy_qty(...),
  entryPx = <adjusted>,
  tpPx    = <override or derived>,
  slPx    = <derived>,
  allowProxy = useProxyRealtimeOnly ? barstate.isrealtime : true
)
```
**Observed**: Wrapper and proxy mechanics present; explicit `gates_for_oppadd()` function **not surfaced**—intent likely encoded elsewhere.

**To verify**: Add/confirm an explicit OppAdd gate function with MidReturn off by default.

---

## 4) Cycle Timeline (Phases & Key Vars)

> Text timeline for Bar‑by‑Bar behavior, including the spec‑driven snapshots and re‑arm rules.

1) **Pre‑Session (Before `rangeEnd`)**
   - Compute box (`rangeStart`, `rangeEnd`), midpoint, range width.
   - Enforce static gates: Sev‑3 block, min/max range.

2) **Window Open (At/after `rangeEnd`)**
   - **Deferred start** if `requireMidpointReturn = true` and price not near midpoint.
   - **First breakout bar** triggers **snapshots**:
     - Validity: `sessionSnapTaken = true`; `sessionValidFrozen`/`sessionInvalidReasonFrozen` set.
     - Sizing: `sizingSnapTaken = true`; `sLong/ShortEntry/Stop/TP` captured.

3) **Primary Entry Placement**
   - All placements via `f_placeBracket(...)`.
   - Projected‑risk gate must pass (daily max loss guard).

4) **Live Position → Outcome A: PROFIT**
   - TP tagged on the filled side.
   - **Policy matrix** (Profit): cancel opposing orders (original opposite + OppAdd), flatten residuals, emit exit, **then re‑arm** (subject to gates, typically MidReturn for new set).
   - **Reset visuals**: `oppTPLineActive := false`, clear OppAdd handles.

5) **Live Position → Outcome B: STOP**
   - SL tagged on the filled side.
   - Opposite side(s) remain **working** per single‑policy design.
   - Maintain `oppOutstanding` counter for opposite legs; **rearm only when `oppOutstanding == 0`** and gates pass.

6) **OppAdd (Bar N)**
   - Single issuance per cycle; IDs `OppAddL_/OppAddS_` set; overlay TP line set via `oppTPLine*` vars.
   - **Proxy** issue allowed in realtime; paired cleanup via atomic exits.

7) **Early‑Exit Window**
   - **No new risk**; invoke `f_exit_atomic(..., doCancelAllBefore = true, ...)` to cancel/close/emit; no immediate re‑arm.

8) **EOS**
   - Invoke `f_exit_atomic("EOS", ..., doCancelAllBefore = true, ...)`.
   - Reset OppAdd overlays; reset counters/latches for next session.

---

## 5) Detailed Gap List & Recommendations (No code changes yet)

1) **OppAdd single‑shot latch per cycle**
   - **Gap**: No explicit `oppAddIssuedThisCycle` independent of TP‑line handle; rare edge cases could double‑issue.
   - **Recommendation**: Add a cycle‑scoped boolean latch that resets only on cycle rollover (EOS/Profit‑rearm/Stop‑drain‑to‑0), and gate OppAdd send on `not oppAddIssuedThisCycle`.

2) **Profit path: cancel scope**
   - **Gap**: Ensure Profit handler passes `doCancelAllBefore = true` and includes **both** the original opposite **and** any OppAdd IDs in cancel targets prior to re‑arm.
   - **Recommendation**: Audit Profit call‑site; align with Early‑Exit/EOS semantics.

3) **OppAdd gates → explicit function**
   - **Gap**: Lack of an explicit `gates_for_oppadd()` separation; MidReturn should be **disabled by default** for OppAdd.
   - **Recommendation**: Surface a small gate function returning `_blocked` for the send wrapper; document that MidReturn is excluded unless deliberately enabled.

4) **Opposite‑outstanding counter lifecycle**
   - **Gap**: Decrement and zero‑test on opposite completion not conclusively located.
   - **Recommendation**: Ensure decrements on opposite TP/SL events, and re‑arm gating uses `oppOutstanding == 0`.

5) **Trade vs Set terminology**
   - **Gap**: UI/panel labels and limits now reflect **sets**; confirm product copy/spec references say **“Max Sets”** to avoid confusion with “Max Trades”.
   - **Recommendation**: Align README/inputs/panel text; note analytics impact.

---

## 6) What to Keep (Improvements Worth Preserving)
- **Session‑anchored Sev‑3 dating** (robust across midnight).
- **Unified atomic exit** logic for all cleanup pathways, including proxy handling.
- **Breakout freeze** of validity and sizing to prevent drift.
- **Immediate Bar‑N OppAdd** with deterministic IDs and proxy support.

---

## 7) Appendices

### A) Key Identifiers & Conventions
- **Entry IDs**: `LongEntry_<bar_index>`, `ShortEntry_<bar_index>` (or equivalent existing convention).
- **OppAdd IDs**: `OppAddL_<bar_index>`, `OppAddS_<bar_index>`.
- **Exit IDs**: `Exit_<bar_index>` per side; proxy exits mirror chart IDs with proxy ticker.
- **OppAdd overlay**: `oppTPLineActive`, `oppTPLineIsLong`, `oppTPLinePrice`.

### B) Inputs (Expected Post‑MVP)
- `slType` ∈ {`RR`, `Points`}, `rr`, `tpPoints`, `slPoints`.
- `oppositeQtyMultiplier`, `oppositeReissueEntryOffsetPts`, `oppositeTpOverride`.
- `useProxyRealtimeOnly`, `proxyQtyMul`.
- `requireMidpointReturn`, `midpointStartTolerancePct`.
- `minRangePoints`, `maxRangePoints`, `useSev3Filter`.
- `earlyExitMinutes`, `haltAfterProfit`.
- Risk/limit inputs including daily max loss and set/trade limit (terminology to be confirmed as **sets**).

---

## 8) Next Steps (if we choose to implement fixes)
1) Add `oppAddIssuedThisCycle` latch and wire into OppAdd send wrapper.
2) Audit Profit call‑site; enforce `doCancelAllBefore = true` with full opposite scope.
3) Introduce `gates_for_oppadd()` (MidReturn off by default); document.
4) Confirm and finalize **set‑based** limits and UI copy.

> This document is intended to live in the repo (Markdown) and act as the authoritative migration review. Once the above “verify/tighten” items are addressed (or explicitly despecified), we can mark the MVP migration **complete**.



---

## 9) Risk Assessment of Identified Gaps (Trade Risk & Residual Controls)

> Scope: Evaluate how each gap could leave live orders without another protective layer (beyond Early-Exit/EOS), and prioritize by net trade risk.

### 9.1 Summary Table

| # | Gap | Primary Failure Mode | Exposure Window (before EOS/Early) | Existing Protective Layers | Severity | Likelihood | Detectability | Priority |
|---:|---|---|---|---|---|---|---|---|
| 1 | **OppAdd single-shot latch per cycle** absent | Duplicate OppAdd in same cycle → unintended stacked exposure (over-sized opposite) | Until TP/SL/Manual/EOS; Early-Exit does not trigger by itself | Projected-risk check (if applied to OppAdd); set/daily-loss limits | **High** | Low–Med | Med | **1** |
| 2 | **Profit path may not cancel both opposing orders** | After Profit, stale opposite primary and/or OppAdd remain working while new set may re-arm → surprise fills later | Persists post-TP until canceled or fills occur | Early-Exit/EOS cancel; projected-risk on new set | **High** | **Med** | Med | **1** |
| 3 | **OppAdd gates not explicitly scoped (MidReturn default unclear)** | OppAdd issued when entry gates would otherwise block (extra risk) **or** OppAdd suppressed when intended (missed hedge) | If allowed improperly: until TP/SL/Manual; if suppressed: opportunity loss (no added risk) | Projected-risk guard; set/daily-loss limits | **Med** (if mis‑allowed) | Med | Med–High | **3** |
| 4 | **Opposite‑outstanding counter lifecycle (decrement/zero‑test)** not confirmed | (A) Rearm too early while opposite orders still live → two concurrent exposures; (B) Never rearm → opportunity loss | Case (A): persists until opposite drains | Projected-risk guard; set/daily-loss limits | **Med–High** (if A) | Low–Med | Low–Med | **2** |
| 5 | **Trade vs Set terminology** mismatch | User sets limit expecting “trades” but engine enforces “sets” → unintended number of orders | Session-long until user corrects input | Daily max loss; manual oversight | Low–Med | Med | High | **4** |

> **Priority legend:** 1 (highest) → 4 (lower). Severity is P&L impact if triggered; Likelihood based on observed code paths & typical runtime states; Detectability is how quickly a user would notice without special logs.

### 9.2 Narrative Risk Analysis

**(1) OppAdd single‑shot latch per cycle**  
- *Why it matters:* Without an explicit `oppAddIssuedThisCycle`, edge cases (e.g., overlay/handle resets, re‑entry of send path on bar updates) can produce **multiple OppAdds** in one cycle. Even if each add passes a projected‑risk check, the **aggregate** exposure may exceed intended.
- *What still catches it:* Daily max‑loss cap can stop additional placements; EOS flattens later; Early‑Exit cancels only when that window is active. Between those, **live stacked orders** can persist.
- *Net risk:* **High severity**, Low–Med likelihood. **Priority 1.**

**(2) Profit path cancellation scope**  
- *Why it matters:* After a TP, **all opposing orders** (original opposite + any OppAdd) should be canceled **before** any re‑arm. If not, those stale opposing orders can fill unexpectedly later, potentially alongside a re‑armed set, creating **unintended hedges/double exposure**.
- *What still catches it:* EOS/Early‑Exit only; otherwise orders remain working.
- *Net risk:* **High severity**, **Med likelihood** because Profit events are common. **Priority 1.**

**(4) Opposite‑outstanding counter lifecycle**  
- *Why it matters:* After a **STOP**, spec requires delaying re‑arm until opposite outstanding drains to 0. If decrement/zero‑test is missing or misplaced, you could **re‑arm too early** while opposite orders are still live (double risk). The opposite failure mode (never re‑arm) is an opportunity cost, not a risk to funds.
- *What still catches it:* Projected‑risk guard may still allow re‑arm if risk budget remains; daily loss/limits provide outer bounds. No automatic cancel beyond EOS/Manual.
- *Net risk:* **Med–High severity** (for early re‑arm case), Low–Med likelihood. **Priority 2.**

**(3) OppAdd gates explicit scope (MidReturn default)**  
- *Why it matters:* OppAdd should **ignore MidReturn by default** (per spec). If it accidentally inherits Entry gates, you might miss intended OppAdds (reduced hedge) or, conversely, allow OppAdd when the **session is otherwise blocked**, increasing exposure.
- *What still catches it:* Projected‑risk, set/daily‑loss limits.
- *Net risk:* **Medium severity** when mis‑allowed; **Priority 3.**

**(5) Trade vs Set terminology**  
- *Why it matters:* Misunderstanding limits can lead to **too many** individual orders compared to user expectation. Usually bounded by daily loss cap; does not create uncontrolled orders by itself.
- *Net risk:* Operational/UX risk; **Priority 4.**

### 9.3 Recommended Risk‑Driven Fix Order (No Code Yet)
1) **Profit cancel semantics** — confirm `doCancelAllBefore = true` and ensure both **original opposite** and any **OppAdd IDs** are targeted.  
2) **Single‑shot OppAdd latch** — add cycle‑scoped latch and reset points (Profit/EOS/Stop‑drain‑to‑0).  
3) **Opposite‑outstanding lifecycle** — wire decrement on opposite completions; re‑arm only at zero.  
4) **OppAdd gates function** — explicit scope with MidReturn disabled by default.  
5) **Terminology alignment** — clarify UI and README to **Max Sets**.

