`beacon_node/beacon_chain/src/beacon_chain.rs`
```rust
/// Accepts a fully-verified and available block and imports it into the chain without performing any
/// additional verification.
///
/// An error is returned if the block was unable to be imported. It may be partially imported
/// (i.e., this function is not atomic).
#[allow(clippy::too_many_arguments)]
#[instrument(skip_all)]
fn import_block(
    &self,
    signed_block: AvailableBlock<T::EthSpec>,
    block_root: Hash256,
    mut state: BeaconState<T::EthSpec>,
    payload_verification_status: PayloadVerificationStatus,
    parent_block: SignedBlindedBeaconBlock<T::EthSpec>,
    mut consensus_context: ConsensusContext<T::EthSpec>,
) -> Result<Hash256, BlockError> {
    // ======================== BLOCK NOT YET ATTESTABLE =========================
    // Everything in this initial section is on the hot path between processing the block and
    // being able to attest to it. DO NOT add any extra processing in this initial section
    // unless it must run before fork choice.
    // ===========================================================================

    let current_slot: Slot = self.slot()?;
    let current_epoch: Epoch = current_slot.epoch(T::EthSpec::slots_per_epoch());
    let block = signed_block.message();
    let post_exec_timer =
        metrics::start_timer(&metrics::BLOCK_PROCESSING_POST_EXEC_PROCESSING);

    // Check against weak subjectivity checkpoint.
    self.check_block_against_weak_subjectivity_checkpoint(block, block_root, &state)?;

    // If there are new validators in this block, update our pubkey cache.
    //
    // The only keys imported here will be ones for validators deposited in this block, because
    // the cache *must* already have been updated for the parent block when it was imported.
    // Newly deposited validators are not active and their keys are not required by other parts
    // of block processing. The reason we do this here and not after making the block attestable
    // is so we don't have to think about lock ordering with respect to the fork choice lock.
    // There are a bunch of places where we lock both fork choice and the pubkey cache and it
    // would be difficult to check that they all lock fork choice first.
    let mut ops = { ... };

    // Read the cached head prior to taking the fork choice lock to avoid potential deadlocks.
    let cached_head = self.canonical_head.cached_head();
    let old_head_slot: Slot = cached_head.head_slot();

    // Take an upgradable read lock on fork choice so we can check if this block has already
    // been imported. We don't want to repeat work importing a block that is already imported.
    let fork_choice_reader = self.canonical_head.fork_choice_upgradable_read_lock();
    if fork_choice_reader.contains_block(&block_root) { ... }

    // Take an exclusive write-lock on fork choice. It's very important to prevent deadlocks by ...
    let mut fork_choice = fork_choice_reader.upgrade();

    // Do not import a block that doesn't descend from the finalized root.
    let signed_block = check_block_is_finalized_checkpoint_or_descendant(
        self,
        &fork_choice,
        signed_block,
    )?;
    let block = signed_block.message();

    // Register the new block with the fork choice service.
    {
        let block_delay: Duration = self
            .slot_clock
            .seconds_from_current_slot_start()
            .ok_or(Error::UnableToComputeTimeAtSlot)?;

        fork_choice
            .on_block(...)
            .map_err(|e: Error<Error>| BlockError::BeaconChainError(Box::new(e.into())))?;
    }

    // If the block is recent enough and it was not optimistically imported, check to see if it
    // becomes the head block. If so, apply it to the early attester cache. This will allow
    // attestations to the block without waiting for the block and state to be inserted to the
    // database.
    //
    // Only performing this check on recent blocks avoids slowing down sync with lots of calls
    // to fork choice `get_head`.
    //
    // Optimistically imported blocks are not added to the cache since the cache is only useful
    // for a small window of time and the complexity of keeping track of the optimistic status
    // is not worth it.
    if !payload_verification_status.is_optimistic()
        && block.slot() + EARLY_ATTESTER_CACHE_HISTORIC_SLOTS >= current_slot
    {
        let fork_choice_timer =
            metrics::start_timer(&metrics::BLOCK_PROCESSING_FORK_CHOICE);

        match fork_choice.get_head(current_slot, &self.spec) {
            // This block became the head, add it to the early attester cache.
            Ok((new_head_root, _)) if new_head_root == block_root => {
                if let Some(proto_block) = fork_choice.get_block(&block_root) {
                    let new_head_is_optimistic: bool =
                        proto_block.execution_status.is_optimistic_or_invalid();

                    if let Err(e: BeaconChainError) =
                        self.early_attester_cache.add_head_block(...)
                    {
                        warn!(...);
                    } else {
                        // Register a server-sent-event for a new head.
                        if let Some(event_handler) = self
                            .event_handler
                            .as_ref()
                            .filter(|handler| handler.has_head_subscribers())
                        { ... }

                        // Register a server-sent-event for a new head v2.
                        if let Some(event_handler) = self
                            .event_handler
                            .as_ref()
                            .filter(|handler| handler.has_head_v2_subscribers())
                        { ... }
                    }
                } else { ... }
            }
            // This block did not become the head, nothing to do.
            Ok(_) => (),
            Err(e: Error<Error>) => error!(...),
        }

        drop(fork_choice_timer);
    }

    drop(post_exec_timer);

    // ======================= BLOCK PROBABLY ATTESTABLE ==========================
    // Most blocks are now capable of being attested to thanks to the `early_attester_cache`
    // cache above. Resume non-essential processing.
    //
    // It is important NOT to return errors here before the database commit, because the block
    // has already been added to fork choice and the database would be left in an inconsistent
    // state if we returned early without committing. In other words, an error here would
    // corrupt the node's database permanently.
    // ===========================================================================

    self.import_block_update_shuffling_cache(block_root, &mut state);
    self.import_block_observe_attestations(...);
    self.import_block_update_validator_monitor(...);
    self.import_block_update_slasher(block, &state, &mut consensus_context);

    // Store the block and its state, and execute the confirmation batch for the intermediate
    // states, which will delete their temporary flags.
    // If the write fails, revert fork choice to the version from disk, else we can
    // end up with blocks in fork choice that are missing from disk.
    // See https://github.com/sigp/lighthouse/issues/2028
    let (_, signed_block, block_data) = signed_block.deconstruct();

    if let Some(blobs_or_columns_store_op) =
        self.get_blobs_or_columns_store_op(block_root, signed_block.slot(), block_data)
    {
        ops.push(blobs_or_columns_store_op);
    }

    let block = signed_block.message();
    let db_write_timer =
        metrics::start_timer(&metrics::BLOCK_PROCESSING_DB_WRITE);

    ops.push(StoreOp::PutBlock(block_root, signed_block.clone()));
    ops.push(StoreOp::PutState(block.state_root(), &state));

    let db_span: EnteredSpan = info_span!("persist_blocks_and_blobs").entered();

    if let Err(e: Error) = self.store.do_atomically_with_block_and_blobs_cache(ops) { ... }

    drop(db_span);

    // The fork choice write-lock is dropped *after* the on-disk database has been updated.
    // This prevents inconsistency between the two at the expense of concurrency.
    drop(fork_choice);

    // We're declaring the block "imported" at this point, since fork choice and the DB know
    // about it.
    let block_time_imported: Duration =
        self.slot_clock.now_duration().unwrap_or(Duration::MAX);

    // compute state proofs for light client updates before inserting the state into the
    // snapshot cache.
    if self.config.enable_light_client_server {
        self.light_client_server_cache
            .cache_state_data(
                &self.spec,
                block,
                block_root,
                // mutable reference on the state is needed to compute merkle proofs
                &mut state,
            )
            .unwrap_or_else(|e: BeaconChainError| { ... });
    }

    metrics::stop_timer(db_write_timer);

    metrics::inc_counter(&metrics::BLOCK_PROCESSING_SUCCESSES);

    // Inform the unknown block cache, in case it was waiting on this block.
    self.pre_finalization_block_cache
        .block_processed(block_root);

    self.import_block_update_metrics_and_events(
        block,
        block_root,
        block_time_imported,
        payload_verification_status,
        current_slot,
    );

    Ok(block_root)
}
```

```rust
    fn import_block_update_metrics_and_events(
        &self,
        block: BeaconBlockRef<T::EthSpec>,
        block_root: Hash256,
        block_time_imported: Duration,
        payload_verification_status: PayloadVerificationStatus,
        current_slot: Slot,
    ) {
        // Only present some metrics for blocks from the previous epoch or later.
        //
        // This helps avoid noise in the metrics during sync.
        if block.slot() + 2 * T::EthSpec::slots_per_epoch() >= current_slot {
            metrics::observe(
                &metrics::OPERATIONS_PER_BLOCK_ATTESTATION,
                block.body().attestations_len() as f64,
            );

            if let Ok(sync_aggregate) = block.body().sync_aggregate() {
                metrics::set_gauge(
                    &metrics::BLOCK_SYNC_AGGREGATE_SET_BITS,
                    sync_aggregate.num_set_bits() as i64,
                );
            }
        }

        let block_delay_total =
            get_slot_delay_ms(block_time_imported, block.slot(), &self.slot_clock);

        // Do not write to the cache for blocks older than 2 epochs, this helps reduce writes to
        // the cache during sync.
        if block_delay_total < self.slot_clock.slot_duration() * 64 {
            // Store the timestamp of the block being imported into the cache.
            self.block_times_cache.write().set_time_imported(
                block_root,
                current_slot,
                block_time_imported,
            );
        }

        if let Some(event_handler) = self.event_handler.as_ref()
            && event_handler.has_block_subscribers()
        {
            event_handler.register(EventKind::Block(SseBlock {
                slot: block.slot(),
                block: block_root,
                execution_optimistic: payload_verification_status.is_optimistic(),
            }));
        }

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
            warn!(
                error = ?e,
                "Failed to send light_client server event"
            );
        }
    }

```
`beacon_node/beacon_chain/src/light_client_server_cache.rs`
```rust
    /// Compute and cache state proofs for latter production of light-client messages. Does not
    /// trigger block replay.
    pub(crate) fn cache_state_data(
        &self,
        spec: &ChainSpec,
        block: BeaconBlockRef<T::EthSpec>,
        block_root: Hash256,
        block_post_state: &mut BeaconState<T::EthSpec>,
    ) -> Result<(), BeaconChainError> {
        let _timer = metrics::start_timer(&metrics::LIGHT_CLIENT_SERVER_CACHE_STATE_DATA_TIMES);
        let fork_name = spec.fork_name_at_slot::<T::EthSpec>(block.slot());
        // Only post-altair
        if fork_name.altair_enabled() {
            // Persist in memory cache for a descendent block
            let cached_data = LightClientCachedData::from_state(block_post_state)?;
            self.prev_block_cache.lock().insert(block_root, cached_data);
        }

        Ok(())
    }
```