# Design: Oracle-Router & Multisig-Threshold Performance and Fuzz Hardening

**Status:** Draft
**Issues:** #422, #431, #427, #428
**Scope:** `contracts/oracle-router`, `contracts/multisig-threshold`

## Prerequisite and scope note

The four issues target `contracts/oracle-router` and `contracts/multisig-threshold`.
Neither contract is present in this checkout — the workspace currently contains only
`archival-probe`, `factory`, `governance`, `stream`, and `stream-nft` under `contracts/`,
and a repo-wide search finds no references to either contract name.

This design therefore specifies the **target behavior and performance characteristics**
for those contracts rather than a retrofit of existing code. Implementation requires
first importing or scaffolding `oracle-router` and `multisig-threshold` into the
workspace. The optimization and fuzz strategies below are defined against that target
so they apply directly once the contracts land. Open question Q1 tracks this.

## Goals

1. Minimize Soroban host CPU instructions in `oracle-router` and `multisig-threshold` by
   eliminating redundant deserialization in loops (#422, #431).
2. Streamline cross-contract invocation (CCI) overhead in `oracle-router` during
   high-frequency batch claims (#427).
3. Add fuzz coverage for edge-case timestamps near the `u64` maximum boundary in
   `oracle-router` (#428).
4. Defensive error handling and checked arithmetic throughout, with no panics on
   untrusted input.

## Non-goals

- Changing external contract APIs or on-wire storage formats beyond what the
  optimizations require.
- Treating before/after instruction counts as a deliverable in themselves; they are
  verification evidence, not a goal.
- Gas tuning unrelated to deserialization, CCIs, or timestamp arithmetic.

## Background: cost model

- **Deserialization.** Reading a collection from durable storage via
  `Env::storage().get()` returns a `Val` that is converted to the target type through
  `TryFromVal`. Re-reading the same key inside a loop, or re-deriving an element per
  iteration, repeats host work proportional to `N`. Binding the result to a local
  once and iterating the local avoids the repeat.
- **Cross-contract invocation.** Each `invoke_contract` / CCI carries fixed host
  overhead (args serialization, dispatch, result conversion). In a per-claim loop this
  cost is multiplied by the batch size.
- **Timestamps.** Soroban ledger timestamps are `u64`. Expiry, staleness-window, and
  next-update arithmetic can overflow near `u64::MAX`; `checked_add`/`checked_sub`
  return a typed `Error` instead of panicking or silently wrapping.

## Design

### 1. Loop deserialization (#422 oracle-router, #431 multisig-threshold)

**Problem.** Iterating a stored collection by calling `storage().get()` per element —
or holding a `Val` and re-converting each iteration — re-enters the host deserializer
`N` times.

**Approach.**

- **Deserialize once, iterate the local.** Pull the collection from storage into a
  strongly-typed local (`Vec<T>` / `Map<K, V>`) once per call, then iterate the bound
  local. Do not call `storage().get()` inside the loop.
- **Iterate, don't index-fetch.** Prefer `iter()` over the bound collection over
  repeated `Vec::get(i)` when the element is consumed once; materialize the whole
  collection up front rather than touching storage per index.
- **Nested collections.** Ensure the outer deserialize fully materializes nested
  structures (e.g. `Vec<Signer>` whose `Signer` holds a nested `Vec`); do not lazily
  re-read inner data per outer element.
- **multisig-threshold shape.** Store signers as a single `Map<Address, u32>`
  (address → weight) or a single `Vec<Signer>`, deserialized once per verification
  pass, instead of per-signer storage reads. Threshold checks then iterate the bound
  local.
- **oracle-router shape.** Cache the active price feed map deserialized once per
  routing decision; route lookups iterate the local rather than re-reading storage.

**Verification.**

- Unit tests at large `N` (256 signers; 256 price entries) assert correct aggregates
  with zero per-element storage reads (a test storage hook counts reads).
- Before/after host CPU instruction counts captured in test output to confirm the
  reduction.

### 2. Cross-contract invocation overhead in batch claims (#427 oracle-router)

**Problem.** In high-frequency batch claims, issuing one CCI per claim to fetch oracle
price(s) multiplies fixed CCI overhead by batch size.

**Approach.**

- **Single-read fan-out.** Fetch the relevant price feed(s) once at the start of the
  batch, bind to a local, and reuse for every claim that consumes the same feed. Only
  re-invoke when a claim requires a different feed/asset.
- **Group claims by feed.** Bucket claims by the price feed they consume; fetch each
  distinct feed at most once, then process each bucket. CCI count becomes
  `O(distinct feeds)`, not `O(claims)`.
- **Compact CCI args.** Pass minimal `Val`-typed arguments across the CCI boundary
  (a feed id / index) rather than large structs.
- **Cache-first reads.** Have the router pull prices into its own storage on update;
  claims read the cache, so a batch claim triggers zero CCIs in the common case and
  falls back to a single CCI per missing feed.

**Verification.**

- Unit/fuzz tests with batch sizes 1, 16, 256 assert CCI count scales with distinct
  feeds, not claims, via a test double that counts invocations.

### 3. Edge-case timestamp fuzzing (#428 oracle-router)

**Problem.** Timestamp arithmetic (expiry, staleness window, next-update) can overflow
near `u64` boundaries, causing panics or silent wraparound.

**Approach.**

- Add a `proptest` fuzz target generating timestamps across boundary regions: `0`,
  `1`, near ledger close time, near `u32::MAX`, near `u64::MAX`, and `u64::MAX` itself;
  deltas include `0`, `1`, large values, and values that force `t + delta` and
  `t - delta` to overflow.
- All timestamp arithmetic uses checked operations; overflow maps to a single typed
  `Error` variant rather than panicking or wrapping.
- **Property under test:** for any `(timestamp, delta)` the router never panics and
  returns a deterministic `Ok`/`Err`; every overflow maps to the same error variant.

**Verification.**

- `cargo test` fuzz target runs many shrinkable cases; CI gate prevents regressions.

### 4. Cross-cutting: defensive error handling and checked arithmetic

- Arithmetic on amounts, weights, timestamps, and counts uses checked operations; on
  overflow return a contract `Error` variant. No `unwrap`/`expect`/unvalidated
  indexing on untrusted input.
- Validate at trust boundaries: batch length, feed id, signer address, threshold
  range.
- Storage reads validated against expected types; missing keys return typed errors.

## Risks and trade-offs

- **Footprint.** Caching deserialized collections increases transient memory within a
  call, bounded by batch size — acceptable for the batch sizes in scope.
- **Failure atomicity.** Batching CCIs changes failure semantics: a single feed fetch
  failure now fails the whole batch. Mitigation: per-feed error isolation where the
  spec allows partial success; otherwise documented as intentional all-or-nothing
  (see Q2).

## Open questions

- **Q1.** Are `oracle-router` and `multisig-threshold` to be scaffolded in this repo, or
  imported from `releaseken/Perpetua-Contracts`? Neither exists in this checkout; this is
  blocking for implementation.
- **Q2.** Should batch claims support partial success, or remain all-or-nothing?

## Out of scope for this draft

Implementation code, the actual contract scaffolding, and measured before/after
numbers — all pending resolution of Q1 and the contracts landing in the workspace.
