# Mainnet Readiness

Tracks the blockers identified in the **2026-08-05 external security assessment** that must be
resolved before Veil wallets hold real user funds on mainnet.

## Blockers

All three original blockers are now resolved or in progress.

| ID | Description | Status |
|----|-------------|--------|
| M1 | Admin auth on `factory.init` — unauthenticated call allowed front-run | ✅ Fixed |
| M2 | Salt uses on-chain SHA-256 host function instead of bundled `sha2` crate | ✅ Fixed |
| M3 | On-curve pubkey validation compiled out of production (`#[cfg(test)]` gate) | ✅ Fixed — see below |

## M3 — On-curve pubkey validation

**Finding (2026-08-05 assessment):** `contracts/factory/src/validation.rs` gated the
`p256::ecdsa::VerifyingKey::from_sec1_bytes` on-curve check behind
`#[cfg(any(test, feature = "testutils"))]`. The check therefore ran in CI tests but was
compiled out of every production WASM build. An invalid (off-curve) public key with a correct
`0x04` prefix would pass validation on-chain.

**Fix (this PR, `fix/m3-on-curve-pubkey-validation`):**

1. Removed the `#[cfg(any(test, feature = "testutils"))]` gate from
   `contracts/factory/src/validation.rs`. The on-curve call to `VerifyingKey::from_sec1_bytes`
   now executes unconditionally in both test and production builds.

2. Added `p256 = { version = "0.13", features = ["ecdsa"] }` to the production
   `[dependencies]` in `contracts/factory/Cargo.toml` (without the `std` feature so it
   compiles under `#![no_std]` for `wasm32-unknown-unknown`). The dev-dependencies retain
   `std` for test ergonomics.

**WASM size delta:** The factory WASM is a lightweight deployment shim — it deploys wallets
but does not perform authentication itself. Adding the on-curve P-256 validation to the
release build increases the factory WASM size. The exact delta is reported in the CI log for
this PR (see `scripts/reproducible-build.sh`). The size budget is governed by
[`docs/wasm-size-optimization.md`](wasm-size-optimization.md).

**WASM hash:** The factory WASM hash in `contracts/expected-hashes.json` must be updated
from the reproducible CI build log after this PR is merged, per the convention in
[`docs/reproducible-build.md`](reproducible-build.md). Do **not** update it from a local
build — only the reproducible Docker build produces the canonical hash.

**Evidence:** A deployed factory on testnet must reject an off-curve key (correct `0x04`
prefix, x/y coordinates all-ones: `0x04 || 0x01*64`) with `InvalidPublicKey`. The transaction
hash demonstrating this rejection is recorded in the PR description.

## Remaining Pre-Mainnet Steps

Once the three security blockers are resolved, the following operational steps remain before
real funds:

1. Reproducible build — generate the canonical WASM artifacts and update
   `contracts/expected-hashes.json` from the CI log.
2. Testnet smoke-test — deploy factory and a wallet, perform a round-trip authentication,
   and confirm the invalid-key rejection described above.
3. Legal / regulatory sign-off (see `docs/NGN_RAILS.md`).
4. Monitoring and alerting in place (see `docs/MONITORING.md`).
