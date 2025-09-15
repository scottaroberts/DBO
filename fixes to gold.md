Got it — we’ll keep the Gold trading pattern (keep the original base bracket; OppAdd is sized/placed “in consideration” of that), and we’ll only back-port fixes that improve live trading correctness/robustness (not refactors or cosmetic logging).

Here’s a ranked punch-list to pull from the Sept 14 “deferred entry” code into Gold, minus the parts that changed the pattern (e.g., deferred OppAdd, cancel-opposite-on-entry) — those we’ll skip.

Priority 0 — trading failure / correctness

Intra-bar entry hardening + qty snapshot for OppAdd sizing
Capture “position opened this bar” even if it happens mid-bar, and snapshot the entry qty so OppAdd sizing is correct on the very bar we flip. Back-port the tiny state block (posAtBarOpen…, seenNonZeroThisBar…, entryQtySnap…) and the merge into enteredLongNow172/enteredShortNow172. This prevents under- or zero-sized OppAdds on fast ticks.

Intra-bar flip fallback for OppAdd direction
If our “entry pulse” misses but we’re already in a position by the time logic runs, derive dirOpp172 from posNow172. This saves OppAdd on same-bar flips.

Stop-out path = NO-OP (don’t flatten/retire the opposite)
On a true SL exit, do nothing beyond the strategy’s own close — keep the opposite/recoup logic intact. This avoids tearing down live recovery paths after a stop. Back-port the “true stop = no-op; profit-gated case only does atomic flatten” split.

Exit de-dupe via lastExitEmitBar everywhere we emit
Gold already does this broadly; verify the same single-bar de-duplication guard is in all paths (profit/paired/early/EOS/emergency) after we carry items 1–3 over.

Priority 1 — robustness / state hygiene

Opposite-cycle accounting (cycleSide / oppOutstanding) fully consistent
Gold already has this; confirm the same “allowProfitRearm only when the same side closed or the opposite is fully cleared” gate is intact after back-ports. This prevents premature re-arming.

Profit re-arm gating mirrors Gold’s snapshot logic
Keep the “midpoint + early-window + trade-limit + session-valid” composite gate exactly as in Gold when re-issuing both brackets after profit; just ensure items 1–2 feed it correctly.

Paired-TP overlay exit stays atomic & single-latch
Ensure the paired-exit clears overlay, sets short locks (waitUntilBar/postTradeLock) and re-arm flags the same way after back-porting #1–2 so we don’t double-emit or re-arm too early.

Priority 2 — safe QoL (optional, non-pattern)

Commission-aware P/L labels (display-only)
Carry over the commPerSide input and net-P/L label math (no effect on trading).

Long-side qty rounding to 10s (policy)
If you want identical sizing to recent tests, back-port the “round LONG base qty to nearest 10 (min 10)” tweak; otherwise skip to preserve original sizing policy.

Explicitly do not back-port (pattern-changing)

Deferred OppAdd (N+1 bar) switch/armers — this was the experiment that failed; leave Gold’s immediate OppAdd behavior.

Cancel-opposite-on-entry atomic — Gold keeps the original bracket in play; do not import that cancel.

Proxy YM routing & proxy-only exits — Gold is chart-ticker only; leave the proxy stack out.

If this priority order looks right, I’ll prep a minimal patch set that only touches the affected §17.2/§17.4 scaffolding and the tiny helper state — no refactors, no renames, and zero behavior drift beyond the fixes above.
