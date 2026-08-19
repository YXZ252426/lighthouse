[../beacon_node/http_api/src/lib.rs#L1788](../beacon_node/http_api/src/lib.rs#L1788)
```rust
    // GET beacon/light_client/updates
    let get_beacon_light_client_updates = beacon_light_client_path
        .clone()
        .and(task_spawner_filter.clone())
        .and(warp::path("updates"))
        .and(warp::path::end())
        .and(warp::query::<api_types::LightClientUpdatesQuery>())
        .and(warp::header::optional::<api_types::Accept>("accept"))
        .then(
            |light_client_server_enabled: Result<(), Rejection>,
             chain: Arc<BeaconChain<T>>,
             task_spawner: TaskSpawner<T::EthSpec>,
             query: LightClientUpdatesQuery,
             accept_header: Option<api_types::Accept>| {
                task_spawner.blocking_response_task(Priority::P1, move || {
                    light_client_server_enabled?;
                    get_light_client_updates::<T>(chain, query, accept_header)
                })
            },
        );

```

[../beacon_node/http_api/src/light_client.rs](../beacon_node/http_api/src/light_client.rs)
```rust
pub fn get_light_client_updates<T: BeaconChainTypes>(
    chain: Arc<BeaconChain<T>>,
    query: LightClientUpdatesQuery,
    accept_header: Option<api_types::Accept>,
) -> Result<Response, Rejection> {
    validate_light_client_updates_request(&chain, &query)?;

    let light_client_updates = chain
        .get_light_client_updates(query.start_period, query.count)
        .map_err(|_| {
            warp_utils::reject::custom_not_found("No LightClientUpdates found".to_string())
        })?;

    match accept_header {
        Some(api_types::Accept::Ssz) => {
            let response_chunks: Vec<u8> = light_client_updates
                .into_iter()
                .flat_map(|update| {
                    map_light_client_update_to_response_chunk::<T>(&chain, update).as_ssz_bytes()
                })
                .collect();

            Builder::new()
                .status(200)
                .body(response_chunks)
                .map(add_ssz_content_type_header)
                .map_err(|e| {
                    warp_utils::reject::custom_server_error(format!(
                        "failed to create response: {}",
                        e
                    ))
                })
        }
        _ => {
            let fork_versioned_response = light_client_updates
                .iter()
                .map(|update| map_light_client_update_to_json_response::<T>(&chain, update.clone()))
                .collect::<Vec<BeaconResponse<LightClientUpdate<T::EthSpec>>>>();
            Ok(warp::reply::json(&fork_versioned_response).into_response())
        }
    }
}

```