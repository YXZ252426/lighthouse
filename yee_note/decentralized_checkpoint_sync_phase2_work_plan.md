# Phase 2 Work Plan: Verifiable Beacon-State Checkpoint Retrieval

> Status: working implementation plan  
> Updated: 2026-08-10  
> Lighthouse baseline: `unstable@f392d254cf63bfb34148d4849fe570a7f27dec7d`  
> Primary near-term target: provider-side proof construction and an experimental REST endpoint

## 1. Phase 2 Goal

Starting from a recent beacon block root already authenticated by a light-client trust chain, obtain a state root and eventually a complete `BeaconState` from an untrusted provider, verify every cryptographic binding locally, and produce a typed checkpoint anchor that can enter Lighthouse's existing weak-subjectivity initialization path.

The end-to-end security chain is:

```text
externally trusted recent block root
  -> verified LightClientBootstrap
  -> verified LightClientStore and updates
  -> verified finalized BeaconBlockHeader
  -> verified state root
  -> downloaded BeaconState
  -> fork-aware SSZ decode and hash_tree_root verification
  -> matching signed beacon block
  -> existing weak_subjectivity_state initialization
```

The provider is trusted only for availability. It must not be trusted to select the chain, state root, slot, fork, or state contents.

### Near-term definition of done

The first provider vertical slice is complete when:

1. A request is keyed by an exact `beacon_block_root`, never by `head` or slot alone.
2. Lighthouse loads the corresponding canonical finalized stored block.
3. It derives the block header and a proof binding `state_root` to the requested root.
4. A transport-independent verifier accepts the valid response.
5. Mutating the block root, state root, proof node, proof length, or proof index causes rejection.
6. An experimental REST endpoint returns the same typed response.
7. No new database column, persistent cache, or libp2p protocol is added unless measurements show it is necessary.

### Phase 2 non-goals for the first slice

- Do not implement state chunk transport yet.
- Do not add a new libp2p protocol yet.
- Do not solve pre-chain P2P startup yet.
- Do not implement historical-accumulator snapshot proofs yet.
- Do not treat an optimistic header as a checkpoint anchor.
- Do not allow an unverified `Hash256` or provider-selected `head` to enter checkpoint initialization.

## 2. Design Gate: Is the New Proof Endpoint Actually Necessary?

This question must be answered before the wire format is frozen.

### 2.1 What a standard light client already knows

`LightClientHeader` already contains a complete `BeaconBlockHeader`, and `BeaconBlockHeader` already contains `state_root`. Once a consumer has verified a finalized light-client header, the state root is already authenticated. No second Merkle proof is needed to read that field.

Lighthouse also already exposes:

```http
GET /eth/v1/beacon/headers/{block_id}
```

The endpoint accepts an exact block root and returns the full header. An untrusted response can be checked by computing:

```text
hash_tree_root(response.header.message) == trusted_beacon_block_root
```

The provider's `canonical` and `finalized` booleans are useful metadata but are not part of the consumer's trust proof.

### 2.2 The proposed compact proof

`BeaconBlockHeader` has five fields and is padded to eight SSZ leaves. Its `state_root` is leaf index `3`, at depth `3` and generalized index `11`.

```rust,ignore
const BEACON_BLOCK_HEADER_DEPTH: usize = 3;
const STATE_ROOT_FIELD_INDEX: usize = 3;

struct BeaconStateRootProof {
    beacon_block_root: Hash256,
    state_root: Hash256,
    state_root_branch: FixedVector<Hash256, U3>,
}
```

Verification is:

```rust,ignore
verify_merkle_proof(
    response.state_root,
    &response.state_root_branch,
    BEACON_BLOCK_HEADER_DEPTH,
    STATE_ROOT_FIELD_INDEX,
    trusted_beacon_block_root,
)
```

This proof is valid if the consumer retains only the trusted block root. However, it has two drawbacks:

- `state_root` plus a three-node branch is 128 bytes, while the full SSZ `BeaconBlockHeader` is 112 bytes.
- It does not authenticate `slot`, which the consumer needs for fork-aware state decoding. The slot must come from the already verified light-client header, a second proof, or the full header.

### 2.3 Required decision

Complete a short design spike and choose one of these outcomes:

| Option | Use when | Recommendation |
| --- | --- | --- |
| Read `state_root` directly from the verified LC header | The consumer retains the verified header | Preferred end-to-end design |
| Fetch the full header by trusted root and re-hash it | The consumer retains only the root | Preferred fallback; already supported by Lighthouse REST |
| Return `BeaconStateRootProof` | The project explicitly requires a field proof as a research artifact | Acceptable experimental endpoint |
| Prove another retained historical state root | The target is not the LC header's own state | Separate later protocol requiring state/historical-root proofs |

The provider proof prototype is still useful as a small vertical slice, but its design note must state that it does not add security when the complete verified LC header is already available.

## 3. Current Lighthouse Baseline

### 3.1 Existing provider data flow

The current light-client server flow is:

```text
import block and post-state
  -> LightClientServerCache::cache_state_data
  -> cache current/next committee and Merkle branches by block root in memory
  -> emit LightClientProducerEvent only for blocks less than 32 slots late
  -> compute_light_client_updates worker
  -> LightClientServerCache::recompute_and_cache_updates
  -> derive optimistic/finality/period update from the attested parent
  -> compare the period candidate with is_better_light_client_update
  -> persist selected period update, committee, and committee branch
  -> serve through REST or Req/Resp
```

Read these entry points together:

- [`light_client_server_cache.rs`](beacon_node/beacon_chain/src/light_client_server_cache.rs)
- [`beacon_chain.rs`](beacon_node/beacon_chain/src/beacon_chain.rs), especially `cache_state_data`, the recent-event gate, and `recompute_and_cache_light_client_updates`
- [`compute_light_client_updates.rs`](../beacon_node/client/src/compute_light_client_updates.rs)

### 3.2 Existing persistent mappings

| Column | Key | Value | Current use |
| --- | --- | --- | --- |
| `LightClientUpdate` | period, big-endian `u64` | fork-aware `LightClientUpdate` | best known update for a period and ordered range reads |
| `SyncCommittee` | period, SSZ/LE `u64` | `SyncCommittee` | bootstrap assembly |
| `SyncCommitteeBranch` | exact block root | `MerkleProof` | bind committee to the requested bootstrap header |

Relevant code:

- [`hot_cold_store.rs`](../beacon_node/store/src/hot_cold_store.rs)
- [`migration_schema_v30.rs`](../beacon_node/beacon_chain/src/schema_change/migration_schema_v30.rs)
- [`store_tests.rs`](../beacon_node/beacon_chain/tests/store_tests.rs)

The v30 migration changed period-update keys to big-endian so range scans remain numerically ordered across period 256.

### 3.3 Cache and reorg limitations

- Finality and optimistic updates are global in-memory values and contain a TODO about fork-aware canonical selection.
- Period updates are persisted only when the new candidate wins `is_better_light_client_update`.
- Committee branches are initially written by block root; finalization migration prunes non-checkpoint branches.
- The producer event excludes blocks more than 32 slots late, so ordinary historical block backfill does not automatically create complete historical LC data.
- Bootstrap assembly explicitly reports that historical committee/branch backfill is absent.

#### Relevant code excerpts

**1. Latest finality and optimistic updates are process-local and not fork-aware**

[`LightClientServerCache` fields and fork-awareness TODO](../beacon_node/beacon_chain/src/light_client_server_cache.rs#L22-L42):

```rust,ignore
pub struct LightClientServerCache<T: BeaconChainTypes> {
    /// Tracks a single global latest finality update out of all imported blocks.
    ///
    /// TODO: Active discussion with @etan-status if this cache should be fork aware to return
    /// latest canonical (update with highest signature slot, where its attested header is part of
    /// the head chain) instead of global latest (update with highest signature slot, out of all
    /// branches).
    latest_finality_update: RwLock<Option<LightClientFinalityUpdate<T::EthSpec>>>,
    /// Tracks a single global latest optimistic update out of all imported blocks.
    latest_optimistic_update: RwLock<Option<LightClientOptimisticUpdate<T::EthSpec>>>,
    // ...
}
```

These fields are initialized to `None` in `LightClientServerCache::new` and updated directly through the `RwLock`; unlike period updates, they have no corresponding database write.

**2. A period update is persisted only when it is better than the stored candidate**

[`recompute_and_cache_updates`](../beacon_node/beacon_chain/src/light_client_server_cache.rs#L190-L206):

```rust,ignore
let prev_light_client_update =
    self.get_light_client_update(&store, sync_period, chain_spec)?;

let should_persist_light_client_update =
    if let Some(prev_light_client_update) = prev_light_client_update {
        prev_light_client_update
            .is_better_light_client_update(&new_light_client_update, chain_spec)?
    } else {
        true
    };

if should_persist_light_client_update {
    store.store_light_client_update(sync_period, &new_light_client_update)?;
    *self.latest_light_client_update.write() = Some(new_light_client_update);
}
```

**3. Committee branches are keyed by block root, then non-checkpoint branches are pruned**

The producer writes the branch using the attested block's tree-hash root in [`recompute_and_cache_updates`](../beacon_node/beacon_chain/src/light_client_server_cache.rs#L118-L121):

```rust,ignore
store.store_sync_committee_branch(
    attested_block.message().tree_hash_root(),
    &cached_parts.current_sync_committee_branch,
)?;
```

The store serializes that exact root as the database key in [`store_sync_committee_branch`](../beacon_node/store/src/hot_cold_store.rs#L835-L846):

```rust,ignore
pub fn store_sync_committee_branch(
    &self,
    block_root: Hash256,
    sync_committee_branch: &MerkleProof,
) -> Result<(), Error> {
    let column = DBColumn::SyncCommitteeBranch;
    self.hot_db.put_bytes(
        column,
        &block_root.as_ssz_bytes(),
        &sync_committee_branch.as_ssz_bytes(),
    )?;
    Ok(())
}
```

After finalization, [`prune_non_checkpoint_sync_committee_branches`](../beacon_node/beacon_chain/src/migrate.rs#L821-L861) retains checkpoint roots and schedules every other finalized branch for deletion:

```rust,ignore
if *slot % E::slots_per_epoch() == 0 {
    epoch_boundary_blocks.insert(block_root);
} else {
    non_checkpoint_block_roots.insert(block_root);
}

if epoch_boundary_blocks.contains(&block_root) {
    non_checkpoint_block_roots.remove(&block_root);
}

// Prune sync committee branch data for all non checkpoint block roots.
non_checkpoint_block_roots
    .into_iter()
    .for_each(|block_root| {
        hot_db_ops.push(StoreOp::DeleteSyncCommitteeBranch(*block_root));
    });
```

**4. Historical imports do not enter the normal light-client producer path**

[`BeaconChain` block-import gate](../beacon_node/beacon_chain/src/beacon_chain.rs#L4949-L4960):

```rust,ignore
// Do not trigger light_client server update producer for old blocks, to extra work
// during sync.
if self.config.enable_light_client_server
    && block_delay_total < self.slot_clock.slot_duration() * 32
    && let Some(mut light_client_server_tx) = self.light_client_server_tx.clone()
    && let Ok(sync_aggregate) = block.body().sync_aggregate()
    && let Err(e) = light_client_server_tx.try_send((
        block.parent_root(),
        block.slot(),
        sync_aggregate.clone(),
    ))
{
    // warning omitted
}
```

The strict `< 32 * slot_duration` condition means older blocks imported during sync do not produce cache work through this channel.

**5. Bootstrap retrieval has no historical committee/branch backfill**

[`get_light_client_bootstrap`](../beacon_node/beacon_chain/src/light_client_server_cache.rs#L389-L423):

```rust,ignore
/// Note: It should be the case that a `sync_committee_branch` and `sync_committee` exist in the db
/// for a finalized checkpoint block root. However, we currently have no backfill mechanism for these values.
/// Therefore, `sync_committee_branch` and `sync_committee` are only persisted while a node is synced.
pub fn get_light_client_bootstrap(
    &self,
    store: &BeaconStore<T>,
    block_root: &Hash256,
    finalized_period: u64,
    chain_spec: &ChainSpec,
) -> Result<Option<(LightClientBootstrap<T::EthSpec>, ForkName)>, BeaconChainError> {
    // ...
    let Some(current_sync_committee_branch) = store.get_sync_committee_branch(block_root)?
    else {
        return Err(BeaconChainError::LightClientBootstrapError(format!(
            "Sync committee branch for block root {:?} not found. This typically occurs when the block is not a finalized checkpoint. Light client bootstrap is only supported for finalized checkpoint block roots.",
            block_root
        )));
    };
    // ...
}
```

### 3.4 Existing transports

REST provider:

- [`http_api/src/light_client.rs`](../beacon_node/http_api/src/light_client.rs)
- [`http_api/src/lib.rs`](../beacon_node/http_api/src/lib.rs)
- [`common/eth2/src/lib.rs`](../common/eth2/src/lib.rs)

Req/Resp provider:

- [`rpc/methods.rs`](../beacon_node/lighthouse_network/src/rpc/methods.rs)
- [`rpc/protocol.rs`](../beacon_node/lighthouse_network/src/rpc/protocol.rs)
- [`rpc/codec.rs`](../beacon_node/lighthouse_network/src/rpc/codec.rs)
- [`service/api_types.rs`](../beacon_node/lighthouse_network/src/service/api_types.rs)
- [`network/src/router.rs`](../beacon_node/network/src/router.rs)
- [`network_beacon_processor/rpc_methods.rs`](../beacon_node/network/src/network_beacon_processor/rpc_methods.rs)

`NetworkEvent::RequestReceived` and `NetworkEvent::ResponseReceived` are generic envelopes. A new P2P request does not need a dedicated `NetworkEvent` variant; it needs new `RequestType`, RPC response, and application `Response` variants carried by the existing event.

### 3.5 Missing consumer path

Lighthouse currently serves light-client RPCs but does not consume them as a startup light client:

- `Router::handle_rpc_response` marks all light-client responses as `unreachable!()`.
- `AppRequestId` has no light-client bootstrap-controller request family.
- There is no complete `LightClientStore` sync state machine in the current tree.
- The official `light_client/sync` EF test handler is absent.
- Normal networking starts only after `BeaconChain` construction, while decentralized checkpoint acquisition must happen before the chain exists.

### 3.6 Current upstream status

- Lighthouse PR [#9666](https://github.com/sigp/lighthouse/pull/9666), the data-collection EF test handler, is still an open draft as of 2026-08-10. Its remaining work includes correct `NewHead` handling and harness initialization.
- Lighthouse issue [#9587](https://github.com/sigp/lighthouse/issues/9587), Gloas light-client type support, is still open. The Phase 2 prototype must reject unsupported Gloas/Heze LC data explicitly rather than decoding it as an older fork.

## 4. Code Reading Plan

Do not read the repository linearly. Read it in the following order and write a one-page trace after each stage.

### R1 - Header root and Merkle proof primitives

Read:

- [`beacon_block_header.rs`](consensus/types/src/block/beacon_block_header.rs)
- [`beacon_block.rs`](consensus/types/src/block/beacon_block.rs), especially `block_header`
- [`signed_beacon_block.rs`](consensus/types/src/block/signed_beacon_block.rs)
- [`merkle_proof/src/lib.rs`](consensus/merkle_proof/src/lib.rs), especially `MerkleTree::generate_proof` and `verify_merkle_proof`

Be able to answer:

1. Why does the block message root equal the derived header root?
2. Why is the `state_root` field index `3`, depth `3`, gindex `11`?
3. In what order are Lighthouse proof nodes stored?
4. Why is a full header smaller and more informative than the proposed field proof?

Exercise: write a unit test that derives a header from a block, proves `state_root`, verifies it against `header.canonical_root()`, and rejects every one-field mutation.

### R2 - Fork-aware light-client objects

Read in this order:

1. [`light_client_header.rs`](consensus/types/src/light_client/light_client_header.rs)
2. [`light_client_bootstrap.rs`](consensus/types/src/light_client/light_client_bootstrap.rs)
3. [`light_client_update.rs`](consensus/types/src/light_client/light_client_update.rs)
4. [`light_client_finality_update.rs`](consensus/types/src/light_client/light_client_finality_update.rs)
5. [`light_client_optimistic_update.rs`](consensus/types/src/light_client/light_client_optimistic_update.rs)
6. [`consts.rs`](consensus/types/src/light_client/consts.rs)

Trace:

- which header is attested and which is finalized;
- which state root each committee/finality branch is checked against;
- how `signature_slot` selects the authenticating committee period;
- how `is_better_light_client_update` ranks candidates;
- where fork-specific execution-header branches enter the LC header;
- where Gloas currently returns `NotImplemented`.

Then read the authoritative consensus-spec sections:

- [Light-client sync protocol](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/light-client/sync-protocol.md)
- [Light-client full-node behavior](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/light-client/full-node.md)
- [Light-client P2P interface](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/light-client/p2p-interface.md)

### R3 - Provider cache and canonical-head flow

Read:

- [`light_client_server_cache.rs`](beacon_node/beacon_chain/src/light_client_server_cache.rs)
- block import and event emission in [`beacon_chain.rs`](beacon_node/beacon_chain/src/beacon_chain.rs)
- [`compute_light_client_updates.rs`](beacon_node/client/src/compute_light_client_updates.rs)
- branch pruning in [`migrate.rs`](beacon_node/beacon_chain/src/migrate.rs)

Trace two different objects:

```text
block post-state -> cached proof material
child sync aggregate -> update about its parent/attested header
```

Do not merge these concepts. The snapshot proof from a block header can be produced directly from the stored block and does not require the LC server cache.

### R4 - Database behavior

Read:

- LC methods in [`hot_cold_store.rs`](beacon_node/store/src/hot_cold_store.rs)
- `DBColumn` definitions in [`store/src/lib.rs`](beacon_node/store/src/lib.rs)
- v30 migration and related tests
- hot/cold block and state retrieval paths

Trace key encoding, fork-aware decoding, pruning, restart behavior, and the difference between a stored block, stored state, and retained proof material.

### R5 - REST and Req/Resp provider routing

Read REST first, then P2P:

1. Existing header-by-root and LC routes in [`http_api/src/lib.rs`](beacon_node/http_api/src/lib.rs)
2. [`http_api/src/light_client.rs`](beacon_node/http_api/src/light_client.rs)
3. Client calls in [`common/eth2/src/lib.rs`](common/eth2/src/lib.rs)
4. RPC method/container definitions
5. Protocol ID and size limits
6. Codec request/response mapping
7. `NetworkEvent` conversion
8. `Router::handle_rpc_request`
9. `NetworkBeaconProcessor` provider handler

Write down the exact route for one `LightClientBootstrap` request before adding a snapshot request.

### R6 - Missing consumer and startup handoff

Read:

- LC response arm in [`network/src/router.rs`](beacon_node/network/src/router.rs)
- [`client/src/config.rs`](beacon_node/client/src/config.rs)
- checkpoint acquisition in [`client/src/builder.rs`](beacon_node/client/src/builder.rs)
- [`BeaconChainBuilder::weak_subjectivity_state`](beacon_node/beacon_chain/src/builder.rs)
- existing HTTP client state/header/block calls in `common/eth2`

Be able to explain the startup cycle:

```text
current: BeaconChain -> NetworkService
required later: bootstrap transport -> verified state/block -> BeaconChain -> normal NetworkService
```

### R7 - Read later, not before the first endpoint

Only after the simple root proof is complete, read:

- `BeaconState::state_roots`, `historical_roots`, and `historical_summaries`;
- fork-dependent state generalized indices;
- archive/pruning and state reconstruction;
- state SSZ streaming and compression limits;
- pre-chain discovery and libp2p construction.

These are required for an alternate historical state target, chunk transport, and final P2P startup, but not for the first header-field proof.

## 5. Minimal Transport-Independent Interfaces

Freeze these interfaces before adding HTTP or Req/Resp code.

### 5.1 Historical light-client readiness and backfill

Historical backfill is an internal provider job. Existing standard REST/P2P bootstrap and updates-by-range methods remain the external serving interface.

```rust,ignore
struct LightClientBackfillRange {
    start_period: u64,
    count: u64,
}

struct LightClientCoverage {
    oldest_bootstrap_epoch: Option<Epoch>,
    oldest_contiguous_update_period: Option<u64>,
    newest_contiguous_update_period: Option<u64>,
    last_completed_period: Option<u64>,
    status: CoverageStatus,
}

enum CoverageStatus {
    NotStarted,
    Running,
    Complete,
    IncompletePrunedData,
    Failed,
}
```

Internal operation:

```rust,ignore
trait HistoricalLightClientBuilder<E: EthSpec> {
    fn backfill_range(
        &self,
        range: LightClientBackfillRange,
    ) -> Result<LightClientCoverage, BackfillError>;
}
```

Required semantics:

- enumerate canonical source blocks/states;
- derive fork-aware candidates;
- select with `is_better_light_client_update`, not participation alone;
- write idempotently;
- expose gaps and pruned-data failures;
- advertise only completed contiguous coverage;
- never promise that every arbitrary trusted root has bootstrap material.

### 5.2 Beacon state-root proof

Request:

```rust,ignore
struct BeaconStateRootProofRequest {
    beacon_block_root: Hash256,
}
```

Response:

```rust,ignore
struct BeaconStateRootProof {
    beacon_block_root: Hash256,
    state_root: Hash256,
    state_root_branch: FixedVector<Hash256, U3>,
}
```

Provider semantics:

1. Load the stored blinded block by exact root.
2. Derive `BeaconBlockHeader` from the block.
3. Require `header.canonical_root() == requested_root` as an internal consistency check.
4. For the first endpoint, serve only if `BeaconChain::is_finalized_block(root, slot)` is true.
5. Generate the branch from the header's eight padded field leaves.
6. Return `ResourceUnavailable` for missing, non-canonical, non-finalized, or pruned data.

Consumer semantics:

1. Ignore provider claims about canonicality.
2. Require `response.beacon_block_root == requested_trusted_root`.
3. Verify the depth-3/index-3 branch against the trusted root.
4. Obtain slot/fork only from already verified LC context or a separately verified full header.
5. Construct a private trusted wrapper only after verification.

```rust,ignore
struct VerifiedStateRoot {
    beacon_block_root: Hash256,
    state_root: Hash256,
}
```

Error classes:

```text
InvalidRequest | ResourceUnavailable | InternalProviderError
InvalidProof | MismatchedRequestedRoot | UnsupportedForkContext
```

### 5.3 Whole-state and chunk retrieval

First define whole-state verification, then chunking.

```rust,ignore
struct StateManifestRequest {
    state_root: Hash256,
}

struct StateManifest {
    state_root: Hash256,
    encoding: StateEncoding,
    total_size: u64,
    chunk_size: u32,
    chunk_count: u32,
    manifest_id: Hash256,
}

struct StateChunkRequest {
    manifest_id: Hash256,
    state_root: Hash256,
    chunk_index: u32,
}

struct StateChunk {
    manifest_id: Hash256,
    state_root: Hash256,
    chunk_index: u32,
    offset: u64,
    bytes: Vec<u8>,
    checksum: Hash256,
}
```

Rules:

- `manifest_id` is a domain-separated hash of one canonical manifest encoding.
- Manifest identity includes `state_root`, encoding/compression, total size, and chunk size.
- Chunk cache keys are `(manifest_id, chunk_index)`.
- Checksums detect transport corruption only.
- Security comes from fork-aware SSZ decode followed by `hash_tree_root(state) == verified_state_root`.
- The trusted slot/fork comes from verified LC/header context, not untrusted manifest metadata.
- Arbitrary byte chunks do not carry SSZ Merkle proofs in the MVP.

## 6. Exact Integration Map

### 6.1 First REST provider slice

| Concern | Integration point |
| --- | --- |
| Proof constants/helper | `consensus/types/src/block/beacon_block_header.rs` or a narrowly scoped new module |
| Shared response type | `consensus/types` if it will later be SSZ/P2P; otherwise prototype in `common/eth2` until reviewed |
| Provider retrieval | new `BeaconChain::get_beacon_state_root_proof(block_root)` method |
| Canonical/finalized gate | `BeaconChain::is_finalized_block` |
| Stored data | `BeaconChain::get_blinded_block` / existing block DB |
| REST route | `beacon_node/http_api/src/lib.rs` plus a small dedicated handler module |
| REST client | `common/eth2/src/types.rs` and `common/eth2/src/lib.rs` |
| Provider tests | `beacon_node/beacon_chain/tests` and `beacon_node/http_api/tests/tests.rs` |
| Consumer verifier tests | colocated with the proof type or a new transport-independent consumer module |

Recommended experimental route:

```http
GET /lighthouse/v1/beacon/state_root_proof/{beacon_block_root}
Accept: application/octet-stream
```

Use a Lighthouse-specific namespace until there is a reviewed Beacon API specification. SSZ should be the primary format; JSON can aid debugging.

### 6.2 Later P2P provider integration

If the REST prototype demonstrates value, a single-response P2P protocol would touch:

| Layer | Required change |
| --- | --- |
| `rpc/methods.rs` | request and successful response containers |
| `rpc/protocol.rs` | `Protocol`, `SupportedProtocol`, protocol ID, request/response limits, `RequestType` |
| `rpc/codec.rs` | SSZ encode/decode mapping |
| `rpc/rate_limiter.rs` and config | quota and request cost |
| `rpc/methods.rs::RpcSuccessResponse` | response variant |
| `service/api_types.rs::Response` | application response variant and conversion |
| `NetworkEvent` | no new event variant; existing generic request/response events carry the new enums |
| `lighthouse_network::Network::inject_rpc_event` | map new RPC variant into generic event |
| `network/src/router.rs` | provider request dispatch and, later, consumer response routing |
| `network_beacon_processor/mod.rs` | schedule provider work |
| `network_beacon_processor/rpc_methods.rs` | call the same `BeaconChain` provider method used by REST |
| consumer request IDs | add a bootstrap-controller request family; do not overload unrelated normal sync IDs |

Do not duplicate proof construction or verification in the transport layers.

### 6.3 Database impact

The first state-root proof requires no new DB column:

```text
block_root -> existing stored blinded block -> derived header -> deterministic proof
```

Only add a cache after measuring proof-generation cost. If a cache is added, its key is:

```text
(beacon_block_root, proof_schema_version)
```

Never key it by `head`, `finalized`, or slot alone.

### 6.4 Per-type integration matrix

This matrix is the implementation checklist for the three proposed interfaces. A dash means the type must not cross that boundary in the first version.

| Type | RPC definition | `NetworkEvent` | Router / provider handler | Database access | Consumer verification |
| --- | --- | --- | --- | --- | --- |
| `LightClientBackfillRange` | None; internal job input | None | Dedicated backfill service, not `Router` | Reads historical blocks/states; writes existing LC columns | None |
| `LightClientCoverage` | REST/admin status first; do not put in consensus Req/Resp yet | None | Read-only coverage handler | Derived from persisted progress plus gap checks | Source selection only; never establishes trust |
| Existing `LightClientBootstrap` | Existing LC bootstrap v1 | Existing generic request/response event | Existing LC provider handler | Block + committee + branch | Bootstrap root and committee-branch verification in LC consumer |
| Existing `LightClientUpdate` | Existing updates-by-range v1 | Existing generic request/response event | Existing range provider handler | Ordered period-update column | Standard LC store validation and processing |
| `BeaconStateRootProofRequest` | Future `beacon_state_root_proof/1/ssz_snappy` single request | Existing `RequestReceived` | New `Router` request arm -> beacon-processor work | Exact-root blinded-block read only | Not applicable on provider |
| `BeaconStateRootProof` | Future `RpcSuccessResponse` and application `Response` variants | Existing `ResponseReceived` | Provider calls shared `BeaconChain` method; consumer routes to bootstrap controller | No new column; deterministic from block | Depth-3/index-3 proof -> `VerifiedStateRoot` |
| `StateManifestRequest` | Future single-response manifest protocol | Existing generic request event | New provider request arm | State metadata/serialized-state source | Require requested verified root and deterministic manifest ID |
| `StateManifest` | Future successful response variant | Existing generic response event | Consumer bootstrap controller, not normal range sync | Optional manifest cache by `manifest_id` | Bounds, root, encoding, size, and canonical ID checks |
| `StateChunkRequest` | Future one-chunk request protocol | Existing generic request event | New chunk provider arm | Read exact byte range from immutable serialized-state source | Not applicable on provider |
| `StateChunk` | Future successful response variant | Existing generic response event | Consumer state-transfer scheduler | Optional cache `(manifest_id, chunk_index)` | Binding/index/offset/checksum, then final state SSZ root |
| `VerifiedStateRoot` | Never serialized | Never sent | Consumer-local only | Optional resumable local metadata | Constructible only by the proof/header verifier |
| `VerifiedCheckpointAnchor` | Never serialized | Never sent | Passed only to checkpoint builder adapter | Existing checkpoint DB initialization | Constructible only after state-root and block/state checks |

For all future P2P rows, `Network::inject_rpc_event` must translate the new RPC enum variant into the existing generic event. Adding separate `NetworkEvent::BeaconSnapshot` or `NetworkEvent::StateChunk` variants would duplicate the current Req/Resp architecture without a demonstrated need.

## 7. Cache Keys and Reorg-Safety Rules

### 7.1 Immutable keys

| Object | Key |
| --- | --- |
| Header state-root proof | `(beacon_block_root, proof_schema_version)` |
| Bootstrap committee branch | exact `beacon_block_root` |
| Sync committee | `sync_committee_period` |
| Period update | `sync_committee_period`, mutable only under standard better-update rules until frozen |
| State manifest | domain-separated hash of canonical manifest fields |
| State chunk | `(manifest_id, chunk_index)` |
| Verified consumer checkpoint | `(beacon_block_root, state_root)` plus authenticated slot/fork context |

### 7.2 Reorg rules

1. The request contains an exact block root; the provider never resolves `head` on the consumer's behalf.
2. The first provider endpoint serves only locally finalized canonical blocks.
3. A proof under a cryptographic block root is immutable. A reorg changes discovery/advertisement, not the proof bytes under that root.
4. Never overwrite data for root A with data derived from root B because they share a slot.
5. Unfinalized period-update candidates may be replaced only through `is_better_light_client_update` and canonical-head selection.
6. Finalized coverage can be advertised as stable; incomplete or pruned ranges must remain explicitly unavailable.
7. Consumer trust comes from its LC-verified root, not from the provider's local finality claim.
8. If orphan-root data is retained, do not advertise it as canonical. Serving it only by an exact consumer-trusted root is cryptographically safe, but the initial prototype should use the stricter finalized-canonical policy.

## 8. Implementation Milestones

Each milestone is intended to be reviewable independently. Estimates are focused engineering days, not calendar commitments.

### M0 - Security proposition and endpoint-necessity spike

**Estimate:** 1-2 days  
**Output:** short design note, no production code

Tasks:

1. Prove that a verified `LightClientHeader` already authenticates `beacon.state_root`.
2. Demonstrate the existing header-by-root REST fallback.
3. Compare full-header and branch response sizes.
4. State exactly whether the endpoint is a research artifact, an availability signal, or a required security bridge.
5. Decide how authenticated slot/fork context reaches state decoding.

Exit criteria:

- The mentor/reviewer approves the security proposition and response semantics.
- The protocol does not claim that the field proof adds security to an already verified full header.

### M1 - Complete the mandatory code-reading trace

**Estimate:** 2-4 days  
**Output:** call graphs and data-lifetime notes

Tasks:

1. Complete R1-R6 above.
2. Trace one bootstrap request from transport to DB and back.
3. Trace one live period update from imported child block to persisted period key.
4. Trace current checkpoint URL startup into `weak_subjectivity_state`.
5. Record every trust boundary and every mutable identifier.

Exit criteria:

- You can explain why LC update computation concerns the attested parent while the proposed header proof concerns the requested block itself.
- You can identify which data survives restart and which remains in memory.

### M2 - Freeze three minimal interface specifications

**Estimate:** 1-2 days  
**Output:** reviewed SSZ/pseudocode specification

Freeze:

1. Historical LC coverage/backfill semantics.
2. `BeaconStateRootProofRequest` and `BeaconStateRootProof`.
3. `StateManifest` and one-chunk request/response semantics.

Include:

- request and response limits;
- exact proof depth/index;
- error taxonomy;
- fork/version behavior;
- cryptographic cache keys;
- unavailable versus invalid behavior;
- test vectors with one valid and several corrupted responses.

Exit criteria:

- No transport code is required to understand or test the verifier.

### M3 - Pure header-proof primitive

**Estimate:** 1-3 days  
**Suggested PR 1**

Tasks:

1. Add a narrowly scoped state-root proof helper for `BeaconBlockHeader`.
2. Reuse `merkle_proof::MerkleTree` and `verify_merkle_proof`.
3. Add a transport-independent verifier returning `VerifiedStateRoot`.
4. Keep proof constants in one place.

Required tests:

- valid proof for multiple block fixtures;
- block/header canonical-root equality;
- wrong state root;
- each branch node corrupted in turn;
- wrong trusted block root;
- truncated/extended branch;
- wrong field index/depth;
- zero/default header edge case.

Exit criteria:

- The primitive has no dependency on `BeaconChain`, HTTP, libp2p, cache, or DB.

### M4 - Provider chain method

**Estimate:** 2-4 days  
**Suggested PR 2**

Tasks:

1. Add `BeaconChain::get_beacon_state_root_proof(block_root)`.
2. Load the existing blinded block by exact root.
3. Derive the full header without reconstructing execution payloads or loading `BeaconState`.
4. Check internal root consistency.
5. Gate the response with finalized canonical status.
6. Return typed unavailable/internal errors.

Required tests:

- canonical finalized stored block succeeds;
- unknown root is unavailable;
- non-finalized root is unavailable;
- same-slot competing root is unavailable;
- provider response verifies with the pure consumer verifier;
- corrupted response fails consumer verification;
- restart does not change the deterministic response.

Exit criteria:

- Provider retrieval uses only canonical stored data and existing DB abstractions.
- No new persistence is introduced.

### M5 - Experimental REST vertical slice

**Estimate:** 2-4 days  
**Suggested PR 3**

Tasks:

1. Add a Lighthouse-specific route keyed by block root.
2. Add SSZ response support and bounded JSON support if useful.
3. Add `common/eth2` response type and client method.
4. Run the client response through the same pure verifier.
5. Document `404/ResourceUnavailable` and invalid request behavior.
6. Compare it in tests with `GET /eth/v1/beacon/headers/{root}`.

Required tests:

- valid JSON/SSZ response;
- malformed block root;
- missing/non-finalized root;
- content negotiation;
- response-size bound;
- end-to-end corruption rejection after transport decoding.

Exit criteria:

- An untrusted HTTP provider can return the proof, but only the local verifier can construct `VerifiedStateRoot`.
- The first provider vertical slice requested by the Phase 2 target is complete.

### M6 - Historical LC provider readiness gate

**Estimate:** parallel dependency; ownership must be agreed  
**Do not block M3-M5 on Altair-to-head completion**

Tasks:

1. Track and review PR #9666 rather than duplicating it.
2. Require correct `NewHead` handling in data-collection vectors.
3. Add/finish idempotent historical derive-compare-persist work.
4. Expose contiguous coverage and gaps.
5. Verify bootstrap material for the exact test trusted root.
6. Verify consecutive period updates survive restart and exclude orphan winners.

Exit criteria:

- A chosen recent test root has a bootstrap and a gap-free update chain to recent finality.
- Requests outside completed coverage return unavailable, not partial success disguised as completeness.

### M7 - Transport-independent light-client consumer

**Estimate:** several PRs  
**Required for trust-minimized end-to-end startup**

Split into:

1. `LightClientStore` and official sync-processing functions.
2. Official `light_client/sync` EF test handler and vectors.
3. `LightClientDataSource` trait.
4. In-memory fixture source.
5. HTTP source using existing standard LC endpoints.
6. Retry/no-progress logic and invalid-versus-unavailable errors.

Output:

```rust,ignore
struct VerifiedFinalizedHeader<E: EthSpec> {
    header: LightClientHeader<E>,
}
```

Exit criteria:

- Starting from one trusted root, the consumer reaches a recent finalized header through an untrusted HTTP provider.
- Official sync vectors pass for supported forks.
- Gloas/Heze are rejected explicitly until implemented.

### M8 - Whole-state retrieval before chunks

**Estimate:** 3-6 days

Tasks:

1. Use `VerifiedFinalizedHeader.beacon.state_root` directly, or the verified proof result if the endpoint remains.
2. Fetch the whole state by exact root over HTTP.
3. Select the decode fork from authenticated slot/spec context.
4. Decode strictly and compute `hash_tree_root`.
5. Require equality with the verified state root.
6. Fetch the signed block by exact root and validate the state/block relationship.

Output:

```rust,ignore
struct VerifiedCheckpointAnchor<E: EthSpec> {
    state: BeaconState<E>,
    block: SignedBeaconBlock<E>,
}
```

Required rejection tests:

- corrupted state bytes;
- valid state for the wrong root;
- wrong fork decoder;
- trailing/invalid SSZ;
- mismatched block;
- provider-selected slot or fork inconsistent with verified LC context.

Exit criteria:

- Integrity is trust-minimized even though transport is still single-provider and bandwidth-heavy.

### M9 - Existing Lighthouse checkpoint handoff

**Estimate:** 3-6 days

Tasks:

1. Add an explicit startup mode rather than changing `CheckpointSyncUrl` silently.
2. Ensure only `VerifiedCheckpointAnchor` can enter the new adapter.
3. Reuse `BeaconChainBuilder::weak_subjectivity_state`.
4. Preserve genesis-root, block/state, blob/column, DB split, and fork-choice checks.
5. Start normal network and CL/EL forward sync only after chain construction.

Exit criteria:

- A fresh node starts from trusted root plus untrusted HTTP sources and reaches normal sync.
- No bare provider state/root can bypass verification.

### M10 - State manifest and byte chunks

**Estimate:** several PRs

Tasks:

1. Implement the already reviewed manifest/chunk interfaces.
2. Add deterministic manifest construction and limits.
3. Add sparse/resumable assembly and multi-source scheduling.
4. Verify checksums per chunk and the SSZ state root only after complete reconstruction.
5. Add restart, duplicate, overlap, missing-chunk, compression-bomb, and mixed-manifest tests.

Exit criteria:

- No incomplete, mixed, or corrupted chunk set can construct `VerifiedCheckpointAnchor`.

### M11 - P2P only after HTTP and consumer boundaries stabilize

There are two separate tasks:

1. **Provider P2P:** straightforward integration into the existing post-chain network using the map in Section 6.2.
2. **Consumer P2P:** requires a bootstrap-only pre-chain network or another architecture that does not require `BeaconChain`.

First architecture-spike criterion:

```text
network config + chain spec + genesis validators root + trusted root + bootnodes
  -> connect to peer
  -> issue standard LightClientBootstrap request
  -> return decoded response
without constructing BeaconChain
```

Only after that spike should snapshot, manifest, or chunk P2P protocols be added.

Exit criteria:

- Replacing `HttpLightClientDataSource` with `P2pLightClientDataSource` does not change LC or snapshot verification logic.

## 9. Immediate Step-by-Step Checklist

Work through these tasks in order:

1. Write the M0 security note, including the existing LC header and header-by-root alternatives.
2. Complete R1 and implement a throwaway proof test without changing network code.
3. Complete R2-R4 and draw the current cache/DB lifetime diagram.
4. Review the three interface definitions with the mentor.
5. Implement M3 as the first small PR.
6. Implement M4 as the second PR.
7. Implement M5 as the third PR.
8. Demonstrate valid and corrupted responses using the REST client.
9. Decide whether to keep the proof endpoint or use the verified full header directly.
10. In parallel, track the M6 provider coverage gate and begin the M7 LC consumer core.
11. Do whole-state verification before designing chunk scheduling.
12. Do P2P only after the same verifier works end to end over HTTP.

## 10. Recommended First Three PRs

### PR 1 - `Add BeaconBlockHeader state-root proof helper`

Scope:

- proof constants;
- proof construction;
- pure verification;
- corruption tests.

No `BeaconChain`, DB, REST, or P2P changes.

### PR 2 - `Serve a finalized block's verifiable state root from BeaconChain`

Scope:

- exact-root block retrieval;
- finalized/canonical gate;
- deterministic proof response;
- provider and restart tests.

No new storage column.

### PR 3 - `Add experimental Lighthouse state-root-proof REST endpoint`

Scope:

- Lighthouse-specific route;
- SSZ response and client method;
- API tests;
- end-to-end verifier use;
- documentation of redundancy with a complete verified LC header.

No libp2p changes.

## 11. Verification Commands by Area

Use targeted tests while iterating, followed by the repository-required compilation check for code changes:

```bash
cargo nextest run -p types
cargo nextest run -p merkle_proof
cargo nextest run -p beacon_chain --test beacon_chain_tests
cargo nextest run -p bn_http_api_tests
cargo nextest run -p lighthouse_network_tests
cargo nextest run -p ef_tests
cargo check
```

Run only the packages touched by a milestone during iteration. Before opening a PR, run formatting and the relevant lint/test commands required by Lighthouse's contributor guidance.

## 12. Stop Conditions

Stop and redesign before adding more code if any of these occur:

- The endpoint accepts `head`, `finalized`, or slot without an exact trusted root binding.
- Slot or fork comes only from provider metadata.
- The proof verifier lives only inside HTTP or libp2p code.
- Snapshot generation mutates LC latest-update caches.
- A new DB column duplicates deterministic data cheaply derivable from a stored header.
- Historical coverage is advertised despite period gaps or missing bootstrap branches.
- A checksum is treated as state authenticity.
- Arbitrary byte chunks are described as independently Merkle-proven.
- P2P consumer work assumes the normal `NetworkService` can start before `BeaconChain`.
- Gloas data is decoded through an older fork variant instead of being rejected.

The shortest correct path is:

```text
prove the security proposition
  -> freeze transport-independent types
  -> pure proof and corruption tests
  -> canonical provider method
  -> REST prototype
  -> LC consumer
  -> whole-state root verification
  -> checkpoint handoff
  -> chunks
  -> pre-chain P2P
```
`beacon_node/client/src/compute_light_client_updates.rs`
```rust
pub async fn compute_light_client_updates<T: BeaconChainTypes>(
    chain: &BeaconChain<T>,
    mut light_client_server_rv: Receiver<LightClientProducerEvent<T::EthSpec>>,
    beacon_processor_send: BeaconProcessorSend<T::EthSpec>,
) {
    // Should only receive events for recent blocks, import_block filters by blocks close to clock.
    //
    // Intents to process SyncAggregates of all recent blocks sequentially, without skipping.
    // Uses a bounded receiver, so may drop some SyncAggregates if very overloaded. This is okay
    // since only the most recent updates have value.
    while let Some(event) = light_client_server_rv.next().await {
        let parent_root = event.0;

        chain
            .recompute_and_cache_light_client_updates(event)
            .unwrap_or_else(|e| {
                debug!("error computing light_client updates {:?}", e);
            });

        let msg = ReprocessQueueMessage::NewLightClientOptimisticUpdate { parent_root };
        if beacon_processor_send
            .try_send(WorkEvent {
                drop_during_sync: true,
                work: Work::Reprocess(msg),
            })
            .is_err()
        {
            error!(%parent_root,"Failed to inform light client update")
        };
    }
}

```
