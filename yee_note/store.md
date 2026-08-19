`beacon_node/store/src/hot_cold_store.rs`
```rust
/// On-disk database that stores finalized states efficiently.
///
/// Stores vector fields like the `block_roots` and `state_roots` separately, and only stores
/// intermittent "restore point" states pre-finalization.
#[derive(Debug)]
pub struct HotColdDB<E: EthSpec, Hot: ItemStore, Cold: ItemStore> {
    /// The slot and state root at the point where the database is split between hot and cold.
    ///
    /// States with slots less than `split.slot` are in the cold DB, while states with slots
    /// greater than or equal are in the hot DB.
    pub(crate) split: RwLock<Split>,
    /// The starting slots for the range of blocks & states stored in the database.
    anchor_info: RwLock<AnchorInfo>,
    /// The starting slots for the range of blobs stored in the database.
    blob_info: RwLock<BlobInfo>,
    /// The starting slots for the range of data columns stored in the database.
    data_column_info: RwLock<DataColumnInfo>,
    pub(crate) config: StoreConfig,
    pub hierarchy: HierarchyModuli,
    /// Cold database containing compact historical data.
    pub cold_db: Cold,
    /// Database containing blobs. If None, store falls back to use `cold_db`.
    pub blobs_db: Cold,
    /// Hot database containing duplicated but quick-to-access recent data.
    ///
    /// The hot database also contains all blocks.
    pub hot_db: Hot,
    /// LRU cache of deserialized blocks and blobs. Updated whenever a block or blob is loaded.
    block_cache: Option<Mutex<BlockCache<E>>>,
    /// Cache of beacon states.
    ///
    /// LOCK ORDERING: this lock must always be locked *after* the `split` if both are required.
    pub state_cache: Mutex<StateCache<E>>,
    /// Cache of historic states and hierarchical diff buffers.
    ///
    /// This cache is never pruned. It is only populated in response to historical queries from the
    /// HTTP API.
    historic_state_cache: Mutex<HistoricStateCache<E>>,
    /// Chain spec.
    pub spec: Arc<ChainSpec>,
    /// Mere vessel for E.
    _phantom: PhantomData<E>,
}
```

```rust
impl<E: EthSpec> HotColdDB<E, MemoryStore, MemoryStore> {
    pub fn open_ephemeral(…
}

impl<E: EthSpec> HotColdDB<E, BeaconNodeBackend, BeaconNodeBackend> {
    /// Open a new or existing database, with the given paths to the hot and cold DBs.
    ///
    /// The `migrate_schema` function is passed in so that the parent `BeaconChain` can provide
    /// context and access `BeaconChain`-level code without creating a circular dependency.
    pub fn open(…
}
```
`beacon_node/store/src/database/interface.rs`
```rust
pub enum BeaconNodeBackend {
    #[cfg(feature = "leveldb")]
    LevelDb(leveldb_impl::LevelDB),
    #[cfg(feature = "redb")]
    Redb(redb_impl::Redb),
}

impl ItemStore for BeaconNodeBackend {}

impl KeyValueStore for BeaconNodeBackend {
    fn get_bytes(&self, column: DBColumn, key: &[u8]) -> Result<Option<Vec<u8>>, Error> {…

    fn put_bytes(&self, column: DBColumn, key: &[u8], value: &[u8]) -> Result<(), Error> {…

    fn put_bytes_sync(&self, column: DBColumn, key: &[u8], value: &[u8]) -> Result<(), Error> {…

    fn sync(&self) -> Result<(), Error> {…

    fn key_exists(&self, column: DBColumn, key: &[u8]) -> Result<bool, Error> {…

    fn key_delete(&self, column: DBColumn, key: &[u8]) -> Result<(), Error> {…

    fn do_atomically(&self, batch: Vec<KeyValueStoreOp>) -> Result<(), Error> {…

    fn compact(&self) -> Result<(), Error> {…

    fn iter_column_keys_from<K: Key>(…

    fn iter_column_keys<K: Key>(&self, column: DBColumn) -> ColumnKeyIter<'_, K> {…

    fn iter_column_from<K: Key>(&self, column: DBColumn, from: &[u8]) -> ColumnIter<'_, K> {…

    fn compact_column(&self, _column: DBColumn) -> Result<(), Error> {…

    fn delete_batch(&self, col: DBColumn, ops: HashSet<&[u8]>) -> Result<(), Error> {…

    fn delete_if(…
}
```
`beacon_node/store/src/memory_store.rs`
```rust
type DBMap = BTreeMap<BytesKey, Vec<u8>>;

/// A thread-safe `BTreeMap` wrapper.
pub struct MemoryStore {
    db: RwLock<DBMap>,
}

impl MemoryStore {
    /// Create a new, empty database.
    pub fn open() -> Self {…
}

impl KeyValueStore for MemoryStore {
    /// Get the value of some key from the database. Returns `None` if the key does not exist.
    fn get_bytes(&self, col: DBColumn, key: &[u8]) -> Result<Option<Vec<u8>>, Error> {…

    /// Puts a key in the database.
    fn put_bytes(&self, col: DBColumn, key: &[u8], val: &[u8]) -> Result<(), Error> {…

    fn put_bytes_sync(&self, col: DBColumn, key: &[u8], val: &[u8]) -> Result<(), Error> {…

    fn sync(&self) -> Result<(), Error> {…

    /// Return true if some key exists in some column.
    fn key_exists(&self, col: DBColumn, key: &[u8]) -> Result<bool, Error> {…

    /// Delete some key from the database.
    fn key_delete(&self, col: DBColumn, key: &[u8]) -> Result<(), Error> {…

    fn do_atomically(&self, batch: Vec<KeyValueStoreOp>) -> Result<(), Error> {…

    fn iter_column_from<K: Key>(&self, column: DBColumn, from: &[u8]) -> ColumnIter<'_, K> {…

    fn iter_column_keys<K: Key>(&self, column: DBColumn) -> ColumnKeyIter<'_, K> {…

    fn compact_column(&self, _column: DBColumn) -> Result<(), Error> {…

    fn iter_column_keys_from<K: Key>(&self, column: DBColumn, from: &[u8]) -> ColumnKeyIter<'_, K> {…

    fn delete_batch(&self, col: DBColumn, ops: HashSet<&[u8]>) -> Result<(), DBError> {…

    fn delete_if(…
}
```