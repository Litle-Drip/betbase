# simple3.html full audit (The Juice betting dApp)

## simple3.html
1. **Resolve window not validated against join window (lines 1170-1193).** `doCreate` clamps both inputs independently but never enforces `resolve > join`, so the transaction can revert with missing revert data if the contract requires that ordering. *Fix:* after computing `jm` and `rm`, add `if (rm <= jm) throw new Error('Resolve window must exceed join window');` or auto-bump `rm` before calling `createBet`.

2. **Stake input can become NaN/zero and pass through to `parseEther` (lines 1079-1088, 1176-1187).** Empty/non-numeric input is coerced to `0`, causing zero-value or invalid `createBet` calls that revert. *Fix:* guard with `const num = Number($('stakeMain').value); if (!Number.isFinite(num) || num <= 0) throw new Error('Enter a valid stake');` and mirror that check on the hidden ETH value before `parseEther`.

3. **Hard-coded fee basis points may mismatch contract (lines 972, 1188-1193, 1050-1061).** The UI assumes a fixed `FEE_BPS = 250` for both the `createBet` call and win/pot previews; if the deployed contract uses a different fee, users see incorrect payouts or reverts. *Fix:* read fee from the contract (e.g., `getBet` or a `feeBps()` getter) and derive previews from that value instead of the constant.

4. **Join flow ignores deadline/state (lines 1213-1235).** `doJoin` only checks opponent presence; if the join window passed or the bet is in another state, the call can revert without friendly messaging. *Fix:* compare `b[5]` (join deadline) against `Math.floor(Date.now()/1000)` and confirm `b[7]` is the expected state before sending `joinBet`, otherwise surface a local error.

5. **Refund sends unconditionally (lines 1283-1296).** The UI never verifies bet existence, state, or deadline before calling `refund`, so users pay gas only to hit contract reverts (missing revert data risk). *Fix:* fetch `getBet`, ensure creator/opponent are set, time is past the refund threshold, and the bet is unresolved before enabling the button.

6. **Payout lacks vote/state gating (lines 1299-1312).** `doPayout` always calls `resolve`; if votes are incomplete or deadlines not reached, the contract will revert. *Fix:* read `getBet` and require the resolvable state (e.g., matching votes or expired window) before calling, with tailored error text.

7. **Vote flow omits resolution/vote windows and role-specific checks (lines 1241-1277).** It checks only `state === 1` and prior votes; if the vote window has closed or the state enum differs, calls revert silently. *Fix:* include timestamp/state validation aligned with contract rules and show explicit errors; also guard against calling before opponent joins.

8. **Share/QR accepts invalid bet IDs (lines 1012-1038, 1405-1416).** Any string in `betId` is embedded into the share URL/QR even though later actions require numeric IDs. *Fix:* validate the input on change, clearing or rejecting non-numeric IDs before updating the URL/QR.

9. **New bet/reset leaves stale statuses and deadlines (lines 1348-1360, 1435-1458).** `startNewBet` clears only `betId`/QR; `btnReset` resets minutes/stake but not status messages. Users can share a "new" link while old errors remain visible. *Fix:* when starting a new bet or resetting, also clear `createStatus`/`actionStatus`, reapply default deadlines, and hide `newBetNote`.

10. **Network/chain drift not handled after initial checks (lines 1418-1424, 1113-1160).** Actions call `ensureBase`, but there is no listener for `chainChanged`/`accountsChanged`; if the user switches networks after connecting, the UI may still show "Connected" and calls can revert. *Fix:* add event listeners to reset the signer/UI on chain or account changes and re-run `ensureBase` before constructing the contract.

11. **Clipboard and fetch failures are silent (lines 971-999, 1411-1416).** Fetching the price or copying the share link swallows errors, giving no user feedback (potential "nothing happens" confusion). *Fix:* show error toasts/status when fetch or clipboard writes fail and keep stale price messaging clear.

12. **Idea text is unused in transactions or share links (lines 805-821, 1315-1346).** The freeform bet description never reaches the contract or the shared URL, so opponents get no context and may join unintended bets. *Fix:* append the idea to the share URL (query param) and render it on load, or pass it to the contract if supported.

13. **Contract address and explorer are hard-coded (lines 1002, 941-965).** If deployed to another chain/address, the UI still points to Base Sepolia 84532, leading to wrong-chain transactions. *Fix:* derive chain ID from `provider.getNetwork()` and make the address/explorer configurable per environment.

14. **Stake increment controls ignore active mode (lines 1078-1109, 1382-1390).** `+/-` always adjusts ETH by `0.0001` even when the user is in USD mode, which can desync the displayed dollar amount and hidden ETH, risking unintended stakes. *Fix:* vary the step based on the active mode (e.g., $1 steps in USD mode, 0.0001 ETH in ETH mode) and reapply conversions.

15. **ABI/enum assumptions may be stale (lines 1113-1122, 1241-1270).** The ABI is hand-copied and state indices (`b[7]`, `b[8]`, `b[9]`) are hard-coded; if the verified contract differs, votes and gating misfire. *Fix:* import the verified ABI JSON and reference named outputs to stay aligned with the deployed contract.

## Files not present
- No Solidity, helper, or config files exist in the repo to audit; contract safety (reentrancy, fee routing, deadlines) cannot be confirmed from the front-end alone.
