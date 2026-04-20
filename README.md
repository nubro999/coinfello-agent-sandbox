# coinfello-agent-sandbox

CoinFello npm CLI test & personal agent sandbox.

## Setup

```bash
npm install
```

Installs `@coinfello/agent-cli` (currently `^0.3.6`).

## Usage

```bash
# list commands
npx coinfello --help

# create a smart account (saves address to local config)
npx coinfello create_account

# view saved account
npx coinfello get_account

# SIWE sign-in to CoinFello server
npx coinfello sign_in

# chat / request a delegation
npx coinfello send_prompt "swap 10 USDC to ETH"
npx coinfello approve_delegation_request

# run the Secure Enclave signer daemon (macOS)
npm run signer
```

## Notes

- Delegation config is stored locally (see `.coinfello/` — gitignored).
- Never commit `.env` or any private key material.
