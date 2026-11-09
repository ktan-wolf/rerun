# 07 — Learning path

This is an ordered study plan for an engineer who knows Rust or Python but not Rerun.
Effort estimates assume focused reading; adjust them to your background.
Steps 1–6 are needed by everyone. After that, pick the track that matches your work.

## Step 0 — Use the product (½ day)

- `pip install rerun-sdk`, then run the `README.md` "short taste" snippet.
- Open an example `.rrd` with `rerun examples/assets/example.rrd`; the file is in git-LFS.
- Read the user docs in this order:
  1. `docs/content/overview/` ("what is Rerun")
  2. `docs/content/concepts/` (entity paths, timelines, static data, blueprints)
  3. `docs/content/getting-started/`
- Click around the viewer: the blueprint tree, selection panel, time panel, and the 2D, 3D and time-series views.

## Step 1 — Set up and build (½–1 day, mostly compile time)

- Follow `05-DEV-GUIDE.md` §1: rustup, pixi, git-lfs, `pixi run check-env`, `pixi run rerun`, `pixi run py-build`.
- Read `AGENTS.md` (repo rules), `BUILD.md` and `CODE_STYLE.md`.
- Skim the task list in `pixi.toml`.

## Step 2 — Prerequisite technologies (2–4 days, depending on background)

Learn them in this order:

| # | Technology | Why | What to learn | Effort |
| --- | --- | --- | --- | --- |
| 1 | **Apache Arrow** (Rust `arrow` crate) | It is the data format everywhere | Arrays, `ListArray`, `RecordBatch`, `Schema` and field metadata, IPC | 1 day |
| 2 | **Immediate-mode GUI: egui / eframe** | The whole viewer is egui | The egui demo app; why immediate mode; `Ui`, `Response`, `Context` | ½–1 day |
| 3 | **Rust concurrency used here** | Batcher threads, rayon views, locks | `parking_lot`, crossbeam channels, `rayon` `par_iter`, tokio basics | ½ day |
| 4 | **gRPC / protobuf with tonic and prost** | SDK streaming and the catalog API | Services, streaming RPCs, interceptors and layers | ½ day |
| 5 | **wgpu basics** (only if you work on rendering) | `re_renderer` | Device, queue, render pipelines, bind groups, WGSL | 1–2 days |
| 6 | **DataFusion** (only if you work on catalog or query) | `re_datafusion`, Python `catalog` | `TableProvider`, `ExecutionPlan`, filter pushdown | 1 day |
| 7 | **PyO3 + maturin** (only for Python SDK work) | `rerun_py` | `#[pyfunction]`, GIL release, Arrow C-FFI | ½ day |
| 8 | **WebAssembly with wasm-bindgen** (only for web work) | Web viewer | wasm limitations (`ARCHITECTURE.md` "Wasm") | ½ day |

## Step 3 — Architecture (1 day)

1. `ARCHITECTURE.md` and `crates/README.md`: the layer rules.
2. This guide, in order: `00-OVERVIEW.md`, `02-ARCHITECTURE.md`, `04-DATA-AND-FLOWS.md` §1 and §3.
3. Run `pixi run crate-graph` (or `git lfs pull` and open `crate_graph.svg`).

## Step 4 — The data model in code (1–2 days)

Read these files in order:

1. `crates/store/re_log_types/src/path/entity_path.rs` (around `:94`), `src/index/{time_int.rs, time_point.rs, timeline.rs}`, `src/store_id.rs`.
2. `crates/store/re_types_core/src/{component_descriptor.rs, row_id.rs, chunk_id.rs, archetype.rs}`.
3. One type definition and its generated output:
   - `crates/build/re_type_definitions/rerun/archetypes/points3d.def.rs`;
   - `crates/store/re_sdk_types/src/archetypes/points3d.rs` (generated) and `points3d_ext.rs`;
   - `rerun_py/rerun_sdk/rerun/archetypes/points3d.py`.
4. `crates/store/re_chunk/src/chunk.rs` (struct at `:242`, `sanity_check` at `:1822`) and `src/latest_at.rs`.
5. `crates/store/re_sorbet/src/lib.rs` and `src/migrations/mod.rs`.
6. `crates/store/re_log_msg/src/log_msg.rs`.

The repo also ships `skills/rerun-data-model/SKILL.md`, a condensed version of this material.

## Step 5 — Storage and query (1–2 days)

1. `crates/store/re_chunk_store/src/lib.rs` (doc comment) and `src/store.rs` (indexes at `:527-683`).
2. `src/writes.rs` (`insert_chunk` at `:53`, `insert_chunk_impl` at `:113`), `src/query.rs` (`:906-1127`), `src/gc.rs`.
3. `crates/store_app/re_query/src/{cache.rs, latest_at.rs, storage_engine.rs}`.
4. `crates/store_app/re_entity_db/src/entity_db.rs` (`add_log_msg` at `:847`, `latest_at` at `:607`).
5. `crates/store/re_log_encoding/src/rrd/frames.rs`: the `.rrd` format.
6. Exercise: trace `04-DATA-AND-FLOWS.md` Flow A, steps 16–18, with a debugger, or by adding `re_log::info!` calls on a local branch.

## Step 6 — The SDK path (1 day)

1. `crates/top/re_sdk/src/recording_stream.rs`: the builder at `:175`, `RecordingStreamInner::new` at `:989`, `forwarding_thread` at `:1644`, `record_row` at `:1858`.
2. `crates/store/re_chunk/src/batcher.rs` and `crates/top/re_sdk/src/log_sink.rs`.
3. `crates/data_flow/re_grpc_client/src/write.rs` and `crates/data_flow/re_grpc_server/src/lib.rs` (MessageProxy).
4. Python side: `rerun_py/rerun_sdk/rerun/__init__.py` (`init`), `_log.py`, `sinks.py`, then `rerun_py/src/python_bridge.rs`. Also read `rerun_py/ARCHITECTURE.md`.
5. Exercise: run `examples/python/minimal` with `RUST_LOG=debug`, and watch the batcher flushes and the gRPC connection.

## Track V — Viewer (3–5 days)

1. `crates/top/re_viewer/src/app/logic.rs` (`logic_impl` at `:27`, `receive_messages` at `:154`) and `app/mod.rs` (`frame_ui` at `:1375`).
2. `crates/viewer_support/re_viewer_context/src/store_hub/mod.rs`, then `viewer_context.rs`.
3. `app_state.rs` (`show` at `:303`) → `crates/panels/re_viewport/src/viewport_ui.rs` → `crates/viewer_support/re_view/src/system_execution.rs`.
4. The view contracts: `re_viewer_context/src/view/{view_class.rs, visualizer_system.rs, view_context_system.rs}`.
5. A small view, `crates/views/re_view_bar_chart/src/view_class.rs`, then a real one, `crates/views/re_view_spatial/src/visualizers/points3d.rs` and `view_3d.rs`.
6. Blueprints: `crates/viewer_support/re_viewport_blueprint/src/viewport_blueprint.rs`, `re_viewer_context/src/blueprint_helpers.rs`.
7. Rendering: `crates/ui_tools/re_renderer/README.md`, `src/context.rs`, `src/view_builder.rs`, `src/renderer/point_cloud.rs`, `shader/point_cloud.wgsl`.
8. Testing: `crates/tests/re_test_context/src/lib.rs`, then any `tests/integration/*.rs` in a view crate.
9. Exercise: add a snapshot test to `re_view_bar_chart`, or follow `examples/rust/custom_view`.

## Track D — Data in/out, catalog and server (3–4 days)

1. `crates/data_flow/re_data_source/src/data_source.rs`, `crates/data_flow/re_importer/src/{lib.rs, import_file.rs, importer_external.rs}`.
2. `crates/data_flow/re_mcap/src/lib.rs` and the decoders; `skills/rerun-mcap/SKILL.md`.
3. `crates/store/re_protos/proto/rerun/v1alpha1/cloud.proto`: skim every RPC.
4. `crates/store_app/re_server/README.md`, `src/rerun_cloud/mod.rs`, `src/store/{in_memory_store.rs, resolved_store.rs}`.
5. `crates/data_flow/re_redap_client/src/{connection_registry.rs, grpc.rs}`, `crates/utils/re_auth`.
6. `crates/store_app/re_datafusion/ARCHITECTURE.md`, then `src/dataframe_query_common/mod.rs` and `src/pipeline/`.
7. Exercise: run `rerun server -d test=<file.rrd>`, then query it from Python with `rerun.catalog` (see `examples/python/dataframe_query`, `skills/rerun-catalog-queries`).

## Track C — Codegen and multi-language SDKs (1–2 days)

1. `crates/build/re_types_builder/src/bin/build_re_types.rs` and `src/lib.rs` (hashing at `:246-273`).
2. Add a field to a testing archetype in `crates/build/re_type_definitions/rerun/testing/`, run `pixi run codegen`, and read the diff in all three languages. Do this on a throwaway branch.
3. The C++ side: `rerun_cpp/README.md`, `rerun_cpp/src/rerun/recording_stream.hpp`, `crates/top/rerun_c/src/lib.rs`.
4. Snippets: `docs/snippets/README.md`, `docs/snippets/snippets.toml`.

## Track R — Build, CI and release (1 day)

1. `.github/workflows/README.md`, `on_pull_request.yml`, `reusable_checks.yml`, `reusable_checks_rust.yml`.
2. `scripts/ci/rust_checks.py`, `scripts/lint.py`, `scripts/check_crate_dependencies.py`.
3. `RELEASES.md`, `release.yml`.
4. `deny.toml`, `clippy.toml`.

## Step 7 — Risks and conventions (½ day)

- Read `06-RISKS-AND-TODO.md` §4 (invariants) and §5 (open questions).
- Read `05-DEV-GUIDE.md` §5 (conventions).
- Pick one P2 item from `06-RISKS-AND-TODO.md` as a first PR: a doc fix or dead-code removal.

**Total:**
- Core path (Steps 0–7): about 8–12 working days.
- Each track: another 1–5 days.

---

## Checklist: "I understand this product"

- [ ] I can explain what an entity path, a timeline, static data, a component, an archetype and a blueprint are, without looking them up.
- [ ] I can draw the path of a `rr.log` call from Python to pixels, naming at least 8 crates in order.
- [ ] I know what a `Chunk` contains, why RowIds sort by time, and why a (RowId × component) cell must never change.
- [ ] I can explain how a latest-at query finds its answer: static first, then the time index with `max_interval_length` backtracking, then the RowId tie-break.
- [ ] I know the layer rules in `crates/README.md` and which folder a new crate belongs in.
- [ ] I know which files are generated, how to regenerate them (`pixi run codegen`, `codegen-protos`, `man`, `crate-graph`, `hakari-generate`), and that CI checks them.
- [ ] I can build and run the native viewer, the Python SDK, and (if relevant) the web viewer and C++ SDK.
- [ ] I can run one Rust test, one Python test, and update an image snapshot.
- [ ] I know the `.rrd` compatibility policy (N-1) and how to add a Sorbet migration.
- [ ] I know the three gRPC services, which ports they use by default, and which ones are authenticated.
- [ ] I can find where a view class is registered and add a visualizer to it.
- [ ] I know what must not happen mid-frame in the viewer, and how to queue a blueprint edit instead.
- [ ] I know which constraints wasm imposes and how to check wasm builds (`bacon wasm`, `rs-check` group `wasm`).
- [ ] I know where analytics, auth credentials and viewer settings are stored on disk, and which env vars control them.
- [ ] I know that the public repo is a mirror of a private monorepo, and what that means for PRs and releases.
- [ ] I have read the P0 items in `06-RISKS-AND-TODO.md` and know who to ask about them.
