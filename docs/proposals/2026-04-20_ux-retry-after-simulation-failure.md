# UX Proposal — Handling Delegation Retry After Simulation Failure

**Author:** user of `@coinfello/agent-cli@0.3.6`
**Date:** 2026-04-20
**Context:** First end-to-end test of CoinFello via the agent CLI, transferring 0.5 USDC on Arbitrum One.
**Tx (eventual success):** [`0x1044f628...07b20`](https://arbiscan.io/tx/0x1044f628be9b38557f83f05e55f4cc12985478b420c401c46b16a35925e07b20)

---

## 1. Summary

When a signed delegation reaches the CoinFello backend and the resulting `redeemDelegations` call fails during simulation, the current flow returns a natural-language suggestion to retry, but:

1. The CLI silently clears the local pending-delegation file **before** observing whether the server-side execution succeeded.
2. The CLI exits with code `0` regardless, so neither humans nor scripts can tell success from soft failure without string-matching the response text.
3. The user must manually send a follow-up prompt (e.g. "retry") to regenerate an identical-scope delegation with a new `callId`, then re-approve.
4. No structured error detail (enforcer name, revert reason, failing caveat, simulated calldata) is surfaced — only the LLM's paraphrase.

For a product whose core value proposition is *secure, auditable, agent-driven onchain execution*, this creates four concrete UX problems (§3) and one security-observability concern (§4).

---

## 2. Reproduction

Exact steps taken:

```bash
npx coinfello create_account          # smart account: 0xD9a4...9717 (Secure Enclave)
npx coinfello sign_in                  # SIWE OK, session saved
npx coinfello send_prompt "send 0.5 USDC on Arbitrum to 0x467701...35cb"
#   → delegation request: erc20TransferAmount(USDC, 500000), callId=call_OcF3Pazz...
npx coinfello approve_delegation_request
#   → "Creating subdelegation..." → "Signing..." → "Sending signed delegation..."
#   → "The transfer failed during simulation: the delegation was only valid
#      for the exact execution pattern, but the send flow needs the proper
#      token transfer path.
#      If you want, I can prepare and execute the USDC transfer correctly."
#   → exit 0, pending_delegation.json deleted

npx coinfello send_prompt "Yes, please retry the transfer correctly."
#   → new delegation request, identical scope, callId=call_mDmc7H4l...
npx coinfello approve_delegation_request
#   → "Done: 0x1044f628...07b20"
```

Both delegations carried the **same** `delegationArgs`:

```json
{
  "chainId": 42161,
  "scope": {
    "type": "erc20TransferAmount",
    "tokenAddress": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
    "maxAmount": "500000"
  }
}
```

So the server's second attempt succeeded with the identical signed scope — the only thing that changed was the server-side execution plan.

---

## 3. Issues Observed

### 3.1 Pending delegation is cleared before success is confirmed

In `dist/index.js` (0.3.6) the approve action does:

```js
const finalResponse = await signAndSubmitDelegation(config, pending);
await clearPendingDelegation();                         // ← always runs
await handleConversationResponse(finalResponse, …);
```

`clearPendingDelegation()` runs unconditionally once the HTTP POST succeeds, even if the server reported a simulation failure in the body. The user loses the only local artifact that describes what they just authorized.

### 3.2 Success is indistinguishable from soft failure

`handleConversationResponse` branches:

```js
if (!response.clientToolCalls?.length && !response.txn_id) {
  console.log(response.responseText ?? "");   // ← soft-failure lands here
  return;                                     // ← exit 0
}
if (response.txn_id && !response.clientToolCalls?.length) {
  console.log("Transaction submitted successfully.");
  console.log(`Transaction ID: ${response.txn_id}`);
}
```

The soft-failure and the "server chose not to respond with a tx" paths are collapsed. The second attempt's success also went through the `responseText` branch (the final output line was `Done: 0x1044…07b20`, not the "Transaction submitted successfully" path). There is no dependable programmatic signal that a tx landed.

### 3.3 Retry requires a whole new user-initiated prompt

After simulation failure the server's response is plain text. Even though the server already has the original prompt and the signed delegation, the client is expected to re-prompt ("Yes, retry"), after which the server emits a fresh `ask_for_delegation` tool call with a new `callId`. The user signs again — with the same Secure Enclave biometric cost — for the same effective scope.

### 3.4 No actionable error detail

The error is an LLM paraphrase: *"the delegation was only valid for the exact execution pattern, but the send flow needs the proper token transfer path."* Nothing about:

- Which enforcer rejected (e.g. `ERC20TransferAmount`, `AllowedTargets`, `AllowedMethods`).
- What execution the backend attempted (target, selector, calldata snippet).
- Whether it was a revert, a paymaster rejection, or a bundler drop.

A user investigating why their authorization failed cannot distinguish server-side planner bug, chain-state drift, insufficient balance, or a broken caveat match.

---

## 4. Security / Observability Concern

An `erc20TransferAmount` caveat does **not** bind the recipient — it only fixes token, selector, value, and cumulative amount (see MetaMask Delegation Framework `ERC20TransferAmount.sol`). The recipient is carried in the server-built execution calldata. That is acceptable under the current trust model (user trusts CoinFello planner), but when retries happen silently with only natural-language feedback, the user has no structured audit trail linking:

`originalPrompt` ↔ `callId_1 (failed)` ↔ `callId_2 (succeeded)` ↔ `txHash`.

If a future incident required proving that the recipient in the final tx matched what the user had reviewed, the only evidence on the user's machine is the **second** `pending_delegation.json`, which has already been deleted by the time the tx hash is printed.

---

## 5. Proposed Changes

Listed in priority order. Each is compatible with the current protocol (no breaking API change required for the bulk).

### 5.1 Preserve pending delegation until execution is confirmed (P0)

Do not call `clearPendingDelegation()` immediately after HTTP 200. Instead:

```js
const finalResponse = await signAndSubmitDelegation(config, pending);

if (executionSucceeded(finalResponse)) {
  await archivePendingDelegation(pending, finalResponse.txHash);   // move → history/
} else {
  await markPendingDelegationAsFailed(pending, finalResponse);     // keep for audit
}
await handleConversationResponse(finalResponse, …);
```

An `~/.clawdbot/skills/coinfello/history/<callId>.json` archive (successful) and `~/.clawdbot/skills/coinfello/failed/<callId>.json` (failed) give the user a local audit trail of everything they authorized.

### 5.2 Return a structured execution result (P0, requires server change)

Add a stable machine-readable shape to the conversation response:

```jsonc
{
  "execution": {
    "status": "submitted" | "simulation_failed" | "reverted" | "rejected",
    "txHash": "0x…",                      // when submitted
    "chainId": 42161,
    "callId": "call_OcF3Pazz…",
    "revertReason": "ERC20TransferAmount: amount exceeds max",
    "attemptedTarget": "0xaf88…5831",
    "attemptedSelector": "0xa9059cbb",
    "retrySuggested": true,
    "retryCallId": "call_mDmc7H4l…"       // if backend already re-issued
  },
  "responseText": "…human-readable message…"
}
```

CLI can then:

- Exit non-zero on `simulation_failed` / `reverted`.
- Print `Transaction ID: <txHash>` deterministically.
- Log structured error for scripts / CI.

### 5.3 One-shot retry with same signed delegation or identical-scope re-issue (P1)

Two acceptable shapes:

1. **Server-driven retry with same signature** — if simulation fails purely because of the server's chosen execution path (not caveat violation), the server re-plans and re-simulates internally using the already-submitted signature. The user sees one approval, one success.
2. **Atomic re-issue in the same response** — if a new delegation is required, the simulation-failure response carries the new `ask_for_delegation` tool call alongside `execution.status = "simulation_failed"`. CLI immediately writes the replacement `pending_delegation.json` and asks the user once: *"First attempt failed; identical scope re-issued (call_mDmc7H4l…). Approve? [y/N]"*. This preserves user-in-the-loop without a round trip.

### 5.4 Preserve retry lineage (P1)

Add `parentCallId` to every re-issued delegation:

```json
{
  "callId": "call_mDmc7H4l…",
  "parentCallId": "call_OcF3Pazz…",
  "retryReason": "simulation_failed: attempted target != tokenAddress"
}
```

CLI displays the parent so the user knows this is a replacement for a prior attempt and can find the archived failure.

### 5.5 Surface the enforcer-level reason (P2)

When a simulation fails, include the decoded revert reason from the first failing caveat:

```
Simulation failed at caveat[0] ERC20TransferAmount
  expected target: 0xaf88…5831
  attempted target: 0x111111125421cA6dc452d289314280a0f8842A65
  selector: 0xa9059cbb (transfer)
```

This turns a debugging dead end into an actionable signal for the user (and for CoinFello's own ops team).

### 5.6 Optional: offer caveat-hardening presets in the CLI (P3)

Current `scope` types don't expose finer caveats (`ArgsEqualityCheck`, `Timestamp`, `LimitedCalls`) that would bind recipient and expiry on chain. A CLI flag like:

```bash
coinfello send_prompt "send 0.5 USDC …" --bind-recipient --expire=1h --once
```

…would compose these enforcers for users who want tighter guarantees than the current scope vocabulary offers. Strictly opt-in; current behavior unchanged.

---

## 6. Out-of-Scope / Non-Goals

- **Changing the trust model.** We are *not* proposing to remove server-side planning — the server's ability to translate natural language into executions is a feature. We are proposing better observability *around* it.
- **Eliminating the second approval.** For scope changes, explicit re-approval is correct. The proposal only removes redundant re-approvals for **identical-scope** retries.
- **Auto-approve.** No proposal here automatically signs or submits without the user's action. Every change preserves user-in-the-loop.

---

## 7. Quick-Win Summary

If CoinFello ships only two things from this, our vote is:

1. **§5.1** — keep the pending-delegation record on disk until execution is confirmed, archive on success, quarantine on failure.
2. **§5.2** — structured `execution` field with stable `status` and `txHash`, so the CLI can exit non-zero on failure and print the tx hash deterministically.

Both are server-side + CLI-side changes with no breaking impact on existing integrations (added fields are additive; `responseText` still works for older clients).

---

## 8. Artifacts

- Repro repo: `coinfello-agent-sandbox` (this branch).
- Delegation review doc: [`docs/delegations/2026-04-20_usdc-arb-0.5.md`](../delegations/2026-04-20_usdc-arb-0.5.md).
- Successful tx doc: [`docs/transactions/2026-04-20_usdc-arb-0.5_tx.md`](../transactions/2026-04-20_usdc-arb-0.5_tx.md).
- Raw on-chain tx: [Arbiscan](https://arbiscan.io/tx/0x1044f628be9b38557f83f05e55f4cc12985478b420c401c46b16a35925e07b20).
- CLI source references: `@coinfello/agent-cli@0.3.6` `dist/index.js` lines `3277–3312` (response handling), `3313–3366` (sign & submit), `3511–3535` (approve command).

Happy to open a GitHub issue / PR against the CLI for §5.1 if helpful; §5.2–§5.5 require server-side work that only the CoinFello team can ship.
