# 06 — Risks, debt, must-dos, open questions

Priority key:

- **P0:** a correctness or security issue to fix or explicitly accept soon.
- **P1:** a reliability, process or scale risk.
- **P2:** hygiene, doc drift or cleanup.

"Verified" means a human or the lead auditor read the cited lines.
Every other claim cites its file but was reported by a sub-audit.

## 1. Prioritized must-do list

### P0

| # | Issue | Evidence | Suggested action |
| --- | --- | --- | --- |
| 1 | **Bug: the wrong store kind is forgotten after a fork.** In `cleanup_if_forked_child`, the *blueprint* branch calls `forget_global(StoreKind::Recording)`. After `os.fork()` the stale global blueprint survives, and a valid data recording is dropped. | `crates/top/re_sdk/src/global.rs:90-95` (verified) | Change it to `StoreKind::Blueprint` and add a fork test. |
| 2 | **Possible bug: Python's garbage queue is bypassed on one path.** `forwarding_thread` sets `msg.on_release` on the drain path but not on the `select!` path. Python-owned Arrow buffers sent on that path may be released without going through `GARBAGE_QUEUE`, which exists to avoid GIL deadlocks. | `crates/top/re_sdk/src/recording_stream.rs:1757-1765` vs `:1777-1785` (verified code); `rerun_py/src/python_bridge.rs:70-103`. Runtime impact is UNVERIFIED. | Confirm with the SDK owners. Most likely fix: add `msg.on_release.clone_from(&on_release)` on both paths. |
| 3 | **Unauthenticated network listeners on all interfaces by default.** The gRPC proxy (`WriteMessages`, plus `ReadMessages`, which replays full history) has no auth and binds `0.0.0.0`. The web viewer and the OSS server bind `0.0.0.0` too, and `--profile` serves puffin on `0.0.0.0:8585`. | `crates/top/rerun/src/commands/entrypoint.rs:163`, `crates/top/rerun/src/clap.rs:89`, `crates/top/re_sdk/src/web_viewer.rs:190`, `crates/store_app/re_server/src/entrypoint.rs:19`, `crates/utils/re_tracing/src/server.rs:29` (verified for entrypoint, clap, web_viewer) | Make this a documented product decision, or default to `127.0.0.1` with an opt-in. |
| 4 | **No per-message size limit on gRPC.** `max_decoding_message_size = u32::MAX` (4 GiB) on servers and clients, and `.rrd` payloads may be up to 16 GiB. Combined with #3, anyone on the network can exhaust memory. | `crates/data_flow/re_grpc_server/src/lib.rs:45-46,437`, `re_grpc_client/src/lib.rs:15`, `re_server/src/entrypoint.rs:156`, `re_protos/src/v1alpha1/rerun.common.v1alpha1.ext.rs:1314` | Set sane limits for network listeners. |
| 5 | **OAuth tokens are stored in plaintext with default file permissions.** | `crates/utils/re_auth/src/oauth/storage.rs:82-88`: `store()` calls plain `std::fs::write` with no permission handling (verified) | Write with mode 0600, or use the OS keychain. |

### P1

| # | Issue | Evidence |
| --- | --- | --- |
| 6 | **Flaky-test retries are not active in CI.** The nextest `ci` profile (`retries = 1`) says "Set via `--profile ci` in CI scripts", but nothing passes it. CI also installs nextest 0.9.89 while pixi requires ≥0.9.140. | `.config/nextest.toml` (verified), `scripts/ci/rust_checks.py:387-402`, `.github/actions/setup-rust/action.yml` |
| 7 | **The nightly pipeline was removed from this repo, but the code still assumes it.** `nightly.yml` was deleted in `164e709e0d` (2026-10-21). The nightly-only checks (`docs_slow`, `individual_crates`, the macOS/Windows check groups, the full C++ matrix, full link check) and `channel = "nightly"` examples have no trigger here. `CHANNEL == 'nightly'` branches remain in `reusable_checks*.yml`, `reusable_build_web.yml` and `reusable_upload_js.yml`. | `git show 164e709e0d` (verified) |
| 8 | **External importer plugins run on every unsupported file.** Any `rerun-importer-*` executable on `$PATH` runs for each opened file with an unsupported extension, including drag-and-drop. | `crates/data_flow/re_importer/src/importer_external.rs:16-180` |
| 9 | **Known-exercised vulnerable dependencies.** Two quick-xml DoS advisories are reachable via user-supplied DAE/URDF files, and one zstd-safe advisory is reachable via MCAP. All three are ignored in `deny.toml`. 23 RUSTSEC IDs are ignored in total, mostly with "No ticket yet". | `deny.toml` advisories section |
| 10 | **No fuzzing of the untrusted-input parsers** (RRD, MCAP/CDR, URDF/DAE, MP4, LeRobot). | Verified for this repo: no `fuzz/` directory, and no `libfuzzer-sys`, `cargo-fuzz` or `afl` dependency (`git ls-files`, `Cargo.toml` grep). Whether the private monorepo has one is unknown. |
| 11 | **A panic in `rerun_c` aborts the host C++ process.** There is no `catch_unwind` (it is also clippy-banned) and release builds use `panic="abort"`. `rerun.h` is kept in sync by hand. | `crates/top/rerun_c/src/lib.rs`, `Cargo.toml:761`, `rerun_cpp/src/rerun/c/rerun.h:485` |
| 12 | **Query-cache correctness TODOs.** Stale data can survive if an invalidation lands before the data is cached (#5974), and a partial latest-at result discards the whole query (RR-3762). | `crates/store_app/re_query/src/latest_at.rs:740,794-802` |
| 13 | **OSS server scalability.** `LazyStore` has no chunk cache (RR-4503). `TaskRegistry` grows without bound. Positional reads block the tokio reactor. Catalog metadata is in memory only. | `crates/store/re_chunk_store/src/lazy_store.rs`, `crates/store_app/re_server/src/store/{task_registry.rs, resolved_store.rs:119}` |
| 14 | **Encoder may write no end-of-stream marker.** With footers disabled, `Encoder::finish` returns early without an End marker, despite its doc. | `crates/store/re_log_encoding/src/rrd/encoder.rs:365-384` (UNVERIFIED whether intended) |
| 15 | **The crash signal handler allocates and does a blocking HTTP flush inside signal context.** The code itself acknowledges the deadlock risk. | `crates/utils/re_crash_handler/src/lib.rs:93-157` (comment at `:142`) |
| 16 | **Privacy: agent-panel analytics.** `agent_turn` analytics carries the redacted full prompt and response text, and `feedback` can carry the whole conversation. Documented as opt-in. | `crates/utils/re_analytics/src/event.rs:605-735` |
| 17 | **The web viewer does not persist blueprints** (TODO #2579), and loading into an already-active blueprint only warns (TODO #5514). | `crates/top/re_viewer/src/app/mod.rs:1811`, `app/logic.rs:~318` |
| 18 | **Backpressure gaps.** The gRPC client queue is bounded by message count, not bytes (RR-3869). The server's internal channels have fixed sizes (1024 messages, 128 MiB). | `crates/data_flow/re_grpc_client/src/write.rs:134`, `re_grpc_server/src/lib.rs:49-53` |

### P2

| # | Issue | Evidence |
| --- | --- | --- |
| 19 | **Doc drift: `AGENTS.md`/`CLAUDE.md`** list `crates/{build,store,top,viewer}`. `crates/viewer` no longer exists. | `AGENTS.md:114-121` (verified) |
| 20 | **Doc drift: a missing pixi task.** `BUILD.md:152` and `examples/python/README.md:67` say `pixi run py-build-examples`; the real task is `py-sync-examples`. | `pixi.toml:516` (verified) |
| 21 | **Doc drift: the chunk size comment.** The `ChunkStoreConfig::chunk_max_bytes` doc says 8 MiB; the default is 384 KiB. | `crates/store/re_chunk_store/src/store.rs:57` vs `:150` (verified) |
| 22 | **Doc drift: the pipeline design doc.** `re_datafusion/PIPELINE_V2.md` says "proposal, not yet implemented"; v2 is the default. | `crates/store_app/re_datafusion/PIPELINE_V2.md:3`, `src/pipeline/config.rs:19-27` (verified) |
| 23 | **Doc drift: a stale comment on perf telemetry.** It claims telemetry is "not compiled into final user builds", but the `rerun-cli` `base` feature includes it. | `crates/top/rerun/src/commands/entrypoint.rs:875` (verified), `crates/top/rerun-cli/Cargo.toml:47,104` |
| 24 | **Doc drift: `ARCHITECTURE.md`** mentions the `instant` crate (the code uses `web-time`), says query caching is "future" (it exists), and omits the C/C++ SDKs. The workflows README names `manual_dispatch.yml` and `on_main.yml`, which don't exist. | `ARCHITECTURE.md`; `.github/workflows/README.md` |
| 25 | **Dead code.** `re_test_mocks::posthog::MockPostHog` and `assert_sink_empty!` (about 600 LOC) are unused. | `crates/tests/re_test_mocks/src/posthog.rs` (verified, no users) |
| 26 | **Unused CI and config files:** `.github/runs-on.yml`, `.github/actions/vercel/`, `reusable_bench.yml` (no caller), and the dead `target.osx-arm64.activation.env` table in `.cargo/config.toml`. | `.github/`, `.cargo/config.toml` |
| 27 | **Scripts with no callers:** `scripts/ci/{pixi_install_wheel.py, repack_wheel.py, install_cargo_insta.sh, mdlint_test.py}`, `scripts/run_all.py`, `scripts/screenshot_compare/`. Also, `check-test-binaries` is not run in CI. | `git grep` |
| 28 | **Config nits:** `.typos.toml` has misspelled corrections (`fufill`, `habor`); `lychee.toml` sets `insecure = true`; mypy is pinned to 1.14.1 in pixi but 1.19.1 in uv; the root `pyproject.toml` version is 0.28 vs 0.43; the `re_protos_builder` config mentions a `manifest_registry` package that no longer exists; `bacon.toml` binds a key to a missing `initial` job. | the respective files |
| 29 | **Avoidable `unsafe`.** A no-op waker is built by casting `&RawWaker` to `&Waker`; `Waker::noop()` exists. | `crates/store_app/re_dataframe/src/query.rs:3660-3710` |
| 30 | **Stale constant.** A stale string path `crates/viewer/re_viewer/...` is left in a fixture. | `crates/utils/re_agent/src/session.rs:2007` (verified) |
| 31 | **Duplicated constants.** Port 9876 is defined in three places, port 9090 is hard-coded in the CLI, and `EXTERNAL_IMPORTER_PREFIX` plus exit code 66 are duplicated. | `re_uri/src/lib.rs:76`, `re_grpc_server/src/lib.rs:43`, `entrypoint.rs:84,323`, `crates/top/rerun/src/lib.rs:165-182` |
| 32 | **Two lens runtimes.** There are duplicate `DEFAULT_RUNTIME`s in `re_lenses_core` and `re_lenses`. | `re_lenses_core/src/selector/runtime.rs:18`, `re_lenses/src/runtime.rs:12` |

## 2. Tech-debt hotspots

**TODO/FIXME/HACK counts by folder** (1 763 lines total; the convention is `TODO(name|#issue|RR-…)`):

| Folder | Count |
| --- | --- |
| crates/store | 252 |
| crates/viewer_support | 199 |
| crates/ui_tools | 192 |
| rerun_py | 148 |
| crates/views | 129 |
| crates/top | 110 |
| crates/store_app | 106 |
| crates/build | 88 |
| rerun_cpp | 79 |
| crates/data_flow | 78 |

**By crate:**

| Crate | TODOs |
| --- | --- |
| `re_renderer` | 110 |
| `re_viewer_context` | 84 |
| `re_view_spatial` | 82 |
| `re_viewer` | 47 |
| `re_protos` | 42 |
| `rerun` | 42 |
| `re_chunk_store` | 35 |

**Top files:**

| File | TODOs |
| --- | --- |
| `crates/store_app/re_dataframe/src/query.rs` | 16 |
| `crates/top/rerun/src/commands/rrd/split.rs` | 12 |
| `crates/store/re_protos/proto/rerun/v1alpha1/cloud.proto` | 11 |
| `crates/store/re_chunk_store/src/dataframe.rs` | 10 |

**God files** (large, frequently changed, hard to review):

| File | LOC |
| --- | --- |
| `re_dataframe/src/query.rs` | 5 521 |
| `re_sdk/src/recording_stream.rs` | 3 646 |
| `re_viewer/src/app/viewer_control/mod.rs` | 3 137 |
| `rerun_py/src/python_bridge.rs` | 2 891 |
| `re_view_time_series/src/view_class.rs` | 2 477 |
| `rerun/src/commands/entrypoint.rs` | 2 454 |
| `re_selection_panel/src/selection_panel.rs` | 2 345 |
| `re_viewer_context/src/cache/video_stream_cache/mod.rs` | 2 317 |
| `re_datafusion/src/datafusion_table_widget.rs` | 2 275 |
| `re_grpc_server/src/lib.rs` | 2 166 |
| `re_viewer/src/app/command_handling.rs` | 2 011 |

**`unsafe`.**
There are 161 `unsafe {}` blocks.
49 of them are in generated `re_sdk_types` code (`get_unchecked`) and 27 in the `rerun_c` FFI.
The others are mostly in audio and memory code, which is legitimate.

**`unwrap`/`expect`.**
There are about 4 536 / 2 334 calls, roughly one third of them in test paths.
`clippy::unwrap_used` warns, and 126 explicit opt-outs exist.

**Duplicated logic and approaches:**
- Two DataFusion pipelines (v1 and v2) coexist.
- `QueryDataset` (deprecated) is kept alive through `ext.legacy_adapter.rs`.
- Two reqwest major versions are linked (`Cargo.toml:458,531`).

**Missing tests in critical paths:** see `05-DEV-GUIDE.md` §3: fork handling, the C ABI, fuzzing, and web persistence.

## 3. Performance and scalability concerns visible in code

| Concern | Evidence |
| --- | --- |
| Too many entities slow the viewer down; multi-million-point clouds are slow | Acknowledged in `README.md` "Status" (issues #7115 and #1136) |
| Immediate mode means every frame re-queries and re-lays out everything | `ARCHITECTURE.md` "Immediate mode" |
| Ingest gets at most 10 ms per frame, so heavy streams lag behind the UI | `crates/top/re_viewer/src/app/logic.rs:261` |
| Latest-at chunk lookup walks back by `max_interval_length`, which is O(n) in degenerate cases (one very long chunk makes every lookup scan); the query cache amortizes it | `crates/store/re_chunk_store/src/query.rs:1080-1127`, `store.rs:285-287` |
| Memory purging is heuristic: it frees `fraction + 0.2` of the store when over the limit | `crates/top/re_viewer/src/app/logic.rs:828-894` |
| Nested `RwLock`s in the query cache, plus a global subscriber `RwLock` | `crates/store_app/re_query/src/cache.rs:136`, `crates/store/re_chunk_store/src/subscribers.rs:86` |
| Process-wide string interner that is never freed; `FORGIVING_PARSE_CACHE` may be unbounded (UNVERIFIED) | `crates/utils/re_string_interner/src/lib.rs:588`, `crates/store/re_log_types/src/path/parse_path.rs:11` |
| `re_sdk_types` (about 101 k LOC, generated) dominates compile time | `crates/store/re_sdk_types` |
| PyPI project size cap of 50 GiB, guarded by `pypi-size-check` | `RELEASES.md`, `scripts/ci/pypi_cleanup.py` |

## 4. Easy-to-break invariants and tribal knowledge

1. **Never edit generated files.** That means `re_sdk_types`, the Python and C++ archetypes, `docs/content/reference/types`, `cli.md`, `re_protos/src/v1alpha1/rerun.*.v1alpha1.rs` and `re_workspace_hack/Cargo.toml`. Edit `*.def.rs` or `.proto`, then regenerate. CI catches stale output.
2. **A (RowId × component) cell is immutable.** Never change chunk data while keeping its ChunkId or RowIds, because caches depend on it (`re_chunk/src/chunk.rs:274-279`).
3. **Chunks must be RowId-sorted before insert** (`re_chunk_store/src/writes.rs:159`). A ChunkId must be unique per recording, and a duplicate is silently ignored.
4. **Static means "no time columns".** `TimeInt::STATIC` must never appear inside a time column.
5. **Static data always wins** over temporal data. Static `Transform3D`/`Pinhole` chunks are deliberately never dropped (RR-4887).
6. **`StoreId` is not unique.** Logging twice with the same IDs merges the data into one recording.
7. **The active recording comes from the route, not from `StoreHub`** (`re_viewer_context/src/store_hub/mod.rs:106-110`).
8. **Don't mutate stores mid-frame** in the viewer. Queue a `SystemCommand` or `ViewportCommand`.
9. **Chunk-store subscribers are global** and see every store. Filter by `StoreId` in your subscriber.
10. **Visualizers run in parallel** with rayon, so they must be `Send + Sync`.
11. **wasm has no threads, filesystem, env or `std::time`.** Gate code with `cfg(target_arch = "wasm32")`, and check it with `bacon wasm` or `rs-check` group `wasm`.
12. **The `wasm-bindgen` version is pinned in three places** (workspace `Cargo.toml`, `re_viewer`, `re_redap_browser`) plus pixi's `wasm-bindgen-cli`, and they must match. `build-wasm.mjs` patches the generated JS.
13. **Adding or removing a crate** requires the right layer folder, then `pixi run crate-graph` and `pixi run hakari-generate`.
14. **Changing a type definition** changes Rust, Python and C++ at once. Update the snippets for all three languages; `compare_snippet_output.py` checks them.
15. **`.rrd` compatibility** is N-1 only. Any change to the persisted layout needs a Sorbet migration (bump `METADATA_VERSION`) and, ideally, a back-compat asset in `tests/assets/rrd`.
16. **`rerun_bindings.pyi` is hand-maintained.** Keep it in step with `python_bridge.rs`; `pixi run py-check-signatures` checks this.
17. **Image snapshot tests** need git-lfs, and they are strict only on Linux CI (lavapipe).
18. **The public repo is a mirror.** Fixes may need to land through the private monorepo; ask before assuming a GitHub PR is the path.
19. **egui, egui_plot, egui_tiles and similar crates are maintained by the team.** Fix bugs upstream rather than working around them locally (`AGENTS.md`).

## 5. Open questions for the team

1. Is defaulting to `--bind 0.0.0.0` with an unauthenticated proxy a deliberate product decision? Is there a threat model doc?
2. Is the `re_sdk/src/global.rs:94` fork bug known? Is the missing `on_release` on the `select!` path intentional?
3. Where do nightly builds, the changelog-label check and branch protection live now that `nightly.yml` is gone? Should the `CHANNEL == 'nightly'` branches and the `channel = "nightly"` examples be removed?
4. Should CI use the nextest `ci` profile (retries)? Why is nextest pinned to 0.9.89 in CI?
5. Does `re_server` enforce authentication by default, and how is it meant to be deployed (local dev only, or as a shared server)?
6. Is `Encoder::finish` skipping the End marker when footers are disabled intended?
7. What is the plan for retiring `LogMsg` (TODO #8631), `ApplicationId` (→ `EntryName`) and the v1 DataFusion pipeline?
8. Are `.github/runs-on.yml`, `.github/actions/vercel/` and `reusable_bench.yml` used from the private monorepo, or are they dead?
9. Is there a fuzzing setup elsewhere (for example in the private repo) for the RRD, MCAP and URDF parsers?
10. What is the support status of the agent panel (experimental, default off)? Is the analytics content of `agent_turn` approved by privacy review?
11. Is `MockPostHog` waiting for a consumer, or can it be deleted?
12. Should `credentials.json` move to the OS keychain?
13. Is a separate viewer-side blueprint migrator planned, or are Sorbet migrations considered enough for blueprints?
14. What is the policy for the 23 ignored RUSTSEC advisories, given that most say "No ticket yet"?
