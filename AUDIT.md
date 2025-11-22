# The Juice Betting DApp — Security & UX Audit

## Scope
Entire repository (`index.html` front-end) with focus on contract interactions defined by the ABI inside `index.html`.

## Key Findings

### Critical
- **Unverified contract logic**: The repository ships only a contract ABI and hard-coded address (`0xD251F4aF...`). Without the Solidity source, users and integrators cannot verify fee deductions, state machine rules, or reentrancy protections. This undermines trust and prevents validating that the UI matches contract behavior. 【F:index.html†L236-L365】
- **Share/new-bet flow can surface nonexistent bet IDs**: `startNewBet` seeds `betId` with a random 8-digit number and immediately surfaces share URL/QR, but no on-chain bet is created. Any follow-on action (join/refund/payout) will revert with opaque errors (“missing revert data”) if the bet ID is invalid. There is no guard to prevent sharing or acting on IDs that do not exist on-chain. 【F:index.html†L547-L555】【F:index.html†L601-L623】

### High
- **No state or role gating before actions**: The UI lets any connected account call `joinBet`, `confirmWinner`, `refund`, or `resolve` without checking creator/opponent roles, deadlines, or current bet status. All validation is deferred to contract reverts, producing poor UX and possible “missing revert data” messages. Preflight read validation (creator/opponent addresses, timestamps, state enums) is absent. 【F:index.html†L405-L511】
- **Join uses on-chain stake but skips existence checks**: `doJoin` fetches `getBet` and uses `stakeWei` but never checks whether the returned bet is initialized (e.g., creator address != 0) or whether join is open. Users can pay gas only to revert if the bet doesn’t exist or is already joined. 【F:index.html†L453-L466】
- **Hard-coded ETH/USD preview price**: A fixed `ETH_USD = 3500` drives USD previews. If the real market price diverges, the UI will display inaccurate USD values and could mislead users about actual stakes. No freshness indicator or API fetch is implemented. 【F:index.html†L238-L297】

### Medium
- **Wallet/state resets are shallow**: Resetting the form (`btnReset`) only resets minutes and stake; it does not clear generated bet links or QR codes, so users may keep sharing stale/nonexistent bet IDs. 【F:index.html†L586-L604】【F:index.html†L614-L623】
- **Base network switch lacks error surfacing**: `ensureBase` swallows most switch/add errors, and `connect` clears the status pill silently. Users get no actionable reason when switching fails. 【F:index.html†L366-L392】
- **Clipboard failure ignored**: The copy-link button suppresses clipboard errors entirely, leaving users without feedback when writes fail (e.g., insecure contexts, permissions). This can block sharing without indication. 【F:index.html†L586-L590】

### Low / UX
- **Typo in address pill font**: `ont-family` typo prevented monospace styling in the contract address pill; corrected in this patch. 【F:index.html†L23-L70】
- **Fee immutability assumption**: UI assumes a fixed 2.5% fee (BPS 250). If the contract allows fee changes or dynamic fees, the UI will be inconsistent. 【F:index.html†L150-L230】【F:index.html†L238-L241】
- **No bet detail display**: The UI never renders `getBet` results (creator/opponent/status/deadlines), forcing users to cross-check explorers and increasing chances of acting in the wrong state. 【F:index.html†L405-L511】【F:index.html†L586-L623】

## Recommendations
1. **Publish contract source and verify deployment** so ABI/address can be trusted and audits can cover actual logic (reentrancy, fee flows, state machine, access control).
2. **Source next bet ID from chain** (e.g., `nextBetId` read) instead of random numbers, and gate sharing until `createBet` succeeds.
3. **Add preflight bet reads** before every action to validate existence, roles, join/resolve windows, and current state; present clear UI errors before sending transactions.
4. **Fetch live ETH/USD pricing** with a timestamped indicator or clarify that USD values are illustrative only.
5. **Reset link/QR when clearing the form**, and surface errors for network switching and clipboard writes.
6. **Render bet details** (addresses, timestamps, state, votes) in the UI to align expectations and reduce accidental reverts.
