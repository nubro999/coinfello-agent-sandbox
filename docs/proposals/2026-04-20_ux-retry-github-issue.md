# Delegation retry after simulation failure is silent, destructive, and unobservable

**CLI:** `@coinfello/agent-cli@0.3.6`
**Repro tx (eventual success):** [`0x1044f628…07b20`](https://arbiscan.io/tx/0x1044f628be9b38557f83f05e55f4cc12985478b420c401c46b16a35925e07b20)

## What happened

Sent 0.5 USDC on Arbitrum end-to-end. Identical `erc20TransferAmount(USDC, 500000)` scope, signed twice:

```text
approve_delegation_request                # callId=call_OcF3Pazz…
  → "The transfer failed during simulation: the delegation was only
     valid for the exact execution pattern, but the send flow needs
     the proper token transfer path."
  → exit 0, pending_delegation.json deleted

send_prompt "Yes, retry correctly."       # callId=call_mDmc7H4l…  (identical scope)
approve_delegation_request                # → "Done: 0x1044…"
```

## Problems

1. **Pending delegation wiped on soft failure.** `dist/index.js:3528-3529` calls `clearPendingDelegation()` unconditionally after HTTP 200, even when the server reports a simulation failure in the body. Local audit trail is lost.
2. **Success indistinguishable from soft failure.** Both fall through the `response.responseText` branch in `handleConversationResponse` (`dist/index.js:3282-3289`) and exit 0. The successful retry printed `Done: 0x1044…` via the same path, not the `txn_id → "Transaction submitted successfully"` branch. Scripts cannot tell outcomes apart.
3. **Redundant re-approval for identical scope.** Same `delegationArgs` JSON, two `callId`s, two Secure Enclave signatures.
4. **Error is an LLM paraphrase.** No enforcer name, no revert reason, no attempted target/selector. Can't distinguish caveat mismatch vs chain drift vs planner bug.

## Audit trail concern

`erc20TransferAmount` binds token/selector/amount but **not the recipient**. Linking `prompt → callId_1 (failed) → callId_2 (succeeded) → txHash` is the user's only check that the recipient they reviewed is the one funded. After step 1 clears the pending file, that chain is broken.

## Proposed fixes (priority order)

### P0 — CLI-only, no protocol change

Preserve the pending record until execution is confirmed:

```js
const finalResponse = await signAndSubmitDelegation(config, pending);
if (executionSucceeded(finalResponse)) {
  await archivePendingDelegation(pending, finalResponse.txHash);  // → history/<callId>.json
} else {
  await quarantinePendingDelegation(pending, finalResponse);      // → failed/<callId>.json
  process.exitCode = 2;
}
```

### P0 — server + CLI, additive field

Stable machine-readable execution result on the conversation response:

```jsonc
{
  "execution": {
    "status": "submitted" | "simulation_failed" | "reverted",
    "txHash": "0x…",
    "chainId": 42161,
    "callId": "call_OcF3Pazz…",
    "revertReason": "ERC20TransferAmount: target mismatch",
    "attemptedTarget": "0x1111…",
    "attemptedSelector": "0xa9059cbb",
    "retryCallId": "call_mDmc7H4l…"
  },
  "responseText": "…"
}
```

Old clients keep working (read `responseText`); new clients exit non-zero on failure and print `txHash` deterministically.

### P1 — retry flow

Either retry with the existing signature when only the server's execution plan changed, **or** ship the replacement `ask_for_delegation` tool call inside the simulation-failure response (with `parentCallId` linking it to the failed attempt). CLI asks once: *"identical scope re-issued (call_mDmc7H4l…, parent call_OcF3Pazz…); approve? [y/N]"*.

### P2 — surface enforcer-level error

```text
Simulation failed at caveat[0] ERC20TransferAmount
  expected target:  0xaf88…5831
  attempted target: 0x1111…
  selector: 0xa9059cbb
```

## What this is not

- Not changing the trust model — server-side planning stays.
- Not removing approval for real scope changes — only removing **redundant** re-approvals when scope is byte-identical to a just-failed attempt.
- Not auto-sign / auto-submit.

## Quick-win vote

§ P0-CLI + § P0-server. Both are additive; both give users an audit trail and scripts a reliable exit code.

Happy to open a PR for the P0-CLI piece.

---

*Full write-up with repro, source references, and extended rationale: [link](./2026-04-20_ux-retry-after-simulation-failure.md).*
