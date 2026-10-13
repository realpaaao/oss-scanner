# Threat model: vultisig-sdk

## What this project does and where untrusted input enters
The Vultisig SDK is the TypeScript core that the Vultisig desktop app, the browser extension and third-party integrators use to create threshold vaults, derive addresses for 30+ chains, build and sign transactions, and talk to the relay and the vultiserver co-signer. Packages: `packages/sdk` (public API), `packages/core/mpc` (keygen/keysign/reshare orchestration), `packages/core/chain` (per-chain transaction construction, fee and gas logic, swap quotes), `packages/lib/dkls`, `packages/lib/schnorr` and `packages/lib/mldsa` (WASM threshold-crypto bindings), `packages/rujira`, `clients/cli` and `clients/mcp`.

Untrusted input:
- MPC protocol messages from other parties and from the relay, all of which may be malicious.
- Vault backup files (`.vult`, protobuf, optionally password-encrypted) that users import.
- Responses from chain RPCs, indexers, price APIs and swap aggregators (THORChain, Maya, 1inch, LiFi, SwapKit and others). These are third-party and can lie.
- Transaction requests from integrators, dApps and the MCP/CLI clients: destinations, amounts, memos, calldata.

## Components that matter most / least
Most: the code that turns a user intent into signing bytes (`packages/core/chain`: transaction builders, UTXO selection, fee/gas calculation, memo encoding, swap-quote handling), keysign message hashing and pre-signing hashes, address derivation, vault import/export and its encryption, MPC session handling in `packages/core/mpc`, and the MCP server's tool surface.
Less: `examples/`, `docs/`, `scripts/`, generated protobuf code. The prebuilt WASM blobs are out of scope; how the SDK calls them is in scope.

## How to exercise it
`yarn workspace @vultisig/sdk test:unit` and `yarn test:core` run offline. `node clients/cli/dist/index.js --help` runs the CLI. Integration and e2e suites need network and funded vaults, so they won't work offline.

## How we rate severity

Vultisig is a self-custodial threshold-signature (MPC) wallet. Every signing path can move user funds and there is no way to reverse an on-chain transaction, so we rate by impact on keys and funds:

- **Critical**: recovery of a private key or of enough key shares to sign; signing something other than what the user approved (wrong destination, amount, chain, contract call, or fee); an unauthenticated party causing a signature or a reshare; remote code execution in any component.
- **High**: disclosure of a single key share, vault backup contents, or a vault password; bypass of a share-encryption step; a malicious peer, relay, or server biasing nonces or extracting secret material from protocol messages; cross-session or cross-vault message confusion that completes a ceremony; theft of the email/verification code flow that gates a server-side share.
- **Medium**: transaction-display bugs that hide or misstate a destination, amount, or fee without changing what is signed; denial of service that blocks a user from signing for longer than a session; information leaks that link vaults, addresses, or devices.
- **Low**: crashes from malformed local input, timing leaks without a demonstrated attack, hardening gaps.

A finding with a working reproducer always ranks above the same class without one. We prefer reports that state the concrete precondition (who the attacker is: dApp, relay operator, network attacker, co-signer, or local malware) and the funds or key material at risk.

## Report format

Please include the exact file and line, a minimal reproducer (a test case in the repo’s own test framework is ideal), and a proposed patch. Patches should be minimal and must not weaken existing validation.

SDK-specific notes: an RPC, indexer or swap-API response that the SDK trusts and that changes the signed destination, amount or fee beyond what the user saw is Critical. The same lie that is only displayed is Medium. A divergence between the hash the SDK signs and the transaction it broadcasts is Critical.

## Anything to leave alone
Third-party APIs and chain nodes being unavailable is not a finding. Do not report dependency CVEs without a reachable path from SDK code.
