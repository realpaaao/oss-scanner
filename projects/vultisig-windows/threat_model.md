# Threat model: vultisig-windows

## What this project does and where untrusted input enters
This monorepo contains the Vultisig desktop app (Wails: Go backend in `app.go`, `mediator/`, `relay/`, `storage/`, `tss/`, `utils/`, plus a React frontend in `clients/desktop`), the Vultisig browser extension (Manifest V3, `clients/extension`), and the shared TypeScript in `core/` and `lib/`. Both clients hold a user's key share, run MPC keysign with other devices through a relay or a local mediator, and sign transactions for 30+ chains.

Untrusted input:
- **Web pages / dApps** talking to the extension's injected providers in `clients/extension/src/inpage/providers` (EIP-1193/EVM, Solana, Cosmos/Keplr, UTXO, THORChain, Maya, Sui, Tron, TON Connect, Cardano CIP-30, Polkadot, XRPL and others): connection requests, sign-message, sign-typed-data, send-transaction and add-chain/watch-asset calls. Any website is an attacker.
- Messages between the extension's content script, background service worker and popup.
- MPC protocol messages from other devices, the relay and the local mediator (`mediator/`, advertised over mDNS on the LAN) and `relay/`.
- Imported vault backup files, QR codes and deep links that start keygen/keysign/send flows.
- Chain RPC, indexer, price and swap-aggregator responses.

## Components that matter most / least
Most: the extension's dApp request handling, origin and permission checks, and approval screens (does the user see every destination, amount, chain and spender that will be signed?); the background service worker; keysign payload construction in `core/`; vault storage and encryption at rest (`storage/`, extension storage); the local mediator and relay servers; deep-link/QR parsing.
Less: `clients/desktop` build tooling, `scripts/`, i18n, UI styling, `docs/`.

## How to exercise it
`yarn test` (core/lib) and `yarn workspace @clients/extension test` run offline. The built extension is in `clients/extension/dist`. `go test ./mediator ./relay ./storage ./tss ./utils` runs the Go side.

## How we rate severity

Vultisig is a self-custodial threshold-signature (MPC) wallet. Every signing path can move user funds and there is no way to reverse an on-chain transaction, so we rate by impact on keys and funds:

- **Critical**: recovery of a private key or of enough key shares to sign; signing something other than what the user approved (wrong destination, amount, chain, contract call, or fee); an unauthenticated party causing a signature or a reshare; remote code execution in any component.
- **High**: disclosure of a single key share, vault backup contents, or a vault password; bypass of a share-encryption step; a malicious peer, relay, or server biasing nonces or extracting secret material from protocol messages; cross-session or cross-vault message confusion that completes a ceremony; theft of the email/verification code flow that gates a server-side share.
- **Medium**: transaction-display bugs that hide or misstate a destination, amount, or fee without changing what is signed; denial of service that blocks a user from signing for longer than a session; information leaks that link vaults, addresses, or devices.
- **Low**: crashes from malformed local input, timing leaks without a demonstrated attack, hardening gaps.

A finding with a working reproducer always ranks above the same class without one. We prefer reports that state the concrete precondition (who the attacker is: dApp, relay operator, network attacker, co-signer, or local malware) and the funds or key material at risk.

## Report format

Please include the exact file and line, a minimal reproducer (a test case in the repo’s own test framework is ideal), and a proposed patch. Patches should be minimal and must not weaken existing validation.

Client-specific notes: a website that gets a signature without an approval prompt, or whose approval prompt misstates what will be signed, is Critical. A website that reads addresses or balances without a granted connection is Medium. Anything reachable from the local network through the mediator/relay that reaches key material or signing is Critical.

## Anything to leave alone
Attacks that need full control of the user's OS or browser profile (local malware with the same privileges) are out of scope unless they cross a boundary we claim, such as decrypting the vault without the password.
