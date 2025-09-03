# DBO Simplification Spec (MVP) — v2

> **Goal:** Reduce branches and errors by keeping only the features you actively use. Lock to one opposite-side policy, a tight set of quality gates, and the impactful Early-Exit behavior. Retain YM proxy adds. Remove ATR & VWAP stop types and unused filters. Keep VWAP/VWMA plots for visual context.

---

## 1) Scope & Definitions
- **Primary instrument:** MYM (micro) for base brackets.
- **Proxy instrument:** YM (full-size) for opposite-side add via proxy alerts/tickets.
- **No quantity parity:** We **do not** attempt to match micro qty on opposite; we **add** scaled YM using `oppositeQtyMultiplier` and risk-delta sizing.

---

## 2) Behavioral Rules (MVP)

### 2.1 Opposite-Side Handling (single policy)
- **Only mode:** **Keep Opposite (Scaled Qty + Adjusted Entry)**.
- We always leave the original opposite micro bracket in place **and** add a full-size YM in the opposite direction per risk delta.
- **Timing:** **Immediate on the entry bar (Bar N)** only. Remove all N+1 delay/pend logic.
- **Placement:** Compute YM add entry/TP from the configured offsets & overrides; no special cancel phase.

### 2.2 Quality & Session Gates (kept)
- **Midpoint distance filter** (start proximity for deferred start; runtime “return after Trade 1”).
- **Min Range** / **Max Range** (range-width checks).
- **Sev-3 New Day** (daily block on configured dates).
- **Early Exit** window is authoritative: **no new risk** during this window; cancel pending.

### 2.3 Order Emission (centralized)
- **Single entry sink**: All primary brackets are placed via `f_placeBracket(...)` (canonical path).
- **Single exit sink**: All exits & cancels go through `emitExit(reason, price)` (one doorway) which handles: strategy closes/cancels; single proxy-exit JSON; local flag resets and cooldown scheduling.

---

## 3) Inputs & Features — Keep vs Remove

### 3.1 Keep (MVP)
- **Opposite behavior:** Hard-fixed to **Keep Opposite (Scaled Qty + Adjusted Entry)**.
- **Opposite scaling & placement:** `oppositeQtyMultiplier`, `oppositeReissueEntryOffsetPts`, `oppositeTpOverride`.
- **Opposite timing:** Immediate on Bar N (implicit; no toggle).
- **Proxy routing:** `useProxyRealtimeOnly`, `proxyQtyMul` and all proxy ticket/JSON paths.
- **Risk/Halts:** `tradeLimit`, `maxDailyLoss`, per-session risk calculation.
- **Quality gates:** Midpoint (start proximity + return-after-Trade1), Min/Max range, Sev-3 New Day.
- **Early Exit**: no new risk during early window; cancel pending.
- **Plots:** Keep **VWAP/VWMA** and range/midpoint visuals.

### 3.2 Remove / Deprecate (MVP)
- Opposite modes: **Delete** “Cancel Opposite” and “Keep Opposite (Same Qty)”.
- Opposite delay: **Delete** Bar N+1 scheduling & pend logic; always immediate.
- **Stop types**: **Delete** `ATR` and **delete** `VWAP` stop types; keep only `RR` and `Points`.
- Filters: **Delete** VWAP Distance filter, EMA Slope filter, RSI filter (keep optional overlay plots if desired).
- VWAP config inputs: **Remove** Anchor/Length/Offset inputs (plots remain).

**Resulting `slType` options:** `RR`, `Points`.

---

## 4) Engine Flow (MVP)
1) **Window & Early Exit** guards (no new risk inside early window; cancel pending).
2) **Halts/blocks** (tradeLimit, daily loss, Sev-3).
3) **Quality snapshot** (see §5; only Midpoint start proximity, Min/Max range, Sev-3 considered for session validity).
4) **Primary placement** via `f_placeBracket(...)`.
5) **On fill → YM opposite add** (scaled; immediate; via proxy wrapper).

---

## 5) Snapshot “Non‑Wiggly” Values (Authoritative at Breakout)

### 5.1 Session Filter Snapshot (first bar of breakout)
- Compute **once** at the first bar of breakout:
  - `sessionValid` = Midpoint start proximity OK **and** Min/Max range OK **and** not Sev‑3 Day.
  - `sessionInvalidReason` = first blocking reason string.
- **Freeze** `sessionValid` (+ reason) for the rest of the session; do **not** re-evaluate moving indicators later.
- Runtime-only exception: **Midpoint “return after Trade 1”** remains a live event condition after entry.

### 5.2 Risk/Sizing Snapshot (first bar of breakout)
- Freeze post‑range TP/SL sizes and ratios used for placements for the remainder of the session (even though ATR/VWAP stop types are removed, this avoids mid-session drift from any other recalculation).

---

## 6) Opposite YM Add — Rules
- **Trigger:** Flat → Position transition confirmed.
- **Gates:** In breakout window; range defined; not Early Exit; sessionValid; not halted; within tradeLimit/daily loss.
- **Sizing:** Compute **additional** full‑size YM via risk delta targeting `oppositeQtyMultiplier` of base risk; enforce daily‑loss guard on the **additional** risk.
- **Emission:** Use the proxy send wrapper; set TP/SL per MVP stop type and overrides; leave original micro bracket intact.

---

## 7) Data & Plots (MVP)
- Keep **VWAP/VWMA** lines and **Range/Min/Max/Midpoint** diagnostics.
- Remove Data-Window statuses for deleted filters to avoid confusion.

---

## 8) Implementation Plan — Anchors & Actions (No code yet)
> The items below list **approximate line numbers** (based on the current file), an **anchor text** to locate the section, and explicit **actions**. Code snippets will be delivered in the patch PR.

### 8.1 Stop Types → Keep only `RR`, `Points`
- **Approx lines:** ~820–980
- **Anchor:** `14) FLEXIBLE STOP LOSS OPTIONS`
- **Action:** Replace switch/if arms to remove `VWAP` and `ATR` branches; update `slType` input options to `["RR","Points"]` and delete snap variables tied solely to ATR/VWAP. Remove any references to `snapAdjTPSize`, `snapAdjPtsSL` that are ATR/VWAP‑specific.

### 8.2 Opposite Modes → Collapse to single policy
- **Approx lines:** ~1180–1260
- **Anchor:** `17) OPPOSITE SIDE HANDLING`
- **Action:** Remove option selector for the three behaviors; hard‑fix to **Keep Opposite (Scaled Qty + Adjusted Entry)**. Delete the “Cancel Opposite” and “Keep Same Qty” branches and their UI inputs. Keep `oppositeQtyMultiplier`, `oppositeReissueEntryOffsetPts`, `oppositeTpOverride`.

### 8.3 Remove N+1 Reissue Delay
- **Approx lines:** ~1260–1330
- **Anchor:** `17.2) PHASE 2 REISSUE SCHEDULER`
- **Action:** Delete bar‑delay logic and any `pendBarXXX`/`oppositeImmediateIssue` toggles. Always schedule the YM add for **Bar N**. Remove inputs tied to the delay.

### 8.4 Centralize Entry Emissions
- **Approx lines:** ~1040–1175
- **Anchor:** `16) BRACKET PLACEMENT / f_placeBracket`
- **Action:** Ensure **all** primary placements call `f_placeBracket(...)`. Remove or redirect any alternate direct `strategy.entry/exit` calls, JSON emitters, or side doors that place entries. Keep a single return payload (qty, ids, comment) for downstream logging.

### 8.5 Centralize Exit Emissions
- **Approx lines:** ~1335–1500
- **Anchor:** `17.7) EARLY EXIT / EOS / EMERGENCY`
- **Action:** Introduce a single `emitExit(reason, price)` that wraps: `strategy.close_all/strategy.cancel`, a **single** proxy exit JSON, and local resets (proxy flags, TP line handle, cooldown clocks). Replace calls in: profit‑exit handler, early‑exit handler, end‑of‑session cleanup, emergency/kill‑switch, paired‑TP exits.

### 8.6 Quality Filters Snapshot at Breakout
- **Approx lines:** ~690–770
- **Anchor:** `11) QUALITY FILTERS & SESSION VALIDATION`
- **Action:** At **first bar of breakout**, compute once and freeze: `sessionValid` and `sessionInvalidReason` using **only** Midpoint start proximity, Min/Max range, Sev‑3. Remove live rechecks later. Preserve the runtime Midpoint return‑after‑Trade1 check separately.

### 8.7 Risk/Sizing Snapshot at Breakout
- **Approx lines:** ~770–820
- **Anchor:** `12) POST‑RANGE SNAPSHOTS`
- **Action:** Freeze post‑range TP/SL sizes and any ratios needed for brackets throughout the session. Remove ATR/VWAP‑specific snap calculations.

### 8.8 Remove Unused Filters
- **Approx lines:** ~600–690
- **Anchor:** `10) OPTIONAL FILTERS`
- **Action:** Remove inputs, calculations, and gates for: VWAP Distance, EMA Slope, RSI. Keep any **plot‑only** overlays if desired, but detach them from gating.

### 8.9 Keep Plots, Drop Plot‑Config Inputs
- **Approx lines:** ~540–595
- **Anchor:** `9) VWAP / VMA PLOTS`
- **Action:** Keep VWAP/VWMA plots; delete unused inputs (Anchor/Length/Offset). Ensure plots reference session values or built‑ins without exposing unused config.

### 8.10 Early‑Exit Strictness
- **Approx lines:** ~1320–1400
- **Anchor:** `17.4) EARLY EXIT WINDOW`
- **Action:** Ensure Early Exit window **forbids** new placements and cancels pending. Route all exit effects through `emitExit(...)` to prevent duplicates.

---

## 9) Test Scenarios (regression harness)
1) Valid session → flat → **first bracket** only after range & snapshot pass.
2) **Filters invalid** at breakout → no placements for the whole session (snapshot holds).
3) Stop‑out long → **no same‑bar** reissue via side paths; YM add only through the single policy on entry.
4) Profit exit → midpoint rule enforced; uniform cooldown applies; exits centralized.
5) Early window → **no new risk**; pending cancels; exit centralized.
6) Extremely narrow or oversized range → blocked by Min/Max.
7) Daily loss crossed mid‑session → immediate halt; no emissions.
8) Proxy add TP hits → paired exit routed through `emitExit(...)` once.

---

## 10) Future (post‑MVP)
- **Compact telemetry panel**: state (Idle/Armed/InPosition/Lockout/Halted/Ended), first blocking reason, trades used/limit, daily PnL vs cap, last action string. (Current labels stay for MVP.)

---

## 11) Migration Checklist
- [ ] Remove ATR & VWAP stop types; `slType ∈ {RR, Points}` only.
- [ ] Remove opposite modes except **Keep Opposite (Scaled Qty + Adjusted Entry)**.
- [ ] Delete N+1 reissue delay path; always immediate.
- [ ] Centralize **all** entries via `f_placeBracket(...)`.
- [ ] Centralize **all** exits via `emitExit(reason, price)`; remove parallel exit paths.
- [ ] Implement **breakout snapshots** for session validity and sizing; no rechecks later.
- [ ] Remove VWAP Distance / EMA Slope / RSI filters and their inputs.
- [ ] Keep VWAP/VWMA plots; remove unused plot config inputs.
- [ ] Preserve Early‑Exit strictness & proxy exit JSON paths.

## 12) Stuff to come back to
1) oppTPOverrideActive, given the hardcoding of options we no longer should need this override. 
