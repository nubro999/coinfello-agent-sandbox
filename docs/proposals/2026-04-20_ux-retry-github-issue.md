# Identical-scope retries accumulate live delegations

**CLI:** `@coinfello/agent-cli@0.3.6`
**Repro tx:** [`0x1044f628…07b20`](https://arbiscan.io/tx/0x1044f628be9b38557f83f05e55f4cc12985478b420c401c46b16a35925e07b20)

## Issue

After a simulation failure, the server issues a new `ask_for_delegation` with an identical `delegationArgs`, and the user signs again. The previously signed delegation is **not invalidated** — nothing in the default scope expires it, and the client-side `clearPendingDelegation()` only removes a local JSON file.

A MetaMask Delegation Framework signature stops being redeemable only via one of:

- a `Timestamp` caveat whose expiry has passed,
- a `LimitedCalls` caveat whose counter is exhausted,
- a `NonceEnforcer` caveat whose nonce was invalidated, or
- an on-chain `disableDelegation(hash)` from the delegator.

None of these are present in `erc20TransferAmount(USDC, 500000)`. The failed-attempt signature is still live.

## Consequence

`maxAmount` is a per-delegation counter, not a global one. Each identical-scope retry adds an independently spendable authorization:

| Signature | On-chain status | Remaining spendable under its own `maxAmount` |
|---|---|---|
| `call_OcF3Pazz…` (failed simulation) | valid, unused | 500000 USDC |
| `call_mDmc7H4l…` (redeemed, tx `0x1044…`) | valid, exhausted | 0 USDC |

User intent was "send 0.5 USDC once." Actual authorization surface after the flow: **1.0 USDC**, split across two live delegations, one of which has never been redeemed and has no expiry.

Across repeated use, this stacks: every simulation-failure retry we ran left another dormant delegation behind.

## Repro

```text
send_prompt "send 0.5 USDC on Arbitrum to 0x467701…35cb"
approve_delegation_request          # call_OcF3Pazz…   simulation failed, signature retained server-side & on-chain-valid

send_prompt "Yes, retry correctly."
approve_delegation_request          # call_mDmc7H4l…   redeemed → tx 0x1044…
```

No `disableDelegation` was issued for `call_OcF3Pazz…`. Without an expiry caveat, it remains redeemable indefinitely.

## What we'd like

Identical-scope retries should not leave an unused earlier signature live. Whether that's handled by attaching an expiry/limited-calls caveat by default, by reusing the prior signature instead of issuing a new one, or by disabling the failed attempt on chain — is for the CoinFello team to choose. The invariant we care about is: **total live authorization surface should match what the user believes they authorized.**
