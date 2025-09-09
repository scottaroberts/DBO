# Codebase Bloat Analysis & Refactoring Roadmap

## Overview
The migration increased the line count from ~1200 lines to ~1700 lines **while reducing feature scope**. This analysis explains the primary causes of code bloat and presents a structured roadmap to reduce line count, simplify design, and preserve trading correctness.

---

## Why the Line Count Increased

### 1. Observability Scaffolding
- Multiple debug layers: L1/L2/L3 labels, midpoint diagnostics, data-window traces, VWAP toggles.
- Large blocks of repeated plotting/labeling code.
- Adds no trading functionality, only visibility.

### 2. Duplicated Reset Logic
- Start-of-session, post-session, EOS cleanup all contain near-identical reset code.
- Each new edge case created more duplication.

### 3. Opposite-Side & Proxy Complexity
- Opposite reissue + proxy position handling introduces large, symmetric branches (long vs short).
- Paired-TP exits rebuild the same exit-context logic repeatedly.

### 4. Pine v6 Assignment Limitations
- No tuple updates (`:=`), no global writes inside functions.
- Forced verbose, inline assignments rather than small helpers.

### 5. Hard-Coded Data Lists
- Sev-3 calendar is dozens of inline `array.push` calls.
- Pure data, but visually noisy and line-expensive.

---

## Refactoring Roadmap

### Step 1. Split **Core vs Debug**
- Add `buildProfile = input.string("Core", options=["Core","Debug"])`.
- Move/strip debug-only code into a companion “Observer” script.
- Candidates for removal from Core:
  - L2/L3 trace labels.
  - Midpoint diagnostics labels.
  - Data-window echo plots of booleans/counters.
  - VWAP/VWMA optional plots.
  - Session info **table**.
- **Savings:** 250–400 lines.

### Step 2. Replace Sev-3 Calendar
- Use CSV input string: `input.string("2025-06-02,2025-06-04,...")`.
- Parse via `str.split` → `sev3Dates`.
- **Savings:** 30–40 lines.

### Step 3. Exit Context Builders
- Add `f_build_exit_ids(_isLongPos)` to return [thisEntry, thisExit, oppPrim, oppPrimExit].
- Add `f_chart_and_proxy()` to return [chartTkr, proxyTkr].
- Replace repeated ID/ticker rebuilds.
- **Savings:** 60–90 lines.

### Step 4. Proxy Cleanup Helper
- Encapsulate proxy reset logic into a single helper returning cleared tuple.
- Remove inline repeated `if proxySent ... reset` blocks.
- **Savings:** 20–40 lines.

### Step 5. Unified Reset Block
- Compute flags (`doSoftReset`, `doHardReset`) at multiple points.
- Centralize into one final reset block at EOF.
- Eliminates drift and duplication across session resets.
- **Savings:** 40–70 lines.

### Step 6. Direction-Agnostic Opposite Reissue
- Precompute direction-agnostic variables (`baseEntry`, `baseStop`, `oppEntry`, etc.).
- Collapse long/short branches into one unified block.
- **Savings:** 60–100 lines.

### Step 7. Post-Trade Lock Helper
- Encapsulate `waitUntilBar` and `postTradeLock` updates.
- Shrinks repetitive 2–4 line blocks.
- **Savings:** 10–20 lines.

### Step 8. Prune Echo Plots
- Drop plots that only echo booleans/counters (scoped windows, flags, halted state, last trade ID, etc.).
- Keep only plots directly used for **live debugging**.
- **Savings:** 50–100 lines.

---

## Expected Net Reduction
By combining the above steps, we can conservatively expect:
- **450–700 lines saved**.
- Codebase reduced to **~1000–1200 lines** (below the old size) while preserving new safety/session correctness improvements.

---

## What Stays As-Is
- **DST-safe window builder** (already compact, correct).
- **Sizing snapshots / frozen targets** (ensures consistency across bars).
- **Core exit & bracket primitives** (`f_issueBracket`, `f_exit_atomic`).

---

## Development Order (Scoped Patches)
1. **Observability split** → Core vs Debug.
2. **CSV-based Sev-3 calendar**.
3. **Exit-context builders + proxy cleanup helper**.
4. **Unified reset block**.
5. **Direction-agnostic opposite reissue**.
6. **Prune echo plots**.

Testing after each patch ensures correctness and avoids regressions.

---

## Summary
The extra 500 lines came primarily from **debug scaffolding, duplication, and Pine v6 constraints**. By restructuring observability, unifying resets, and compressing exit/proxy logic, we can:
- Cut the codebase back under its original size.
- Keep new safety/risk improvements.
- Maintain modular, testable structure.

