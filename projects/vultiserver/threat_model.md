# Threat model: vultiserver

## What this project does and where untrusted input enters
vultiserver is the server co-signer for Vultisig "Fast Vaults": a 2-of-2 (or larger) threshold vault in which the user holds one share on a device and vultiserver holds the other, encrypted with a password the user chose. It runs keygen, keysign, reshare, key import and ML-DSA key creation (DKLS23 and Schnorr via the native go-wrappers libraries, GG20 via mobile-tss-lib). The encrypted server share is also emailed to the user as a backup.

Untrusted input enters through:
- The HTTP API in `api/server.go` (`/vault/create`, `/vault/sign`, `/vault/reshare`, `/vault/import`, `/vault/migrate`, `/vault/batch/*`, `/vault/mldsa`, `/vault/get`, `/vault/exist`, `/vault/resend`, `/vault/verify`, `/getDerivedPublicKey`). All request bodies are attacker-controlled.
- MPC protocol messages that arrive through the relay from the other party, who may be malicious.
- Vault files read from storage (`vaults/`, `storage/`), which an attacker may be able to influence through the API.

## Components that matter most / least
Most: `service/` (keysign_dkls.go, reshare_*.go, keygen_*.go, import_*.go, worker*.go, the mpc wrapper), `api/`, share encryption and decryption, vault storage, and the email verification-code flow.
Less: `demo/`, `test/e2e` fixtures, Makefile targets. The native DKLS/Schnorr libraries are vendored binaries from go-wrappers; bugs in how vultiserver calls them are in scope, bugs inside the binaries themselves are not.

## How to exercise it
`go build ./...` and `go test ./...` in /src. `redis-server --daemonize yes` then `vultiserver` (config: `config-example.yaml`) starts the API on :8080. The relay endpoint is unreachable offline, so full ceremonies need a local relay or a mocked one.

## How we rate severity

Vultisig is a self-custodial threshold-signature (MPC) wallet. Every signing path can move user funds and there is no way to reverse an on-chain transaction, so we rate by impact on keys and funds:

- **Critical**: recovery of a private key or of enough key shares to sign; signing something other than what the user approved (wrong destination, amount, chain, contract call, or fee); an unauthenticated party causing a signature or a reshare; remote code execution in any component.
- **High**: disclosure of a single key share, vault backup contents, or a vault password; bypass of a share-encryption step; a malicious peer, relay, or server biasing nonces or extracting secret material from protocol messages; cross-session or cross-vault message confusion that completes a ceremony; theft of the email/verification code flow that gates a server-side share.
- **Medium**: transaction-display bugs that hide or misstate a destination, amount, or fee without changing what is signed; denial of service that blocks a user from signing for longer than a session; information leaks that link vaults, addresses, or devices.
- **Low**: crashes from malformed local input, timing leaks without a demonstrated attack, hardening gaps.

A finding with a working reproducer always ranks above the same class without one. We prefer reports that state the concrete precondition (who the attacker is: dApp, relay operator, network attacker, co-signer, or local malware) and the funds or key material at risk.

## Report format

Please include the exact file and line, a minimal reproducer (a test case in the repo’s own test framework is ideal), and a proposed patch. Patches should be minimal and must not weaken existing validation.

## Anything to leave alone
Production infrastructure (Cloudflare, Mandrill, api.vultisig.com) is out of scope. Do not report the use of the deprecated GG20 library by itself; report concrete flaws in how it is driven.
