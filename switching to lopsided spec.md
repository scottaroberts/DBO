# Rebaseline Spec — MYM Only (Delta from Current Code)

Based on `DBO Sept 9 4430 after exit edits.txt`

---

## Scope (what changes vs. today)
- **Do not** change the state machine, gating, or 1-phase issuance.
- **Remove all proxy/YM behavior** (inputs, helpers, emissions, state).
- **Keep OppAdd**, but **issue on the chart ticker** (`syminfo.ticker`) with local orders + JSON.
- **Keep Opp TP override** as it already applies to the base ticker.
- **Add**: on first fill, **cancel** the opposite bracket and **emit one Exit JSON** (“CanceledOppositeOnEntry”) for tidy logs.

---

## Minimal patch list (by section / identifier)

### 1) Inputs & globals
**Change / remove usage** (leave inputs temporarily or delete—your call):
- `useProxyRealtimeOnly`, `proxyQtyMul` → **hard-disable** and **stop referencing** everywhere.
- `proxyPosActive`, `proxyTickerPos`, `proxyPosDir`, `proxyPosQty` → **delete** and remove all reads/writes.

**Keep:**
- `oppositeTpOverride` and its flag `oppTPOverrideActive` (already session-frozen).

---

### 2) Exit plumbing
**Function `f_exit_atomic(...)`:**
- **Delete proxy branch** entirely: the conditional block that builds a YM JSON and `alert(js, ...)`. Only **chart-ticker** (`tpd.SendExitAlert(price=...)`) remains.  
- Drop return `proxySent`; keep a single `didExit` boolean (update call sites to ignore proxy).  
- Keep cancel → close → emit ordering invariant.

**Call sites:** In §17.4, §17.7, §17.8, §17.9, and the “session end alert” block, remove all `proxyTkr_*` plumbing and `proxy*` resets. Pass only the chart ticker to `f_exit_atomic`.

---

### 3) OppAdd on base ticker (no proxy helpers)
**Delete helpers:**
- `f_get_proxy_ticker`, `f_send_opposite_on_proxy`, `f_calc_additional_proxy_qty`.  

**Keep & reuse helper:**
- `f_calc_additional_micro_qty(...)` stays for risk-based micro adds on **the base ticker**.  

**Add a tiny replacement helper:**
- `f_send_opposite_on_base(_isLongOpp, _entry, _stop, _tp, _qty)` → calls `f_issueBracket` with `tkr = syminfo.ticker` and `placeLocal = true`.  

---

### 4) §17.2 — Open trade handler (Immediate Opposite Reissue)
**What to keep:** Your **1-phase** detector (`enteredLongNow172/enteredShortNow172`), **Opp TP override** math (`useTpSizeOpp`), distance checks, **risk gates**, and the **Opp TP overlay line** logic.

**What to change:**
- Replace both LONG/SHORT “proxy” branches to:
  - **Size:** compute `qMicroAdd` the same way you do now (risk delta to reach `oppositeQtyMultiplier` of the base leg). **Do not** quantize via `proxyQtyMul`; just use `qMicroAdd`.  
  - **Issue:** call `f_send_opposite_on_base(...)` with `qMicroAdd`.  
  - **Remove** the “+1 full-size if risk allows” bump (that was a YM artifact).  
- **On the same entry pulse**, **cancel** the still-waiting opposite primary bracket and **emit one Exit JSON** with reason `"CanceledOppositeOnEntry"`. Do this **before** issuing OppAdd.  
  - Concretely: if we entered LONG, `strategy.cancel(idShort)` and `strategy.cancel(idShort + "_Exit")`, then `tpd.SendExitAlert(price=close)` with reason tag. Mirror for SHORT.  
  - Keep your `oppTPLineActive` arming as-is after issuance.  

Result: immediate OppAdd remains; the only behavioral change on entry is the required **cancel+exit-json** for the original opposite bracket, and the fact the OppAdd is placed on the chart ticker.

---

### 5) §17.4 — Closed trade handler (Profit vs Stop)
**Profit path:** Keep your current cleanup + **reissue both** baseline brackets logic **unchanged** (this is your “golden” behavior now). It already fits the readme’s state machine. Remove all proxy-exit guard code.

**Stop path:** Keep today’s behavior: clear the stopped side, leave the opposite if not in early window; stand down only in early window. No proxy touches remain.

---

### 6) §17.7 — Paired exit on Opp TP line
- Keep the **paired-exit trigger** on the overlay TP price.  
- Remove all `proxyTkr_*` and `proxy*` state resets. Call the (now chart-only) `f_exit_atomic(...)` once, then drop the overlay and set your re-arm flags exactly like today.

---

### 7) §17.8 / §17.9 / EOS / “Exit Alert Fired”
- Strip proxy arguments and proxy resets; all exits remain chart-ticker only.  
- Leave session resets, snapshots, and cycle state resets **unchanged**.

---

## State machine (no logic changes)
- Use the README state machine verbatim; you’re **not** changing transitions or timers—only which ticker receives OppAdd and exits. All existing `waitUntilBar/postTradeLock/midpoint gating` and **profit→rearm** flow stay the same.

---

## JSON / Alerts
- **Single stream only**: all **entry/exit JSON** now reference **`syminfo.ticker`**.  
- On first fill, emit **one Exit JSON** for the **canceled opposite bracket** (`"CanceledOppositeOnEntry"`).  
- OppAdd emits **Entry JSON** on the base ticker (already handled by `f_issueBracket`).

---

## Safety & invariants (unchanged)
- **Cancel → Close → Emit** ordering retained by `f_exit_atomic`.  
- Risk gates (`f_projectedRiskOK`), daily loss halts, Sev-3 filter, early-exit window, midpoint gating: **unchanged**.

---

## Quick acceptance checklist
- [ ] No references remain to `YM`, `proxy`, `f_get_proxy_ticker`, `proxyQtyMul`, `useProxyRealtimeOnly`, or `proxy*` state.  
- [ ] OppAdd fires on base ticker with same risk math; Opp TP overlay still plots and drives paired exit.  
- [ ] On first fill, opposite **primary** bracket is **canceled** and **one Exit JSON** is emitted.  
- [ ] All exits/alerts are **single-ticker** and the UI/debug labels behave as before.  
- [ ] README state machine behavior is intact.
