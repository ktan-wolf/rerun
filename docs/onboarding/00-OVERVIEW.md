# 00 — Overview

Snapshot audited: `main` @ `bb934c2c80` (2026-11-05), workspace version `0.43.0-alpha.1+dev` (`Cargo.toml:40`).

## What the product is

Rerun is "the data layer for physical AI" (`README.md`): SDKs (Python, Rust, C++, plus a JS web-viewer wrapper) that log multimodal, time-indexed data (images, point clouds, tensors, transforms, time series, video, …) as Apache Arrow chunks, and a Viewer (native and WebAssembly) that visualizes that data in sync on one or more timelines.
Around that core sit a query layer (dataframes / DataFusion SQL), file importers (MCAP, LeRobot, MP4, Parquet, HDF5, URDF, …), an OSS local catalog server (`re_server`) speaking the "Rerun Data Protocol" (redap) over gRPC, and a client for the commercial hosted catalog ("Hub"/cloud).

## Repo type

**Hybrid polyglot monorepo** built around one Cargo workspace (`Cargo.toml` `[workspace] members`), with language bindings and packaging that live next to it:

| Area | What | Evidence |
| --- | --- | --- |
| `crates/` | ~123 Rust crates in 10 layer folders (layering enforced by `scripts/check_crate_dependencies.py`) | `crates/README.md`, `ARCHITECTURE.md` |
| `rerun_py/` | Python SDK `rerun-sdk` (pyo3 + maturin), also a workspace member | `rerun_py/pyproject.toml`, `Cargo.toml` members |
| `rerun_cpp/` | C++ SDK (CMake) wrapping the C SDK `crates/top/rerun_c` | `rerun_cpp/CMakeLists.txt`, `CMakeLists.txt` |
| `rerun_js/` | npm packages `web-viewer` and `web-viewer-react` wrapping the wasm viewer | `rerun_js/package.json` |
| `rerun_notebook/` | Jupyter/anywidget notebook widget (Python + TS) | `rerun_notebook/pyproject.toml` |
| `examples/` | 93 Python, 32 Rust, 18 C++, plus C/JS/notebook examples | `examples/manifest.toml` |
| `docs/` | Docs site content and multi-language code snippets | `docs/README.md` |
| `tests/` | Cross-language integration/stress tests and shared assets | `tests/README.md` |

Size (tracked files, `git ls-files`): 9 343 files; Rust ≈ 793 k lines in 3 209 files, Python ≈ 164 k lines, C++ ≈ 79 k lines (`.cpp`+`.hpp`), 42 WGSL shaders, 8 `.proto` files.

## Top-level directory map

| Path | Purpose |
| --- | --- |
| `crates/` | All Rust crates, one folder per layer (see below). |
| `rerun_py/` | Python SDK: Rust bindings in `rerun_py/src/`, Python package in `rerun_py/rerun_sdk/rerun/`, docs (`mkdocs.yml`), tests. |
| `rerun_cpp/` | C++ SDK sources, Arrow C++ build helpers, tests. |
| `rerun_js/` | JS/TS web-viewer npm packages. |
| `rerun_notebook/` | Notebook widget package. |
| `rerun_pixi_env/` | Small Python package used by the pixi environment (`rerun_pixi_env/pyproject.toml`). |
| `examples/` | Example programs per language, indexed by `examples/manifest.toml`. |
| `docs/` | `docs/content/` (rerun.io/docs source), `docs/snippets/` (runnable snippets in Py/Rust/C++, configured by `snippets.toml`), `docs/speculative_links/`. |
| `tests/` | `tests/python`, `tests/rust`, `tests/cpp`, `tests/js` stress/integration tests; `tests/assets` test data. |
| `scripts/` | Dev, lint, CI, release, snapshot and docs scripts (Python/shell). |
| `skills/` | Agent skills shipped to users (`npx skills add rerun-io/rerun`, see `README.md`). |
| `.github/` | Workflows (≈50), composite actions (`setup-rust`, `vercel`), PR/issue templates. |
| `ci_docker/` | Docker image used by CI (`Dockerfile`, `publish.sh`). |
| `hooks/` | Optional git `pre-push` hook. |
| `.cargo/`, `.config/` | Cargo config (`config.toml`, `cranelift.toml`), nextest and cargo-hakari config. |
| `.vscode/`, `.idea/`, `.run/` | Editor settings and run configs. |
| Root files | `Cargo.toml`/`Cargo.lock` (Rust workspace), `pixi.toml`/`pixi.lock` (task runner and toolchain env), `pyproject.toml`/`uv.lock` (uv Python workspace), `CMakeLists.txt` (C++ build), `rust-toolchain`, lint configs (`clippy.toml`, `deny.toml`, `.typos.toml`, `lychee.toml`, `taplo.toml`, `.oxlintrc.json`, `.prettierrc.toml`, `.clang-format`), `about.toml` (license bundling), `bacon.toml`, `kitdiff.toml`, `.mcp.json`, `.RERUN_ROOT` (empty marker file). |

### `crates/` layers (lowest may not depend on higher; `crates/README.md`)

```
crates/
├── tests/          test-only support crates (dev-dependencies only)
├── top/            rerun, rerun-cli, re_sdk, re_viewer, rerun_c, re_web_viewer_server, re_viewer_mcp
├── views/          re_view_* (12 view types)            ┐ siblings
├── panels/         blueprint tree, selection, time, …   ┘
├── viewer_support/ re_viewer_context, re_view, component UIs, blueprint model
├── store_app/      re_entity_db, re_query, re_dataframe, re_datafusion, re_server, re_tf, re_catalog_client
├── data_flow/      gRPC client/server, redap client, importers (mcap, lerobot, hdf5, mp4, parquet), lenses
├── store/          re_chunk, re_chunk_store, re_log_types, re_sdk_types, re_protos, re_log_encoding, …
├── ui_tools/       re_renderer (wgpu), re_ui (egui theme), re_plot, re_agent_ui, … (sibling of store stack)
├── build/          codegen: re_type_definitions, re_types_builder, re_protos_builder
└── utils/          small leaf crates: re_log, re_tracing, re_memory, re_auth, re_analytics, re_video, …
```

## Git history (Phase 1.4)

Commands used: `git log`, `git shortlog`; full, non-shallow history (`git rev-parse --is-shallow-repository` → `false`).

- 10 390 commits since the initial commit on 2022-04-08.
- Cadence is high and rising: 160–270 commits/month in 2025, 357 in 2026-01, **426 in 2026-10**.
- **Most active areas, last 12 months** (file-touches): generated SDK types (`re_sdk_types`, `rerun_py`, `rerun_cpp` — a single type-definition change regenerates code in three languages), snapshot tests, docs, viewer UI (`re_viewer`, `re_component_ui`, `re_view_spatial`, `re_viewer_context`, `re_dataframe_ui`), redap/cloud (`re_protos`, `re_redap_client`, `re_server`, `re_datafusion`, `re_redap_tests`).
- **Most-changed individual files** (excluding locks, snapshots, changelog): `Cargo.toml` (229), `rerun_py/rerun_bindings/rerun_bindings.pyi` (148), `re_viewer/src/app.rs` (148), `pixi.toml` (111), `re_view_time_series/src/view_class.rs` (98), `rerun/src/commands/entrypoint.rs` (95), `re_sdk_types/src/reflection/mod.rs` (89), `re_test_context/src/lib.rs` (87), `re_viewer/src/app_state.rs` (84), `rerun_py/rerun_sdk/rerun/catalog/_entry.py` (79), `re_redap_client/src/grpc.rs` (76).
- **Recent direction (Aug–Nov 2026, from commit subjects):** an in-viewer **coding-agent panel** and viewer MCP server (~100 commits mention "agent"), **blueprint** editing/migrations, **catalog / Hub** features (multipart S3 uploads, dataset policies, "tiered indexing" in the metastore), **LeRobot / MCAP / lenses** ingestion, video, and a large **crate-folder reorganization** (`bc69c22128` 2026-09-14 "Give the views their own crate folder" through `2fdee20853` 2026-11-02 "Move re_renderer … into crates/ui_tools").
- **Main contributors per area (last 6 months, `git shortlog -sn -- <dir>`):** Emil Ernerfeldt leads every area. Next most active: store — Isse, Antoine Beyeler, Jochen Görtler; store_app — Antoine Beyeler, Jochen Görtler, Andrea Reale; data_flow — Jochen Görtler, Antoine Beyeler; views — Andreas Reich, Isse; panels — Isse, Michael Grupp; viewer_support — Isse, Andreas Reich, Lucas Meurer; rerun_py — Antoine Beyeler, Jochen Görtler; rerun_cpp — Andreas Reich, Isse, Gábor Gyebnár; CI/scripts — Jochen Görtler, Jan Procházka, Andreas Reich.
- **Long-untouched code.** No crate is dormant: every crate had a Rust source change in 2026. Least recently changed crate sources (last `src/*.rs` modification, following moves): `re_capabilities` (2026-02-12), `re_test_mocks` (2026-06-24), `re_memory` (2026-07-27), `re_mutex` (2026-08-03), `re_types_builder_macros` (2026-08-12). Least recently touched examples/tests: `examples/python/shared_recording` and `examples/python/stdio` (2024-04-22), `examples/cpp/shared_recording`, `examples/cpp/stdio` (2025-02-04), `tests/python/gil_stress` (2025-03-10), `examples/cpp/spawn_viewer` (2025-04-01).
- Caveat: because of the reorg, `git log -- crates/<layer>/<crate>` without `--follow` only sees history since the move. Use `git log --follow -- <file>` for per-file history.

## Docs that are outdated or contradict the code (Phase 1.3)

| Doc | Claim | Reality |
| --- | --- | --- |
| `AGENTS.md:114-121` (`CLAUDE.md` is a symlink to it) | `crates/` has `build/ store/ top/ viewer/` | There is no `crates/viewer/`; there are 10 layer folders (`ls crates`, `crates/README.md`). |
| `crates/utils/re_agent/src/session.rs:2007` | Builds a path `crates/viewer/re_viewer/src/app/mod.rs` | `re_viewer` now lives in `crates/top/re_viewer` (likely a test fixture string; still stale). |
| `ARCHITECTURE.md` "Logging APIs" | "Python SDK or our Rust SDK" | C++ SDK (`rerun_cpp`) and C SDK (`rerun_c`) also exist. |
| `ARCHITECTURE.md` "Wasm" | "use `instant` crate" for time | The workspace uses `web-time = "1.1"` (`Cargo.toml:574`); `instant` is not a workspace dependency. |
| `ARCHITECTURE.md` "Immediate mode" | "In the future we plan on caching queries" | Query caching already exists (`crates/store_app/re_query/src/cache.rs`). |
| `crate_graph.svg` | Crate diagram | Stored in git-LFS; without `git lfs pull` it is a 131-byte pointer file. |
| Root `pyproject.toml:14` | uv workspace version `0.28.0-alpha.1+dev` | Product version is `0.43.0-alpha.1+dev` (`Cargo.toml:40`); the root project is a dev-only uv workspace and its version is not maintained. |

| `BUILD.md:152`, `examples/python/README.md:67` | `pixi run py-build-examples` | That task does not exist; use `py-sync-examples` (`pixi.toml:516`). |
| `crates/store/re_chunk_store/src/store.rs:57` | The default `chunk_max_bytes` is 8 MiB | The default is 384 KiB (`:150`). |
| `crates/store_app/re_datafusion/PIPELINE_V2.md:3` | "proposal, not yet implemented" | v2 is the default pipeline (`src/pipeline/config.rs:19-27`). |
| `.github/workflows/README.md` | Lists `manual_dispatch.yml` and `on_main.yml` | Neither file exists. |
| `RELEASES.md`, `lychee.toml`, `examples/README.md` | Describe a daily nightly build | `nightly.yml` was deleted in `164e709e0d` (2026-10-21). |

The full list is in `06-RISKS-AND-TODO.md` §1 (P2).

## One-page summary

- **What it is.** A Rust-first stack for logging, storing, querying and visualizing time-indexed multimodal data. Users call SDKs (Python, Rust, C++); data becomes **Arrow "chunks"**, which go over gRPC or into `.rrd` files and land in an in-memory **chunk store** that an **immediate-mode egui/wgpu viewer** queries every frame.
- **Data model.**
  - Entity paths × timelines × components.
  - Components are grouped into archetypes, which are **generated** for three languages from `crates/build/re_type_definitions`.
  - Static data has no time and always wins over temporal data.
  - Blueprints, the viewer layout, are stored as ordinary data in a separate store.
- **Main runtime pieces:**
  - `re_sdk` (`RecordingStream`, batcher, sinks);
  - `re_grpc_server` (message proxy on :9876);
  - `re_entity_db`, `re_chunk_store` and `re_query` (the database);
  - `re_viewer` with `re_viewer_context`, the views and the panels;
  - `re_renderer` (wgpu);
  - `re_data_source` and `re_importer` (files and URLs; MCAP, LeRobot, …);
  - `re_server` and `re_redap_client` (catalog protocol "redap", with DataFusion queries);
  - `re_log_encoding` (the `.rrd` format, with N-1 compatibility via Sorbet migrations).
- **Tooling.**
  - Everything runs through `pixi run <task>`.
  - Rust is 1.98.1, edition 2024.
  - Python goes through uv and maturin; C++ through CMake.
  - Tests use nextest, insta, egui_kittest image snapshots (git-LFS), pytest and Catch2.
  - CI runs on GitHub Actions.
  - The public repo is exported from a private monorepo.
- **Biggest risks** (details in `06-RISKS-AND-TODO.md`):
  - an SDK fork-handling bug;
  - unauthenticated listeners on `0.0.0.0` with no gRPC message-size cap;
  - plaintext credentials;
  - CI retries and nightly checks that are not actually running;
  - parsers of untrusted input that are not fuzzed.

## Reading order

1. **This file**, for orientation.
2. **`07-LEARNING-PATH.md`**: what to study and in what order. Use it as your plan.
3. **`02-ARCHITECTURE.md`**: diagrams, the module classification, coupling, global state, entry points.
4. **`04-DATA-AND-FLOWS.md`**: the data model, the `.rrd` format, the APIs, the traced flows, and every env var.
5. **`03-MODULES.md`**: per-module deep dives. Look things up here as you need them.
6. **`01-STACK.md`**: technology inventory and versions.
7. **`05-DEV-GUIDE.md`**: setup, build, test, release, conventions.
8. **`06-RISKS-AND-TODO.md`**: risks, debt, invariants, open questions.

**Method note.**
This guide came from a read-only audit: file reading, `grep` and `git log`/`git shortlog`.
No builds, tests or network calls were run.
Claims cite `path:line`, and anything that could not be confirmed is marked **UNVERIFIED**.
Line numbers are from `bb934c2c80` and will drift.
