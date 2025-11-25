# simple3.html audit notes

## Issues
1. **Join window upper bound not validated against resolve window (lines 1182-1193).** The UI clamps join mins to 5–43200 and resolve mins to 30–43200, but it never checks that the resolve window exceeds the join window before calling `createBet`. If the contract requires `resolveWindowSeconds > joinWindowSeconds`, `createBet` will revert with user-provided values. Fix: add a guard such as `if (rm <= jm) throw new Error('Resolve deadline must exceed join deadline');` before the transaction, or auto-bump `rm` above `jm`.

2. **Join action ignores join window expiry (lines 1219-1235).** Before calling `joinBet`, the script only checks opponent presence. If the join deadline has elapsed, `joinBet` will revert and the UI just surfaces the revert data. Fix: compare `b[5]` (join deadline timestamp) to `Date.now()/1000` and show a friendly error before sending the transaction.

3. **Refund button lacks state/deadline checks (lines 1283-1294).** The UI calls `refund` unconditionally. If the contract only allows refunds after deadlines or specific states, the call will revert. Fix: gate the button by reading `getBet` and verifying the current time exceeds the refundable deadline and the bet is still unresolved, otherwise display a local error.

4. **Finalize action ignores state/vote prerequisites (lines 1299-1312).** `doPayout` calls `resolve` without ensuring both votes are in or that deadlines allow resolution. This likely causes revert data when prerequisites are unmet. Fix: check `b[7]`/vote fields after `getBet` and only send when the contract is resolvable.

5. **Network indicator can show connected even on wrong chain (lines 1142-1156).** On any error in `connect()`, the UI sets status to "Not connected" but leaves any previous `badge` styling intact. More importantly, `ensureBase()` only runs once; if the user later switches networks externally, `getContract()` will still proceed and transactions will revert. Fix: subscribe to `chainChanged`/`accountsChanged` and reset UI or re-run `ensureBase()` before each action.

6. **Stake mode conversion allows Infinity/NaN from empty input (lines 1081-1088).** `Number($('stakeMain').value) || 0` coerces invalid inputs to `0`, potentially sending a zero-value createBet that fails. Fix: explicitly check `Number.isFinite(num)` and reject empty/invalid input before computing stake and calling `parseEther`.

7. **Share link/QR uses raw betId without validation (lines 1012-1038, 1405-1409).** Non-numeric `betId` values entered by the user are embedded into the URL and QR even though `requireBetId()` later rejects them. Fix: validate `betId` on input and either clear or normalize it so the shared link cannot point to an impossible bet ID.

8. **New bet button only resets local UI (lines 1348-1360, 1427-1433).** It clears the `betId` and QR but does not reset action statuses or deadlines, so a user might think they are sharing a fresh bet while still showing previous errors/deadlines. Fix: reuse the `btnReset` logic (deadlines + stake) and clear `actionStatus`/`createStatus` when starting a new bet.

9. **Hard-coded contract address and explorer links (lines 1002, 973-979).** The UI is locked to Base Sepolia; if deployed elsewhere the "Switch network" button still points to 84532. Fix: surface the active chain ID from `provider.getNetwork()` and adjust `BASE` values dynamically or enforce chain matching when the contract address changes.

10. **Fee display not aligned with contract (lines 972, 950-957).** The footer assumes a fixed `FEE_BPS = 250` and uses it to compute winner payout, but the contract may emit or store fee differently. If the contract fee changes, the UI payout preview is wrong. Fix: read fee from the contract (e.g., `getBet` or a `feeBps` getter) instead of a hard-coded constant and update the summary accordingly.

11. **Missing idea text persistence (lines 820-847, 1315-1346).** The "bet idea" entry is never sent on-chain or included in the share URL. Opponents receive no context when following the shared link, creating UX mismatch. Fix: include the idea in the share link (query param) and render it when loading `?bet=` or store it on-chain if supported.

12. **ABI/state assumptions may be stale (lines 1115-1122, 1249-1270).** The ABI uses positional outputs and assumes `state` at `b[7]` equals `1` for voting, with votes at `b[8]`/`b[9]` defaulting to `-1`. If the deployed contract differs, these checks misfire and block voting or miss already-cast votes. Fix: import the canonical ABI JSON from the verified source instead of hand-copying, and decode using named properties for clarity.

## Wiring gaps
- All UI buttons are wired (`btnCreate`, `btnJoin`, `btnRefund`, vote buttons, `btnPayout`), but none of the status fields are reset on tab switch or new bet creation, so stale errors persist. Add small resets when switching tabs or calling `startNewBet()`.
- The stake increment/decrement controls (`ethPlus`/`ethMinus`) only adjust in 0.0001 ETH even when stakeMode is USD, which can confuse users expecting dollar steps. Consider aligning steps with the active mode.

## Suggested front-end/contract alignment checks
- Ensure `joinBet`, `refund`, and `resolve` are gated client-side using the same timing/state logic as the contract to avoid user-paid revert fees.
- After every transaction, re-fetch `getBet` to refresh UI state (pot, opponent, deadlines) and disable buttons that no longer apply.
