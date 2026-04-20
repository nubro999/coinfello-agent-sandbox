# Don't ask for a second signature when retrying an identical-scope delegation

**CLI:** `@coinfello/agent-cli@0.3.6`
**Repro tx:** [`0x1044f628…07b20`](https://arbiscan.io/tx/0x1044f628be9b38557f83f05e55f4cc12985478b420c401c46b16a35925e07b20)

## Issue

When a first approval fails server-side simulation, the user is forced to re-send a prompt and sign a **new** delegation with a **new `callId`** — even though its `delegationArgs` are byte-identical to the one just signed.

```text
approve_delegation_request                # callId=call_OcF3Pazz…   scope={erc20TransferAmount, USDC, 500000}
# simulation failed, natural-language "retry?" reply

send_prompt "Yes, retry correctly."
approve_delegation_request                # callId=call_mDmc7H4l…   scope={erc20TransferAmount, USDC, 500000}
# → tx 0x1044…
```

Same scope, same signer, same chat — second Secure Enclave prompt, second user decision, for an authorization that was already granted in the first one.

## Why the second signature is unnecessary

The delegation the user signed the first time is a standalone EIP-712 object. Nothing about it ties to a particular execution plan:

- `delegator`, `delegate`, `authority`, `caveats`, `salt`, `signature` — all independent of *how* the delegate chooses to redeem it.
- `ERC20TransferAmount` is a **cumulative** caveat, not a single-use one. The same signed delegation can legitimately back multiple `redeemDelegations` calls, as long as the running total stays under `maxAmount`.
- No `LimitedCalls` or `NonceEnforcer` caveat is applied, so there is no on-chain reason the first signature cannot be redeemed again with a corrected execution.

So when the only thing that changed between the failed and successful attempts is **the server's execution plan**, the server already has everything it needs: the signed delegation. Asking the user to re-sign is pure client-round-trip overhead.

## Proposed behavior

On simulation failure where the caveats themselves did not reject the attempted execution (i.e. the server's planner picked a path that wouldn't even enter the caveat check — wrong `target`, wrong selector, wrong call shape, etc.):

1. **Server re-plans internally** using the already-submitted signed delegation.
2. **Re-simulates** with the corrected execution.
3. **Submits** on success, returns the tx hash in the same HTTP response.

The user sees one prompt → one signature → one result. No "retry" prompt, no new `callId`, no second biometric.

If the server's replan requires a genuinely different scope (e.g. needs to touch a second token, hit a bridge, exceed `maxAmount`), that's a real scope change and a second signature is appropriate — but that is **not** what we hit in the repro above. The second `delegationArgs` was identical to the first.

## What still requires a fresh signature

Unchanged:

- Different chain, token, or `maxAmount` — scope changed, re-approve.
- A caveat actively rejected the attempted execution — the scope the user granted is genuinely insufficient; re-approve after scope widening.
- User-initiated new request after the chat turn is closed.

## Minimal contract

On the conversation API response after simulation failure, the server can either:

- (a) Complete the redemption internally and return the final execution result in the same response — no retry round-trip needed, or
- (b) If a client round-trip is still required (e.g. to re-use a fresh nonce salt), return a lightweight `reuse_prior_signature` directive keyed to the prior `callId`, and the CLI re-submits the already-cached signature without prompting the user again.

Option (a) is simpler and is the preferred fix.

## Out of scope

- Server-side execution planning itself. The planner can keep choosing paths; we only ask that identical-scope retries don't bubble up to the user as a fresh approval.
- Auto-signing. Nothing here bypasses the user's initial Secure Enclave prompt — only the redundant second one.
