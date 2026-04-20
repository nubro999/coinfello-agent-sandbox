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

Same scope, same signer, same chat — second Secure Enclave prompt, second user decision, for an authorization that was already granted.

## Observed across repeated use

This is not a one-off. Across multiple end-to-end runs from this sandbox (different prompts, different amounts, different recipients), the same failure-then-retry pattern reappeared regularly, and in every case the retry delegation's scope was byte-identical to the first. A noticeable share of the total Secure Enclave prompts we went through were these redundant second approvals. The friction compounds — each identical-scope re-approval is another biometric, another review step that adds nothing the user didn't already authorize.

## Why the second signature is unnecessary

The delegation the user signed the first time is a standalone EIP-712 object. Nothing about it ties to a particular execution plan:

- `delegator`, `delegate`, `authority`, `caveats`, `salt`, `signature` — all independent of *how* the delegate chooses to redeem it.
- `ERC20TransferAmount` is a **cumulative** caveat, not a single-use one. The same signed delegation can legitimately back multiple `redeemDelegations` calls, as long as the running total stays under `maxAmount`.
- No `LimitedCalls` or `NonceEnforcer` caveat is applied, so there is no on-chain reason the first signature cannot be redeemed again with a corrected execution.

When the only thing that changed between the failed and successful attempts is the server's execution plan, the signed delegation already covers the retry. Asking the user to re-sign is client-round-trip overhead, not a security requirement.

## What we'd like

Identical-scope retries after a simulation failure should not surface to the user as a fresh approval. How that's achieved — server-side replan, signature reuse, or something else — is for the CoinFello team to decide; the goal is just to remove the redundant second signature.

## What still requires a fresh signature

Unchanged:

- Different chain, token, or `maxAmount` — scope changed, re-approve.
- A caveat actively rejected the attempted execution — the scope the user granted is genuinely insufficient; re-approve after scope widening.
- User-initiated new request after the chat turn is closed.
