# 03 — Module deep-dives (CORE and MAIN)

Every CORE and MAIN module from `02-ARCHITECTURE.md`, grouped by layer.
Paths are relative to the repo root.
"LOC" counts `.rs` lines including tests.
"TODO" counts matches of `TODO|FIXME|HACK|XXX`.
Sizes were measured at `bb934c2c80`.

Contents:
1. Data model and store (`crates/store`, `crates/store_app`)
2. SDKs, CLI and data flow (`crates/top`, `crates/data_flow`, `rerun_py`, `rerun_cpp`)
3. Viewer (`crates/top/re_viewer`, `viewer_support`, `panels`, `views`, `ui_tools`)
4. Cross-cutting utilities (`crates/utils`)

---

## 1. Data model and store

Dependency counts ("rdeps") are the number of workspace `Cargo.toml` files that name the crate, counting dev-deps.

### `re_log_types` (CORE, 65 rdeps) — `crates/store/re_log_types`

- **Purpose.** The foundation identifiers and time types.
- **Interface:**
  - `EntityPath` (`src/path/entity_path.rs:94`): a precomputed 64-bit hash plus `Arc<Vec<EntityPathPart>>`.
  - `EntityPathFilter` (`src/path/entity_path_filter.rs`, 1 619 LOC, the subtlest logic in the crate).
  - `StoreId {kind, application_id, recording_id}` (`src/store_id.rs:117`) and `StoreKind {Recording, Blueprint}` (`:28`).
  - `ApplicationId` (`src/app_id.rs:36`), being phased out in favour of `EntryName`.
  - `Timeline`, `TimeType {Sequence, DurationNs, TimestampNs}`, `TimeInt`, `TimeCell`, `TimePoint` (`src/index/*`).
- **Notes.**
  - `TimeInt::STATIC` is stored as `i64::MIN` (`src/index/time_int.rs:44`).
  - Paths whose root starts with `__` are reserved (`src/path/entity_path_part.rs:7`).

### `re_types_core` (CORE, 30) and `re_sdk_types` (CORE, 54)

- **`re_types_core`** defines the traits every type implements:
  - `Archetype` (`src/archetype.rs:16`), `Component` (`src/loggable.rs:123`), `AsComponents`, `ComponentBatch`;
  - `ComponentDescriptor {archetype, component, component_type}` (`src/component_descriptor.rs:21`);
  - `RowId` (`src/row_id.rs:67`) and `ChunkId` (`src/chunk_id.rs:48`), both built on `re_tuid::Tuid` (16 bytes: big-endian nanos plus a counter, so they sort in creation order);
  - runtime `reflection`.
- **`re_sdk_types`** holds every built-in archetype, component, encoding and blueprint type. It is about 101 k LOC, and 381 of its files are generated (header "DO NOT EDIT"). There are 155 hand-written `*_ext.rs` files.
  - Generated from `crates/build/re_type_definitions/rerun/**.def.rs` by `pixi run codegen`.
  - `src/reflection/mod.rs` (6 703 LOC) is generated and is the 7th most-changed file in the last year.
- **Invariant.** `ComponentDescriptor`'s manual hash must call `write_u64` exactly once (`component_descriptor.rs:42-58`).

### `re_sorbet` (CORE, 26) — Arrow schema conventions and migrations

- **Purpose.** Defines what makes an Arrow `RecordBatch` a Rerun chunk (`ChunkBatch`, a single entity with a RowId column) or a dataframe (`SorbetBatch`, multiple entities).
- **Metadata keys.** `sorbet:version` (currently `0.1.4`), `rerun:id`, `rerun:entity_path`, and per field `rerun:kind`, `rerun:component`, `rerun:archetype`, `rerun:component_type`, `rerun:is_static`, `rerun:is_sorted` (`src/sorbet_schema.rs:75-127`, `src/component_column_descriptor.rs:204-253`, `src/metadata.rs`).
- **Migrations.** Every incoming batch is migrated when it is parsed: `SorbetBatch::try_from_record_batch` (`src/sorbet_batch.rs:213`) calls `migrate_record_batch` (`src/migrations/mod.rs`). The chain runs 0.0.1 → 0.0.2 → 0.1.0 → 0.1.1 → 0.1.2 → 0.1.3 → 0.1.4. See `04-DATA-AND-FLOWS.md` §1.3.

### `re_chunk` (CORE, 39) — the atomic data unit

- **`Chunk`** (`src/chunk.rs:242`) has these fields:
  - `id`, `entity_path`, `is_sorted`;
  - `row_ids` (FixedSizeBinary(16));
  - `timelines: IntMap<TimelineName, TimeColumn>`;
  - `components: IntMap<ComponentIdentifier, SerializedComponentColumn>`, where each column is a sparse `ListArray` with one list per row.

  A chunk always holds exactly one entity.
- **Interface:**
  - `ChunkBuilder`;
  - transport conversions (`src/transport.rs:18-269`);
  - `latest_at`/`range` on a single chunk (`src/latest_at.rs:109`);
  - `concat_and_sort`, `split_rows`;
  - `ChunkBatcher` (`src/batcher.rs:129`, not on wasm), which is used by the SDK.
- **Invariants** (`Chunk::sanity_check`, `chunk.rs:1822`):
  - RowIds are sorted if the chunk claims to be.
  - Every column length equals the row count.
  - Time columns never contain `TimeInt::STATIC`; a static chunk has no time columns.
  - A (RowId × component) cell must never change.

### `re_chunk_store` (CORE, 34) — the in-memory time-series DB

- **Purpose.** Stores and indexes chunks by entity, timeline, component and time. It is "oblivious to individual rows" (`src/store.rs:520-523`).
- **Interface:**
  - `ChunkStore` (`src/store.rs:527`) and `ChunkStoreHandle` (`Arc<RwLock<ChunkStore>>`, `:387`).
  - `ChunkStoreConfig` (`:35`). Its default is 384 KiB / 4 096 rows / 1 024 unsorted rows (`:141-150`); env `RERUN_CHUNK_MAX_BYTES`, `RERUN_CHUNK_MAX_ROWS`, `RERUN_CHUNK_MAX_ROWS_IF_UNSORTED` override it.
  - `insert_chunk` (`src/writes.rs:53`), `gc` (`src/gc.rs:192`), `latest_at_relevant_chunks` (`src/query.rs:906`), `range_relevant_chunks` (`:1231`).
  - `ChunkStoreEvent`/`ChunkStoreDiff`, and `ChunkStoreSubscriber` registered in the process-global `SUBSCRIBERS` (`src/subscribers.rs:86`).
  - `LazyStore` (manifest-backed; used by `re_server`).
- **Data owned.**
  - All physical chunks of one store.
  - Temporal indexes `Entity → Timeline → Component → {per_start_time, per_end_time, max_interval_length}`, which also include *virtual* (not loaded) chunks.
  - The static index `Entity → Component → ChunkId`.
  - Lineage (splits and compactions).
- **Insert pipeline.**
  1. Reject unsorted RowIds.
  2. Split chunks that are too large.
  3. Static: per component, the newest RowId wins.
  4. Temporal: elect a neighbour to compact with (`src/compaction_election.rs:22`) and merge.
  5. Emit events to subscribers.
- **Risks.**
  - Subscribers are global, so they see events from every store.
  - Static `Transform3D`/`Pinhole` chunks are deliberately never dropped (`src/writes.rs:34`, RR-4887).
  - The `chunk_max_bytes` doc comment says 8 MiB, but the default is 384 KiB (`store.rs:57` vs `:150`).
  - 35 TODOs.

### `re_query` (MAIN, 13) — query cache

- **Purpose.** Wraps the store with cached latest-at, range and latest-all queries, and applies query-time `Clear` semantics.
- **Interface:**
  - `QueryCache` (`src/cache.rs:136`), keyed by `QueryCacheKey {entity_path, timeline, component}`;
  - `LatestAtResults`, `RangeResults`;
  - `StorageEngine` (`src/storage_engine.rs:20-66`), which pairs the store and cache locks. Its constructor is `unsafe` so that guards cannot escape; this is the main defence against deadlocks.
- **Invalidation.** `on_events` (`cache.rs:312`). Latest-at invalidations are deferred into `pending_invalidations` and applied on the next query (`src/latest_at.rs:626-632`).
- **Risks.**
  - Stale cached data can survive if an invalidation lands before the data is cached (TODO #5974, `latest_at.rs:794-802`).
  - A partial result discards the whole query (TODO RR-3762, `:740`).

### `re_entity_db` (CORE for the viewer, 24) — per-recording database

- **`EntityDb`** (`src/entity_db.rs:127`) wraps one `StorageEngine` (store + cache, never exposed publicly), plus:
  - the entity path set, the time histogram per timeline, ingestion stats;
  - `RrdManifestIndex` (with a chunk prioritizer/prefetcher for lazy loading, `src/rrd_manifest_index/chunk_prioritizer.rs`, 1 833 LOC).
- **Interface:**
  - ingestion: `add_log_msg` (`:847`), `add_chunk` (`:884`);
  - queries: `latest_at` (`:607`);
  - GC and dropping: `gc_with_target` (`:950`), `drop_time_range` (`:1031`);
  - export and cloning: `to_messages` (`:1092`), `clone_with_new_id` (`:1164`).
- **`StoreBundle`** (`src/store_bundle.rs:21`) maps `StoreId` → `EntityDb`.

### `re_log_msg` (CORE, 25) and `re_log_encoding` (CORE, 17)

- **`re_log_msg`** (617 LOC):
  - `LogMsg {SetStoreInfo, ArrowMsg(StoreId, ArrowMsg), BlueprintActivationCommand}` (`src/log_msg.rs:66`). It is slated for removal (TODO #8631).
  - `ArrowMsg` wraps `Arc<ChunkBatch>` and has an `on_release` callback.
  - `TableMsg` travels separately and is never stored in `.rrd`.
- **`re_log_encoding`** owns the `.rrd` byte format and the transport ↔ application conversion.
  - `ToTransport`/`ToApplication` (`src/transport_to_app.rs:49,129`).
  - `Encoder` and the `Decoder` state machine (`src/rrd/decoder/state_machine.rs`).
  - `FileSink`.
  - Footer and manifest reading for random access (`src/rrd/footer_reader.rs`, `chunk_reader.rs`).

  The format is described in `04-DATA-AND-FLOWS.md` §1.2.
- **Risk.** With footers disabled, `Encoder::finish` returns before writing an End marker (`src/rrd/encoder.rs:384`). Its doc (`:365-371`) says it always writes one. Whether this is intended is UNVERIFIED.

### `re_protos` (CORE, 18) — every wire format

- **Proto sources.** `crates/store/re_protos/proto/rerun/v1alpha1/{common,log_msg,sdk_comms,viewer_control,cloud}.proto`.
- **Generated Rust.** `src/v1alpha1/rerun.<pkg>.v1alpha1.rs` ("@generated by prost-build"; do not edit).
- **Hand-written extensions.** `src/v1alpha1/*.ext*.rs`, for example `rerun.cloud.v1alpha1.ext.rs` (3 635 LOC) and `…ext.legacy_adapter.rs`, which answers the deprecated `QueryDataset` via `FindChunks`.
- **Services** (full RPC list in `04-DATA-AND-FLOWS.md` §2):
  - `MessageProxyService`
  - `ViewerControlService`
  - `RerunCloudService` (42 RPCs)

### `re_chunk_index` (MAIN, 21) — manifests for lazy loading

- **Purpose.** Describes which chunks exist and where they are, without the chunk data itself.
- **Interface:**
  - `RawRrdManifest` (`src/raw_rrd_manifest.rs:146`) has one row per chunk: `chunk_id`, `chunk_entity_path`, `chunk_byte_offset`, `chunk_byte_size`, and per-timeline and per-component time ranges.
  - `RrdManifest`.
  - `ChunkProvider` (async `load_chunks`).
- **Users.** The viewer's lazy loading, `re_server`, and the `rerun rrd` tooling.

### `re_dataframe` (MAIN) and `re_datafusion` (MAIN)

- **`re_dataframe`** is the public dataframe API.
  - Interface: `QueryEngine` (`src/engine.rs:23`; `from_rrd_filepath` `:47`, `query` `:122`) and `QueryHandle` (`src/query.rs:242`; `next_row`, `next_row_batch`, `iter`).
  - Risks: `query.rs` is a 5 521-LOC monolith. An `unsafe` no-op waker at `:3660-3710` could be replaced by `std::task::Waker::noop()`.
- **`re_datafusion`** (31.5 k LOC, the largest store_app crate) provides DataFusion `TableProvider`s over redap.
  - Interface: `RedapCatalogProviderList`, `DataframeQueryTableProvider`, `LocalChunkStoreTableProvider`, plus filter pushdown (`src/pushdown_expressions.rs`).
  - **Two pipelines.** `RERUN_DATAFRAME_PIPELINE` selects one (`src/pipeline/config.rs:19-27`):
    - `v2` (streaming) is the default;
    - `v1` (IO loop + CPU workers) must be requested explicitly.
  - Risk: `PIPELINE_V2.md` still says "proposal, not yet implemented".

### `re_server` (MAIN) — OSS catalog server

- **Purpose.** The open-source implementation of most of `RerunCloudService`. It powers `rerun server` and `rr.server.Server`, and is not the same as `rerun --serve-grpc` (`crates/store_app/re_server/README.md`).
- **Storage.**
  - `InMemoryStore` (`src/store/in_memory_store.rs:39`) holds datasets (segments × layers) and tables.
  - An RRD with a footer becomes a `LazyStore` that reads chunks from disk on demand. One without a footer is loaded eagerly (`src/store/resolved_store.rs:16-160`).
  - Catalog metadata lives in memory only.
  - Lance tables require feature `lance`.
- **Unimplemented in OSS.**
  - `DoMaintenance` and `DoGlobalMaintenance`.
  - The multipart-upload RPCs (`src/rerun_cloud/mod.rs:1190-2178`).
- **Risks.**
  - `LazyStore` has no cache (TODO RR-4503).
  - `TaskRegistry` grows without bound.
  - Positional reads block the reactor (`resolved_store.rs:119`).

### Supporting store crates

- `re_uri` (URL schemes, `src/redap_uri.rs:10`).
- `re_log_channel` (`LogSender`/`LogReceiver` and `DataSourceMessage`; `src/data_source_message.rs:21-42`).
- `re_lenses_core` (lens AST plus a small selector language).
- `re_tf`:
  - It is SUPPORTING by fan-in but on every spatial view's hot path.
  - It resolves the transform-frame forest, using `TransformResolutionCache` (`src/transform_resolution_cache/cache.rs:40`).
- `re_catalog_client` (high-level catalog API shared by Python and the viewer; `src/lib.rs:40`).

PERIPHERAL:
- `re_chunk_optimizer` (`rerun rrd optimize`)
- `re_arrow_json` (JSON blueprints over ViewerControl)

---

## 2. SDKs, CLI and data flow

### `rerun-cli` (CORE) — `crates/top/rerun-cli`

- **Purpose.** Builds the shipped `rerun` binary (`[[bin]] rerun`, `crates/top/rerun-cli/Cargo.toml:32-34`).
- **Entry point.** `src/bin/rerun.rs:19` `main()`. It installs `re_memory::AccountingAllocator<mimalloc::MiMalloc>` as the global allocator (`:15-17`), then calls `rerun::run`.
- **Features.** Default features are `web_viewer, base`, and `base` = `audio, native_viewer, map_view, oss_server, perf_telemetry`. The release features (`release_*`) add `nasm` and `lance` (`Cargo.toml:47-104`).
- **Risk.** The bin and lib names collide, as noted in the comment at `Cargo.toml:35-36`.

### `rerun` (CORE) — `crates/top/rerun`

- **Purpose.** The Rust SDK facade (it re-exports `re_sdk`) plus every CLI command implementation.
- **Public interface:**
  - `run`, `CallSource` (`src/lib.rs:138`);
  - modules `clap`, `native_viewer`, `log_integration::Logger`;
  - `EXTERNAL_IMPORTER_PREFIX` (`src/lib.rs:165`, a duplicate of the `re_importer` constant).
- **Key files.** `src/commands/entrypoint.rs` (2 454 LOC) holds:
  - `struct Args` `:137`, `enum Command` `:637`, `run` `:721`, `run_inner` `:739`, `run_impl` `:1036`;
  - `start_native_viewer` `:1220`, `connect_to_existing_server` `:1482`, `serve_web` `:1522`, `serve_grpc` `:1593`, `save_or_test_receive` `:1625`.

  The subcommands live in `src/commands/{rrd,mcap,analytics,auth.rs,download.rs}`.
- **Mode selection** (`run_impl`), in order:
  1. `--test-receive` or `--save`
  2. `--serve-grpc`
  3. `--serve-web`
  4. a viewer already listening on the port → forward to it
  5. otherwise start the native viewer
- **Risks.**
  - `entrypoint.rs` is a god-file with heavy feature gating, and it is the 6th most-changed file in the last year.
  - It calls `unsafe std::env::set_var("OTEL_SERVICE_NAME")` (`:763-767`).
  - `--bind` defaults to `0.0.0.0` (`:163`).
  - TODO count: 42.

### `re_sdk` (CORE) — `crates/top/re_sdk`

- **Purpose.** The logging SDK used by Rust directly and by Python and C/C++ through bindings.
- **Public interface:**
  - `RecordingStream`, `RecordingStreamBuilder` (`src/recording_stream.rs:175`), whose constructors are `buffered` `:396`, `memory` `:418`, `set_sinks` `:448`, `connect_grpc[_opts]` `:466/:483`, `serve_grpc[_opts]` `:511/:538`, `save` `:567`, `stdout` `:586`, `spawn[_opts]` `:610/:634`;
  - sinks (`src/log_sink.rs`): `BufferedSink`, `MemorySink`, `CallbackSink`, `GrpcSink`, `MultiSink`, `BinaryStreamSink`, `GrpcServerSink`, plus `FileSink` (defined in `re_log_encoding`);
  - `spawn`/`SpawnOptions` (`src/spawn.rs`), `DEFAULT_SERVER_PORT`, `DEFAULT_CONNECT_URL` (`src/lib.rs:40,45`).
- **Internals.** `RecordingStreamInner::new` (`recording_stream.rs:989`) sets up the stream in this order:
  1. resolves the batcher config (`:941`), honouring the `RERUN_FLUSH_*` env vars;
  2. creates the `ChunkBatcher` (`re_chunk/src/batcher.rs:129`);
  3. sends `SetStoreInfo`;
  4. creates a byte-bounded command channel (`max_bytes_in_flight / 2`);
  5. starts the `forwarding_thread` (`:1644`).

  Data flows `record_row` (`:1858`) → `ChunkBatcher::push_row` → `batching_thread` → `forwarding_thread` → `LogSink::send`.
- **Global state.** Global and thread-local recording slots `GLOBAL_DATA_RECORDING` and `GLOBAL_BLUEPRINT_RECORDING` (`src/global.rs:67,72`). `RecordingStream::get` resolves an explicit stream first, then the thread-local, then the global.
- **Flush.** `flush_blocking`, `flush_with_timeout` and `flush_async` (`recording_stream.rs:2058-2165`) do four things:
  1. wait for importer threads;
  2. flush the batcher;
  3. send `PopPendingChunks`;
  4. send a sink flush.

  An unconnected gRPC sink waits at most 5 s (`re_grpc_client/src/write.rs:80-87`).
- **Risks.**
  - **Bug:** `src/global.rs:94` calls `forget_global(StoreKind::Recording)` in the *blueprint* branch of `cleanup_if_forked_child`.
  - **Possible bug:** `on_release` is set on the drain path (`recording_stream.rs:1764`) but not on the `select!` path (`:1777-1785`).

  See `06-RISKS-AND-TODO.md`.

### `rerun_py` (CORE) — Python SDK

- **Layout.**
  - Rust `cdylib` `rerun_bindings` (`rerun_py/Cargo.toml:14-15`), with `#[pymodule]` at `rerun_py/src/python_bridge.rs:184` and about 100 functions.
  - The Python package is `rerun_py/rerun_sdk/rerun/`. `rerun_py/rerun_sdk/rerun_cli/` is the console script wrapping the bundled binary.
  - Type stubs `rerun_py/rerun_bindings/rerun_bindings.pyi` are maintained by hand (2 253 lines; 2nd most-changed file in the last year). `scripts/ci/python_check_signatures.py` checks them.
- **Key modules:**
  - `rerun/__init__.py` (`init` `:278`), `recording_stream.py`, `_log.py` (`log` `:20`), `sinks.py`;
  - `catalog/_entry.py` (2 138 LOC, redap client API);
  - `experimental/`, `notebook.py`.
- **Deps.** Nearly every `data_flow` and `store_app` crate, plus `pyo3`, `numpy`, `arrow`, `datafusion` (+ffi), `tokio`, `tonic`.
- **Global state** (`python_bridge.rs`):
  - `ALL_RECORDINGS` `:55`;
  - `GARBAGE_QUEUE` `:94`, which defers dropping Python-owned Arrow buffers until the GIL can be taken safely (`:70-103`);
  - `WEB_HANDLE`, `PROFILER`, `TELEMETRY`;
  - a tokio `RUNTIME` (`rerun_py/src/utils.rs:17`).
- **Risks.**
  - Arrow C-FFI buffers interact with the GIL, which makes deadlocks easy to introduce.
  - The hand-maintained `.pyi` stubs can drift.
  - The wheel bundles the `rerun` binary (`rerun_py/pyproject.toml:254-270`).

### `rerun_c` (MAIN) and `rerun_cpp` (MAIN)

- **`crates/top/rerun_c`** is a `staticlib` C ABI over `re_sdk` (`Cargo.toml:31`).
  - It exports 28 `extern "C"` functions in `src/lib.rs`, for example `rr_recording_stream_new` `:626`, `_log` `:1210` and `_send_columns` `:1402`.
  - Integer handles map to streams through `RECORDING_STREAMS` (`src/recording_streams.rs:41`).
- **`rerun_cpp`** is the C++ SDK: hand-written core (`src/rerun/recording_stream.{hpp,cpp}`) plus generated archetypes and components.
  - It links a prebuilt `rerun_c` per platform (`rerun_cpp/CMakeLists.txt:44-86`). There is no prebuilt for Intel macOS (`:53`).
  - Arrow C++ is downloaded and built by default (`RERUN_DOWNLOAD_AND_BUILD_ARROW=ON`, `:117`).
- **Risks.**
  - `rerun_cpp/src/rerun/c/rerun.h` is kept in sync with `rerun_c/src/lib.rs` by hand. A version handshake catches mismatches (`rerun.h:485`).
  - There is no `catch_unwind`. A Rust panic aborts the host process, because release builds use `panic = "abort"` (`Cargo.toml:761`).
  - `rerun_c` has 36 `unsafe` lines.

### `re_data_source` (CORE) — `crates/data_flow/re_data_source`

- **Purpose.** Turns a CLI argument, URL or dropped file into a `LogReceiver`.
- **Interface.** `LogDataSource` (`src/data_source.rs:21`), with variants File, FileContents, HttpUrl, Stdin, RedapDatasetSegment and RedapProxy.
  - `from_uri` `:86` classifies the input using heuristics (`:93-164`).
  - `stream_with_options` `:248` starts the streaming.
- **Risk.** `from_uri` is heuristic. An extension is required for files. A bare host gets `http://` prepended (`:160-164`).

### `re_grpc_server` (CORE) and `re_grpc_client` (CORE) — legacy "message proxy"

- **Server** (`crates/data_flow/re_grpc_server/src/lib.rs`, 2 166 LOC).
  - Implements `MessageProxyService` (`WriteMessages`, `ReadMessages`, `WriteTable`, `ReadTables`; `crates/store/re_protos/proto/rerun/v1alpha1/sdk_comms.proto:30`).
  - Its in-memory history is GC'd to `--server-memory-limit`, default 1 GiB (`:64-90`, `gc` `:967`).
  - It also serves `ViewerControlService` behind a `LoopbackOnly` interceptor (`:197`).
  - Entry points: `serve` `:295`, `spawn_with_recv_and_services` `:673`. There is an optional `[[bin]] re_grpc_server` (feature `bin`).
- **Client** (`crates/data_flow/re_grpc_client`).
  - `write::Client` (`src/write.rs:133`) runs a dedicated `message_proxy_client` thread with reconnect backoff from 100 ms to 1 s (`:373`).
  - `read::stream` (`src/read.rs:12`) is the read side.
- **Risks.**
  - There is no authentication.
  - The default bind is `0.0.0.0`.
  - `max_decoding_message_size = u32::MAX` (`lib.rs:45`).
  - The client queue is bounded by message count, not bytes (TODO RR-3869, `write.rs:134`).

### `re_importer` (CORE) and `re_mcap` (MAIN)

- **`re_importer`** (`crates/data_flow/re_importer`) is the pluggable file-import system.
  - The `Importer` trait is in `src/lib.rs`. Built-in importers (`BUILTIN_IMPORTERS`, `:531-549`): Rrd, Archetype (images, meshes, audio, point clouds, text), Directory, Mcap, Parquet, LeRobot, External, Puffin, Urdf.
  - `import` (`src/import_file.rs:136`) runs every importer on a file in parallel with rayon. An importer that cannot handle the file returns `Incompatible`.
  - External plugins are any `$PATH` executable named `rerun-importer-*` (`src/importer_external.rs:16`). Each one gets the file path plus `ImporterSettings::to_cli_args` (`lib.rs:193`) and must write an RRD stream to stdout. Exit code 66 means "not mine".
- **`re_mcap`** (`crates/data_flow/re_mcap`, 9.2 k LOC) converts MCAP to chunks.
  - It covers protobuf reflection, ROS 2 CDR (via `re_ros_msg`), raw messages, attachments and recovery.
  - Hotspots: `decoders/protobuf.rs` (1 977 LOC), `decoders/mod.rs`.
  - The MCAP importer memory-maps files with `unsafe Mmap::map` (`re_importer/src/importer_mcap/importer.rs:258`).

### `re_redap_client` (MAIN) — Rerun Data Protocol client

- **Purpose.** The official client for `RerunCloudService` (42 RPCs in `crates/store/re_protos/proto/rerun/v1alpha1/cloud.proto`). It is used by the viewer, the CLI and Python. 14 workspace crates depend on it.
- **Interface:**
  - `ConnectionRegistry::{new_with_stored_credentials, new_without_stored_credentials}` (`src/connection_registry.rs:62,80`);
  - `ConnectionClient` (`src/connection_client/`, split into submodules on 2026-11-04);
  - `stream_blueprint_and_segment_from_server`;
  - the `with_retry*` helpers (`src/lib.rs:235,263`).
- **Auth.** Credentials are tried in this order (`connection_registry.rs:325-400`):
  1. stored per-origin credentials;
  2. the fallback token;
  3. the `REDAP_TOKEN` env var;
  4. no token.
- **TLS.** `channel()` is at `src/grpc.rs:40`. `RERUN_REDAP_LOCAL_CERT_PATH` adds a custom CA and hardcodes SNI `localhost` (`:47-66`).
- **Hotspots.** `grpc.rs` (1 404 LOC); TODO count 20.

---

## 3. Viewer

### Frame loop at a glance

```
eframe::App::logic  → AppCore::logic_impl            crates/top/re_viewer/src/app/logic.rs:27
   purge_memory_if_needed (:828) → receive_messages (:154, 10 ms budget at :261)
   → serve agent viewer-control calls → receive_fetched_chunks → prefetch_chunks (:952)
   → run_pending_system_commands
eframe::App::ui     → AppCore::frame_ui                app/mod.rs:1375
   move_time → store_hub.begin_frame_caches → store_hub.read_context(route)
   → AppState::show (app_state.rs:303) → viewport → views → re_renderer callbacks
   → run_pending_{ui,recording,system}_commands         app/command_handling.rs:54,1274,42
```

### `re_viewer` (CORE) — `crates/top/re_viewer` (30 k LOC)

- **Purpose.** The eframe app that hosts every panel and view.
- **Public interface** (`src/lib.rs`):
  - `App`, `AppCore` (`:79`), `StartupOptions`;
  - `run_native_app` (`:111`), `run_headless_app` (`:116`), `customize_eframe_and_setup_renderer` (`:260`);
  - `reset_viewer_persistence` (`:297`);
  - on wasm, `WebHandle` (`src/web.rs:30`).
- **Key files:**
  - `app/mod.rs`: `AppCore` struct with about 35 fields (`:125-201`), constructor `:229`.
  - `app/logic.rs`, `app/command_handling.rs` (2 011 LOC).
  - `app_state.rs` (`AppState::show` `:303`).
  - `app/registries.rs` (`Registries::new` `:30`).
  - `default_views.rs` (`:23`, registers all 13 view classes).
  - `app/viewer_control/mod.rs` (3 137 LOC, the ViewerControl gRPC implementation).
  - `saving.rs` (blueprint `.rbl` persistence).
  - `web.rs` (wasm exports).
- **State owned:**
  - the `StoreHub` with every `EntityDb`;
  - persisted `AppState` (RON under the eframe `APP_KEY`; `app/mod.rs:296-313`);
  - log receivers, the command channel, the connection registry, notifications and the agent panel.
- **Memory.**
  - The default limit is 75 % of RAM on native and 2.5 GB on wasm (`crates/utils/re_memory/src/memory_limit.rs:40-49`, used at `re_viewer_context/src/app_options.rs:144`).
  - When the limit is exceeded, the viewer purges app caches first, then a fraction of the store (`StoreHub::purge_fraction_of_ram`, `store_hub/mod.rs:744`), then the network chunk cache.
- **Risks.**
  - `AppCore` is a god object.
  - `command_handling.rs` and `viewer_control/mod.rs` are very large.
  - Blueprints are not persisted on web (`app/mod.rs:1811`, TODO #2579).
  - Loading into an already-active blueprint only warns (TODO #5514).

### `re_viewer_context` (CORE) — `crates/viewer_support/re_viewer_context` (32 k LOC, 32 dependents)

- **Purpose.** Shared viewer state and the contracts that views implement.
- **Key types:**
  - `ViewClass` (`src/view/view_class.rs:219`; `ui(...)` `:391`, `spawn_heuristics` `:330`);
  - `VisualizerSystem` (`src/view/visualizer_system.rs:318`; `visualizer_query_info` `:326`, `execute` `:339`);
  - `ViewContextSystem` (`src/view/view_context_system.rs:17`), `ViewClassRegistry` (`src/view/view_class_registry.rs:199`);
  - `ViewerContext`, `StoreHub` (`src/store_hub/mod.rs:124`), `AppOptions` (`src/app_options.rs:11`), `TimeControl`;
  - `CommandSender`/`SystemCommand`, `ComponentUiRegistry`, `FallbackProviderRegistry`;
  - the GPU bridge (`src/gpu_bridge/re_renderer_callback.rs`), caches (`src/cache/`, including the 2.3 k-LOC video stream cache), `viewer_url.rs`, `undo.rs`.
- **Invariant.** `StoreHub` does **not** track the active recording. The active recording is derived from the current route (doc comment at `store_hub/mod.rs:106-110`).
- **Risk.** It has the highest TODO count in viewer_support (84).

### `re_viewport_blueprint` (CORE) and `re_viewport` (CORE)

- **`re_viewport_blueprint`** reads the blueprint store into a model:
  - `ViewportBlueprint::from_db` (`src/viewport_blueprint.rs:80`), `spawn_heuristic_views` `:304`, `save_to_blueprint_store` `:932`;
  - `ViewBlueprint`, `ContainerBlueprint`, `ViewContents`;
  - `query_results_for_views` (`src/data_query_result_cache.rs:29`), `auto_layout::tree_from_views`.

  Layout edits are deferred as `ViewportCommand`s and applied at the end of the frame.
- **`re_viewport`** is small (1.3 k LOC) but on the hot path:
  - `ViewportUi::viewport_ui` (`src/viewport_ui.rs:55`);
  - `execute_systems_for_all_views` (`src/system_execution.rs:9`) runs rayon over the active `egui_tiles` tiles;
  - `TilesDelegate::pane_ui` (`:312`) calls each view's `ui`.

### `re_view` (CORE) — `crates/viewer_support/re_view`

- **Purpose.** Helpers for implementing views.
  - `execute_systems_for_view` (`src/system_execution.rs:121`) runs context systems and visualizers in parallel.
  - Blueprint-resolved queries: `latest_at_with_blueprint_resolved_data` (`src/query.rs:450`), `range_with_blueprint_resolved_data` (`:260`), `query_archetype_with_history` (`:763`).
- **Component value priority** (`src/query.rs:248-256`), highest first:
  1. blueprint override
  2. annotation context
  3. recording data
  4. view default
  5. visualizer fallback
  6. placeholder

### Views (MAIN unless noted) — `crates/views/*`

All 13 view classes are registered in `crates/top/re_viewer/src/default_views.rs:23-98`. Each view crate has exactly one normal dependent, `re_viewer`.

| Crate | Class(es) | LOC | Notes |
| --- | --- | --- | --- |
| `re_view_spatial` (CORE) | `SpatialView2D`, `SpatialView3D` | 28.8 k | About 24 visualizers (`src/visualizers/mod.rs:77-132`); transform tree context; picking; camera in `eye.rs`; 227 snapshot PNGs; 82 TODOs |
| `re_view_time_series` | `TimeSeriesView` | 10.3 k | `view_class.rs` is 2 477 LOC; uses `egui_plot` |
| `re_view_dataframe` | `DataframeView` | 2.8 k | Table of entity data |
| `re_view_tensor` | `TensorView` | 2.6 k | N-D tensor slicing |
| `re_view_text_log` | `TextView` | 4.3 k | Log table that scrolls with time |
| `re_view_state_timeline` (SUPPORTING) | `StateTimelineView` | 6.3 k | New; uses `re_time_ruler` |
| `re_view_graph`, `re_view_bar_chart`, `re_view_text_document`, `re_view_raw_data`, `re_view_map` (feature `map_view`), `re_view_audio` (feature `audio`) | — | 0.6–2.5 k | SUPPORTING or optional |

### Panels — `crates/panels/*`

- **MAIN:**
  - `re_selection_panel` (`selection_panel.rs` 2 345 LOC)
  - `re_time_panel` (`time_panel.rs` 2 068 LOC)
  - `re_blueprint_tree`
  - `re_recording_panel`
  - `re_redap_browser` (9.8 k LOC, catalog browsing)
- **SUPPORTING:** `re_chunk_store_ui`
- **PERIPHERAL:**
  - `re_agent_panel`, which is experimental and gated by `AppOptions.experimental.agent_panel` (default false, `re_viewer_context/src/app_options.rs:258`);
  - `re_memory_view`, which is used only by the dev panel.

### `re_renderer` (CORE) — `crates/ui_tools/re_renderer` (29 k LOC, 110 TODOs)

- **Purpose.** A custom renderer on wgpu, independent of Rerun's data model.
- **`RenderContext`** (`src/context.rs:93`) holds:
  - device and queue, `DeviceCaps`;
  - lazily-created renderers;
  - texture managers, `CpuWriteGpuReadBelt`;
  - `WgpuResourcePools`: static pools for pipelines, layouts and shaders, and dynamic, frame-reclaimed pools for buffers, textures and bind groups.
- **`ViewBuilder`** (`src/view_builder.rs`) is created once per view per frame:
  - `queue_draw` `:736`;
  - `draw` `:751` renders the draw phases (`src/draw_phases/mod.rs:31`: Opaque, Background, Transparent, Volume, PickingLayer, OutlineMask…, Compositing) into the view's own targets;
  - `composite` `:1004` draws the result into egui's pass.
- **Renderers** (`src/renderer/`): point cloud, lines, rectangles, mesh, depth cloud, gaussian splat, volume, voxel grid, world grid, skybox, compositor.
- **Shaders.** 41 `.wgsl` files in `crates/ui_tools/re_renderer/shader/` with a custom `#import` system.
  - Release and wasm builds embed them; debug builds hot-reload them from disk (`build.rs:134-138`).
- **Backends.**
  - Native: Vulkan, Metal or GL.
  - Web: WebGPU with a WebGL2 fallback (`src/device_caps.rs:510-521`). WebGL2 is the `Limited` capability tier, which has fewer features.

### `re_ui` (CORE) — `crates/ui_tools/re_ui` (22 k LOC, 36 dependents)

- **Purpose.** Rerun's egui theme and widgets:
  - design tokens, hot-reloadable (`src/design_tokens.rs`, `hot_reload_design_tokens.rs`);
  - `UiExt`, `list_item`, `notifications`, `CommandPalette`, `modal`, icons.
- **Testing.** It provides the kittest harness builder used by snapshot tests (`src/testing/mod.rs:30-80`). Pixel thresholds are strict only on Linux CI (lavapipe).

### `re_dataframe_ui` (MAIN) — `crates/viewer_support/re_dataframe_ui` (17.6 k LOC)

- **Purpose.** A DataFusion-backed table widget (`datafusion_table_widget.rs`, 2 275 LOC) with column filters and table blueprints.
- **Users.** The dataframe view and the redap browser.

---

## 4. Cross-cutting utilities (CORE ones)

| Crate | Dependents (non-dev) | Why it matters | Key file |
| --- | --- | --- | --- |
| `re_log` | 91 | `tracing`-based logging. `RUST_LOG` filter defaults to `info` in release and `debug` in debug builds. `RERUN_PANIC_ON_WARN` turns warnings into panics in tests. | `crates/utils/re_log/src/lib.rs:176-206`, `src/setup.rs` |
| `re_tracing` | 70 | puffin `profile_function!`/`profile_scope!`. `rerun --profile` starts a puffin server on `0.0.0.0:8585`. | `crates/utils/re_tracing/src/lib.rs`, `src/server.rs:29` |
| `re_byte_size` | 47 | `SizeBytes` trait used for memory accounting everywhere | `crates/utils/re_byte_size/src/lib.rs` |
| `re_format` | 43 | number, duration and timestamp formatting | `crates/utils/re_format/src/lib.rs` |
| `re_arrow_util` | 32 | Arrow helpers | `crates/utils/re_arrow_util/src/` |
| `re_quota_channel` | 20 | byte-size-bounded channels; this is how backpressure works | `crates/utils/re_quota_channel/src/sync.rs` |
| `re_tuid` | 14 | time-ordered 128-bit IDs (RowId and ChunkId are built on it) | `crates/utils/re_tuid/src/lib.rs` |
| `re_memory` | 8 | `AccountingAllocator`, `MemoryLimit` | `crates/utils/re_memory/src/accounting_allocator.rs`, `memory_limit.rs` |
| `re_video` | 12 (MAIN) | MP4/AV1/H.264/H.265 demux and decode: dav1d fork natively, WebCodecs on web, the **ffmpeg CLI** for H.264/H.265 | `crates/utils/re_video/src/decode/` |
| `re_auth` | 6 (MAIN) | WorkOS OAuth plus JWT; tokens in `~/.config/rerun/credentials.json` | `crates/utils/re_auth/src/oauth/storage.rs:65` |
| `re_analytics` | 9 (MAIN) | opt-out PostHog analytics via `tel.rerun.io` | `crates/utils/re_analytics/src/lib.rs:327-345` |
