``
```rust
/// Given a block with a SyncAggregate computes better or more recent light client updates. The
/// results are cached either on disk or in memory to be served via p2p and rest API
pub fn recompute_and_cache_updates(
    &self,
    store: BeaconStore<T>,
    block_slot: Slot,
    block_parent_root: &Hash256,
    sync_aggregate: &SyncAggregate<T::EthSpec>,
    chain_spec: &ChainSpec,
) -> Result<(), BeaconChainError> {
    metrics::inc_counter(&metrics::LIGHT_CLIENT_SERVER_CACHE_PROCESSING_REQUESTS);
    let _timer: Option<HistogramTimer> =
        metrics::start_timer(histogram: &metrics::LIGHT_CLIENT_SERVER_CACHE_RECOMPUTE_UPDATES_TIMES);

    let signature_slot: Slot = block_slot;
    let attested_block_root: &FixedBytes<32> = block_parent_root;

    let sync_period: u64 = block_slot
        .epoch(T::EthSpec::slots_per_epoch())
        .sync_committee_period(chain_spec)?;

    let attested_block: SignedBeaconBlock<<T as BeaconChainTypes>... = store.get_blinded_block(attested_block_root)?.ok_or(…
    )?;

    let cached_parts: LightClientCachedData<<T as BeaconChainTy... = self.get_or_compute_prev_block_cache(…
    )?;

    let finalized_period: u64 = cached_parts
        .finalized_checkpoint
        .epoch
        .sync_committee_period(chain_spec)?;

    store.store_sync_committee_branch(…
    )?;

    self.store_current_sync_committee(&store, &cached_parts, sync_committee_period: sync_period, finalized_period)?;

    let attested_slot: Slot = attested_block.slot();

    let maybe_finalized_block: Option<SignedBeaconBlock<…, _>> = store.get_blinded_block(&cached_parts.finalized_block_root)?;

    let sync_period: u64 = block_slot
        .epoch(T::EthSpec::slots_per_epoch())
        .sync_committee_period(chain_spec)?;

    // Spec: Full nodes SHOULD provide the LightClientOptimisticUpdate with the highest
    // attested_header.beacon.slot (if multiple, highest signature_slot) as selected by fork choice
    let is_latest_optimistic: bool = match &self.latest_optimistic_update.read().clone() {…
    };

    if is_latest_optimistic {…
    };

    // Spec: Full nodes SHOULD provide the LightClientFinalityUpdate with the highest
    // attested_header.beacon.slot (if multiple, highest signature_slot) as selected by fork choice
    let is_latest_finality: bool = match &self.latest_finality_update.read().clone() {
        Some(latest_finality_update: &LightClientFinalityUpdate<…>) => {…
        None => true,
    };

    if is_latest_finality & !cached_parts.finalized_block_root.is_zero() {…

        let new_light_client_update: LightClientUpdate<<T as BeaconChainTypes>… = LightClientUpdate::new(
            sync_aggregate,
            block_slot,
            cached_parts.next_sync_committee,
            cached_parts.next_sync_committee_branch,
            cached_parts.finality_branch,
            &attested_block,
            maybe_finalized_block.as_ref(),
            chain_spec,
        )?;

        // Spec: Full nodes SHOULD provide the best derivable LightClientUpdate (according to is_better_update)
        // for each sync committee period
        let prev_light_client_update: Option<LightClientUpdate<…>> =
            self.get_light_client_update(&store, sync_committee_period: sync_period, chain_spec)?;

        let should_persist_light_client_update: bool =
            if let Some(prev_light_client_update: LightClientUpdate<<T as BeaconChainTypes>…>) = prev_light_client_update {
                prev_light_client_update
                    .is_better_light_client_update(&new_light_client_update, chain_spec)?
            } else {…
            };

        if should_persist_light_client_update {
            store.store_light_client_update(sync_committee_period: sync_period, &new_light_client_update)?;
            *self.latest_light_client_update.write() = Some(new_light_client_update);
        }
    }

    metrics::inc_counter(&metrics::LIGHT_CLIENT_SERVER_CACHE_PROCESSING_SUCCESSES);
    Ok(())
}
```

`SyncAggregate`
```rust
struct SyncAggregate {
    sync_committee_bits: BitVector<512>,
    sync_committee_signature: AggregateSignature,
}
```

For block B101:
```plain
Sync committee members sign root(B100)
        ↓
signatures are aggregated
        ↓
B101 includes SyncAggregate101
```
Therefore:
```plain
B101.sync_aggregate
    authenticates
B100.block_root
```