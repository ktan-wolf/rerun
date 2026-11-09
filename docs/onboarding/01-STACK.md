# 01 — Technology stack

Versions are taken from manifests at `bb934c2c80`.
"Declared" means the version in a manifest; "locked" means the version in `Cargo.lock` where it differs.

## 1. Languages and runtimes

| Tech | Version | Declared in | Why it's here / what to know |
| --- | --- | --- | --- |
| Rust | toolchain `1.98.1`, MSRV `1.98`, edition 2024, resolver 2 | `rust-toolchain:13-15`, `Cargo.toml:27,39`, `clippy.toml` | Everything core is Rust: store, viewer, SDK core, servers. The `wasm32-unknown-unknown` target is preinstalled for the web viewer. Bumping the toolchain also means rebuilding the CI docker image (`rust-toolchain:9-12`, `ci_docker/publish.sh`). |
| Python | 3.11 for dev (`.python-version`); the published SDK needs `>=3.11` | `rerun_py/pyproject.toml:29`, root `pyproject.toml:16` (`>=3.11,<3.13`) | The most-used SDK (`rerun-sdk`), plus all dev scripts. |
| C++ | C++17, CMake `3.16...3.27` | `rerun_cpp/CMakeLists.txt:7,30`, `CMakeLists.txt:1` | The C++ SDK wraps the C ABI in `crates/top/rerun_c`. `RERUN_SET_CXX_VERSION` overrides the standard (`CMakeLists.txt:19-20`). |
| C | — | `crates/top/rerun_c`, header `rerun_cpp/src/rerun/c/rerun.h` | A stable ABI between Rust and C++. |
| TypeScript/JS | Node `>=22` (pixi), `24.x` in CI; TypeScript `^6.0.3` (web-viewer) and `^5.4.5` (notebook) | `pixi.toml`, `rerun_js/web-viewer/package.json`, `rerun_notebook/package.json` | The web-viewer npm package and the notebook widget. npm only (`package-lock.json`). |
| WGSL | 41 shaders | `crates/ui_tools/re_renderer/shader/` | GPU shaders for `re_renderer`. They use a custom `#import` preprocessor (`src/file_resolver.rs`). |
| Protobuf | proto3, package `rerun.*.v1alpha1` | `crates/store/re_protos/proto/rerun/v1alpha1/*.proto` | Every wire format: gRPC and the `.rrd` framing. |

## 2. Major runtime libraries (Rust)

All are declared in the root `Cargo.toml` under `[workspace.dependencies]`.

| Library | Version | Used for | What a newcomer must know |
| --- | --- | --- | --- |
| `arrow` (+ `parquet`) | 58.4.1 (`Cargo.toml:269,419`), default features off | The in-memory and wire data format of every chunk | Learn `ArrayRef`, `ListArray`, `RecordBatch`, `Schema` and field metadata. Clippy bans `RecordBatch::try_new` and `arrow::compute::{concat,filter,take}`; use the `re_arrow_util` wrappers instead (`clippy.toml`). |
| `datafusion` | 54.0, locked 54.1.0 (`Cargo.toml:302`) | SQL and dataframe queries over catalogs (`re_datafusion`, Python `catalog`) | Learn `TableProvider`, `ExecutionPlan` and filter pushdown. The Python extra pins `datafusion==54.0.0`, so keep the two in sync. |
| `lance` | 10.0 (`Cargo.toml:356`) | Lance tables in `re_server`; optional feature `lance` | Needs a system `protoc`. It is on in release CLI builds and in `rerun_py`. |
| `egui`, `eframe`, `egui-wgpu`, `egui_kittest`, `egui_inspection` | 0.37.0 | Immediate-mode GUI and app framework | The whole UI is rebuilt every frame. The Rerun team maintains egui, so fixes belong upstream (`AGENTS.md` "Addressing review comments"). |
| `egui_tiles` / `egui_plot` / `egui_table` | 0.18 / 0.38 / 0.11 | Viewport tiling, plots, tables | — |
| `wgpu` | 30.0 (`Cargo.toml:582`) | GPU abstraction: Vulkan, Metal, DX12, GL, WebGPU, WebGL2 | `re_renderer` sits on top. On the web, WebGL2 falls back to a "Limited" capability tier (`re_renderer/src/device_caps.rs`). |
| `tokio` | 1.53 (locked 1.53.3) | Async runtime for gRPC, the server and the Python catalog | Not available on wasm. Use the `re_async` shims instead. |
| `tonic` (+`prost`, `tonic-web`, reflection) | 0.14.6 / prost 0.14.4 | gRPC client and server | `tonic-web` allows grpc-web from browsers. The wasm client is a fork, `rerun-tonic-web-wasm-client` 0.8.3. |
| `pyo3` / `numpy` | 0.28.3 / 0.28.0 (`Cargo.toml:442`) | Python bindings | Two RUSTSEC advisories on pyo3 are ignored with justification (`deny.toml`). |
| `clap` | 4.6 | CLI parsing (`rerun` binary) | — |
| `parking_lot` | 0.12.5 | Mutexes and RwLocks | `std::sync::{Mutex,RwLock,Condvar}` are banned (`clippy.toml`). |
| `crossbeam`, `rayon` | rayon 1.12 | Channels and data parallelism (views run in parallel) | `crossbeam::Sender::send` and `unbounded` are banned; use `re_quota_channel`. |
| `mimalloc` | `=0.1.48` | Global allocator, wrapped by `re_memory::AccountingAllocator` | Set in `crates/top/rerun-cli/src/bin/rerun.rs:15-17`. |
| `glam` | 0.33.11 | Math | `Vec{2,3}::normalize` is banned in favour of the safe variants. |
| `mcap`, `re_ros_msg` | mcap 0.25.0 | MCAP/ROS import | — |
| `re_rav1d` (package alias `dav1d`) | 0.1.3 (`Cargo.toml:216`) | AV1 decoding, a Rerun fork of rav1d | Feature `nasm` needs the `nasm` CLI at build time. |
| `ffmpeg-sidecar` | 2.5 (`Cargo.toml:323`) | Runs the **system `ffmpeg` CLI** for H.264/H.265 | ffmpeg is a *runtime* dependency for those codecs (`crates/utils/re_video/src/decode/ffmpeg_cli/`). |
| `re_walkers` | 0.63.1 (`Cargo.toml:259`) | Map tiles; a fork of `walkers` | — |
| `opentelemetry` family | 0.32.x | Perf telemetry (`re_perf_telemetry`) | Off at runtime by default. |
| `rmcp`, `agent-client-protocol`, `rig-core` | rmcp 3.4, ACP 2.1 | Viewer MCP server and the agent panel | Experimental features. |
| `wasm-bindgen` | `0.2.129` (`Cargo.toml:567`), matching the pixi `wasm-bindgen-cli ==0.2.129` | Web build | The version must match exactly. `scripts/build_web_viewer.py:27-40` checks this. |
| Rerun-owned crates on crates.io | `quiver` 0.6.1, `re_mp4` 0.5.1 | Columnar helpers, MP4 demux | — |

`[patch.crates-io]` has only one active patch, `re_workspace_hack` (`Cargo.toml:973-1038`).
`release.yml` refuses to publish while any other crates.io patch is active.

## 3. Python packages

- **`rerun-sdk` runtime dependencies** (`rerun_py/pyproject.toml`): `attrs>=23.1`, `numpy>=2`, `pillow>=8`, `psutil>=7`, `pyarrow>=18`, `typing_extensions>=4.5`.
- **Extras:**
  - `notebook` → `rerun-notebook`
  - `catalog` → `datafusion==54.0.0`, `pandas>=2`
  - `dataloader` → `torch>=2.5`, `torchvision`, `av`
  - `tracing` → `opentelemetry-*`
  - `all`
  - `tests`
- **`rerun-notebook`** depends on `anywidget`, `jupyter-ui-poll` and `ipykernel!=7.0.*`. It is built with hatchling.
- **Dev group** (root `pyproject.toml:18-63`): `maturin>=1.14.1`, `ruff==0.16.9`, `mypy==1.19.1`, `ty==0.0.84`, `pytest`, `syrupy`, `inline-snapshot`, plus script dependencies.
- **Docs group:** `mkdocs==1.6.1`, `mkdocs-material`, `mkdocstrings`.
- **uv constraint:** `pyarrow>=23.0.1,<24` (`pyproject.toml:171`).

## 4. Data stores, caches, queues, object storage

Rerun is **not** a typical web service with an external database. Its "databases" are its own:

| Kind | What | Evidence |
| --- | --- | --- |
| In-memory time-series DB | `re_chunk_store::ChunkStore`, one per recording or blueprint, wrapped by `re_entity_db::EntityDb` | `crates/store/re_chunk_store/src/store.rs:527` |
| Query cache | `re_query::QueryCache` | `crates/store_app/re_query/src/cache.rs:136` |
| File format | `.rrd` (recordings) and `.rbl` (blueprints): protobuf-framed Arrow IPC with an optional footer and manifest | `crates/store/re_log_encoding/src/rrd/` |
| Catalog server storage (OSS) | In-memory catalog metadata; RRD files read lazily from disk; optional Lance tables | `crates/store_app/re_server/src/store/` |
| Message queue | In-process channels (`re_log_channel`, `re_quota_channel`) and the gRPC message proxy's in-memory history (default 1 GiB) | `crates/data_flow/re_grpc_server/src/lib.rs:64-90` |
| Object storage | No direct S3/GCS SDK. `object_store 0.13` comes in only through datafusion and lance. Cloud uploads go through write grants and multipart presigned HTTP (UNVERIFIED in detail). | `crates/utils/re_workspace_hack/Cargo.toml`, `crates/store_app/re_catalog_client/src/staging/multipart.rs` |
| Viewer persistence | eframe storage (RON), `.rbl` blueprint files, a map tile cache | `crates/top/re_viewer/src/app/mod.rs:296-313`, `src/saving.rs:99-154` |
| Search (docs only) | Meilisearch 1.5.1 for docs search indexing (dev/CI) | `pixi.toml` task `meilisearch`, `search-index` |

## 5. Build tools, package managers, monorepo tooling

| Tool | Version | Role | Gotchas |
| --- | --- | --- | --- |
| **pixi** | `>=0.78.0`; CI pins v0.82.1 | Task runner and conda toolchain (cmake, clang 19.1.7, protoc via libprotobuf 6, ffmpeg 7, nasm, wasm-bindgen-cli, node, lychee, typos, taplo, …) | Run almost everything via `pixi run <task>`. Environments are `default`, `cpp` and `coverage` (`pixi.toml:133-160`). `cpp` cannot be the default because it breaks the web build on macOS (issue #6852). |
| **cargo** | Rust 1.98.1 | Rust build | `.cargo/config.toml` sets alias `codegen`, `incremental = false`, `--cfg rerun_workspace_hack`, `PYO3_CONFIG_FILE`, and wasm SIMD flags. |
| **cargo-hakari** | `>=0.9.36` | Feature unification (`crates/utils/re_workspace_hack`) to avoid recompiles | Run `pixi run hakari-generate` after dependency changes. CI checks it with `hakari-check`. |
| **uv** | `==0.13.4` | Python env (`.venv`) and uv workspace (`uv.lock`) | The pixi `uv` wrapper unsets `CONDA_PREFIX` (`AGENTS.md`). |
| **maturin** | `>=1.9.3` build, `>=1.14.1` dev | Builds the Python wheel from `rerun_py` | `pixi run py-build` = `maturin develop`. |
| **CMake + Ninja** | 3.27.6 / 1.11.1 (pixi) | C++ SDK, examples, snippets, tests | Arrow C++ is downloaded and built by default (`rerun_cpp/CMakeLists.txt:117`). |
| **npm, esbuild** | esbuild ^0.25 | JS packages | `rerun_js/web-viewer/build-wasm.mjs` patches the wasm-bindgen output. It is fragile (TODO #8766, `Cargo.toml:557-566`). |
| **Cranelift** (optional) | nightly | Fast dev builds: `pixi run rerun-build-fast` (`.cargo/cranelift.toml`) | Needs a nightly toolchain. |

Cargo profiles (`Cargo.toml:605-815`):

| Profile | Settings |
| --- | --- |
| `dev` | `opt-level=1`, no debug info; deps `opt-level=2` |
| `release` | `panic="abort"`, thin LTO, 1 codegen unit |
| `web-release` | `opt-level='z'`, full LTO |
| `debugging` | `opt-level=0` with debug info |
| `dev-fast` | Cranelift |

## 6. Testing

| Tool | Version | Scope |
| --- | --- | --- |
| cargo-nextest | pixi `>=0.9.140`; CI installs `0.9.89` (`.github/actions/setup-rust/action.yml`) | Rust tests. Profiles are in `.config/nextest.toml`. |
| insta | 1.48 | Rust text snapshots (670 `*.snap`) |
| egui_kittest | 0.37 | UI image snapshots (1 364 PNGs in git-LFS), rendered with a software rasterizer on CI |
| pytest (+ syrupy, inline-snapshot, pytest-benchmark) | pytest 9.0.3 (tests extra) | Python SDK (`rerun_py/tests`) |
| Catch2 | v3.4.0 via FetchContent (`rerun_cpp/tests/CMakeLists.txt:3-18`) | C++ SDK |
| Playwright | — | JS viewer-control tests (`tests/js/viewer_control`) |
| headless_chrome, wasm-pack | — | Browser integration tests (`re_integration_test`, `re_web_tests`) |
| cargo-llvm-cov | `>=0.8.7` | Coverage (`pixi run -e coverage rs-coverage`) |

## 7. Linting, formatting, type checking, hooks

| Tool | Config | Notes |
| --- | --- | --- |
| rustfmt | default (`docs/snippets/rustfmt.toml` uses width 80) | `pixi run rs-fmt` (always run this after changes, `AGENTS.md`) |
| clippy | workspace lints `Cargo.toml:819-972`; `clippy.toml` | `pedantic` plus about 70 restriction lints as warnings; `unsafe_code = "deny"`. About 50 disallowed methods and types steer code to Rerun wrappers. Wasm has a separate `scripts/clippy_wasm/clippy.toml`. |
| cargo-deny | `deny.toml` | Licenses, bans, sources, advisories. 23 ignored RUSTSEC IDs, each with a reason (see `06-RISKS-AND-TODO.md`). |
| cargo-shear | `Cargo.toml [workspace.metadata.cargo-shear]` | Finds unused deps (warnings only, `cargo_shear.yml`). |
| Custom linter | `scripts/lint.py` (`pixi run lint-rerun`) | Enforces `…` over `...`, dash style, the `TODO(name)` format, snapshot options, and more. |
| ruff | `pyproject.toml:291-430`; stricter rules in `rerun_py/pyproject.toml:86-250` | Version 0.16.9 |
| mypy (strict) and ty | `pyproject.toml:432+`, `:481-610` | mypy is 1.19.1 via uv; pixi also pins 1.14.1, which appears unused. |
| clang-format / clang-tidy | `.clang-format`, `rerun_cpp/.clang-tidy` | C++ |
| taplo, prettier, typos, lychee, oxlint, buf | `taplo.toml`, `.prettierrc.toml`, `.typos.toml`, `lychee.toml`, `.oxlintrc.json`, `scripts/ci/buf.yaml` | TOML, YAML/JS, spelling (American English), links, TypeScript, protobuf (`buf breaking` against `origin/main`). |
| Pre-push hook | `hooks/pre-push` → `scripts/pre-push.sh` → `pixi run fast-lint` | Opt-in: `git config core.hooksPath hooks` (`hooks/README.md`). |
| bacon | `bacon.toml` | Watch mode: clippy, check, test, nextest, wasm, doc, run, codegen. |

## 8. CI/CD, containers, deployment targets

- **CI:** GitHub Actions, 53 workflows (`.github/workflows/`). Actions are pinned by SHA, and Rust builds use sccache in GCS (`.github/scripts/setup_sccache.sh`). Details are in `05-DEV-GUIDE.md` §7.
- **Container:** a single CI image, `ci_docker/Dockerfile` (manylinux_2_28 + rustup). There is no production container; the product is shipped as binaries, wheels, crates and npm packages.
- **IaC:** none in this repo. The GCS buckets and load balancers are configured elsewhere; `scripts/update_docs_url_rewrite.py` touches GCP URL maps.
- **Deployment targets:**
  - crates.io, PyPI (`rerun-sdk`, `rerun-notebook`), npm (`@rerun-io/web-viewer`, `-react`), GitHub Releases;
  - GCS buckets `rerun-builds` (build.rerun.io), `rerun-web-viewer` (app.rerun.io) and `rerun-docs` (ref.rerun.io);
  - conda-forge (an external feedstock).

## 9. Third-party services the product talks to

| Service | Purpose | Where | Control |
| --- | --- | --- | --- |
| PostHog (via `tel.rerun.io`) | Opt-out product analytics and crash events | `crates/utils/re_analytics/src/native/sink.rs:11` | `rerun analytics disable`; off in CI, in debug builds, and in any process that does not opt in (`re_analytics/src/lib.rs:327-345`) |
| WorkOS | OAuth login and JWT issuer for Rerun Hub | `crates/utils/re_auth/src/oauth/api.rs:12-15` | `RERUN_OAUTH_*` env vars |
| GitHub releases API | Update check at viewer startup | `crates/top/re_viewer/src/version_check.rs:8` | App option `check_for_updates_on_startup` |
| app.rerun.io | Examples manifest; hosted web viewer; notebook widget JS | `re_viewer/src/ui/welcome_screen/example_section.rs:251`, `rerun_notebook/__init__.py:105` | `EXAMPLES_MANIFEST_URL`, `RERUN_NOTEBOOK_ASSET` |
| OpenStreetMap / Mapbox | Map view tiles | `crates/views/re_view_map/src/map_view.rs:525-570` | `RERUN_MAPBOX_ACCESS_TOKEN` (secret) |
| Anthropic / OpenAI APIs; ACP agents via `npx -y` | Experimental agent panel | `crates/utils/re_agent/src/model_api_backend.rs:200-237`, `agent_harness/profiles.rs:191-290` | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` |
| Rerun Hub (redap cloud) | Remote catalogs and datasets | `crates/data_flow/re_redap_client` | Stored credentials or `REDAP_TOKEN` |

## 10. Observability

| Concern | Implementation | How to use |
| --- | --- | --- |
| Logging | `re_log` on `tracing`/`tracing-subscriber` (`crates/utils/re_log/src/setup.rs`) | `RUST_LOG`. The default is `info` in release and `debug` in debug builds, and noisy crates such as wgpu and naga are turned down (`re_log/src/lib.rs:134-206`). Use `re_log::warn_once!` for per-frame warnings. |
| Profiling | puffin via `re_tracing::profile_function!` and `profile_scope!` | `rerun --profile` serves on `0.0.0.0:8585` and opens `puffin_viewer`. `rerun dump-puffin` converts `.puffin` files to JSON. The `skills/investigate-puffin` skill covers this. |
| Distributed tracing and metrics | `re_perf_telemetry` (OTLP, Prometheus, Tracy) | `TELEMETRY_ENABLED=true` plus the `OTEL_*` env vars (`re_perf_telemetry/src/args.rs`). Mostly for the server side and the Python SDK. |
| Memory | `re_memory::AccountingAllocator`, `MemoryLimit`; memory panel in the dev UI (`re_memory_view`) | `--memory-limit`, `RERUN_TRACK_ALLOCATIONS=1` |
| Error reporting | No Sentry. `re_crash_handler` prints the stack and sends `crash-panic`/`crash-signal` to PostHog if analytics is on (`crates/utils/re_crash_handler/src/lib.rs:21-157`). `scripts/fetch_crashes.py` pulls them for triage. | — |
