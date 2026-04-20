# Identical-scope delegation retry forces a second signature without re-verifiable recipient

**CLI:** `@coinfello/agent-cli@0.3.6`
**Repro tx:** [`0x1044f628…07b20`](https://arbiscan.io/tx/0x1044f628be9b38557f83f05e55f4cc12985478b420c401c46b16a35925e07b20)

## Issue

When a first approval fails simulation, the server re-issues a delegation with an **identical scope** but a **new `callId`**, requiring a second signature. `erc20TransferAmount` pins `tokenAddress`, `selector`, and `maxAmount`, but **it does not bind the recipient** — the recipient lives only inside the server-built `redeemDelegations` execution calldata, which the user never sees.

So a re-approval that looks byte-identical to the user (same scope JSON) can, in principle, carry a different recipient than the one just reviewed. Nothing in the signed data prevents this.

## Repro

```text
send_prompt "send 0.5 USDC on Arbitrum to 0x467701…35cb"
# callId=call_OcF3Pazz…   scope={erc20TransferAmount, USDC, 500000}

approve_delegation_request
# simulation failed, server suggests retry

send_prompt "Yes, retry correctly."
# callId=call_mDmc7H4l…   scope={erc20TransferAmount, USDC, 500000}   ← identical
# recipient: not in scope, only in backend's execution plan

approve_delegation_request
# → tx 0x1044…07b20
```

Both signatures authorized "up to 0.5 USDC of USDC on Arbitrum, via `transfer()`, to *whoever the server puts in calldata*." The user's only check that the recipient matches is the `originalPrompt` string in `pending_delegation.json` — which is informational, not binding.

## Why it matters

- **Trust model leak.** A caveat is the on-chain guarantee; a prompt string is not. Making the user re-sign without letting them re-verify the binding target collapses the two into one trust boundary.
- **Repetition blunts review.** When scope JSON is byte-identical across attempts, users will fast-approve the second one. That's the moment recipient mutation would land unnoticed.
- **No local lineage.** `callId_1` is discarded; there is no structured link from `call_mDmc7H4l…` back to `call_OcF3Pazz…`, so a diff of "what changed between the two" is impossible.

## Proposed fix

### 1. Bind the recipient on chain for identical-scope retries

When the server re-issues after a simulation failure, compose the delegation with an additional `ArgsEqualityCheck` caveat that pins the ERC-20 `transfer(address,uint256)` recipient argument to the exact address the user's original prompt resolved to. Retry is then provably to the same address; mutation reverts on chain.

### 2. Atomic re-issue with explicit lineage

Ship the replacement `ask_for_delegation` inside the simulation-failure response, carrying `parentCallId` and a diff summary:

```jsonc
{
  "execution": { "status": "simulation_failed", "callId": "call_OcF3Pazz…" },
  "clientToolCalls": [{
    "name": "ask_for_delegation",
    "callId": "call_mDmc7H4l…",
    "parentCallId": "call_OcF3Pazz…",
    "scopeDiff": "identical",
    "recipient": "0x467701…35cb",
    "recipientBindingCaveat": "ArgsEqualityCheck@0x…"
  }]
}
```

CLI displays: *"Retry of `call_OcF3Pazz…`, scope identical, recipient `0x467701…35cb` now bound by caveat. Approve? [y/N]"* The second signature is no longer redundant — it is explicitly authorizing the hardened version.

### 3. If (1) is not feasible, at least show the resolved recipient

The CLI's delegation-review output today shows scope fields only. It should also display the resolved recipient (and chainId-specific token symbol) that the server intends to use, separately labelled as *unsigned intent* vs *signed caveat* so users know which is enforced.

## Out of scope

- Server-side execution planning stays as is.
- Real scope changes (different token, larger amount, different chain) still require explicit re-approval — nothing here weakens that.
- Not auto-approve; user still signs.

## Quick-win

Fix (1). It turns identical-scope retries from "trust the server to put the right recipient in calldata twice in a row" into "chain refuses to deliver to any other recipient." One caveat addition, no protocol break for clients that don't look for it.
