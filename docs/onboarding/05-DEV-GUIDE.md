# 05 — Developer guide

How verified each command is:

- **[verified]**: the task or command exists in `pixi.toml` or the repo docs, and the definition was read.
- **[UNVERIFIED-run]**: the command exists, but it was not executed during this audit, because the audit was read-only.

Nothing below was run.
Use `pixi run check-env` (`scripts/check_env.py`) to validate your setup.

## 1. From a clean machine to a running viewer

| # | Step | Command | Evidence |
| --- | --- | --- | --- |
| 1 | Install Rust via rustup. The pinned toolchain `1.98.1` (+ wasm target, clippy, rustfmt) installs itself from `rust-toolchain`. | <https://rustup.rs/> | `BUILD.md:17`, `rust-toolchain` |
| 2 | Install pixi `>= 0.78` (CI uses 0.82.1). | <https://pixi.sh> | `BUILD.md:25`, `pixi.toml:40` |
| 3 | Install git-lfs and pull LFS files. Snapshot PNGs, test assets and `crate_graph.svg` live in LFS. | `git lfs install && git lfs pull` | `BUILD.md:59-70`, `.gitattributes` |
| 4 | Generate the PyO3 config. This runs automatically on pixi activation, but run it by hand if you see PyO3 errors. | `pixi run ensure-pyo3-build-cfg` | `BUILD.md:42`, `rerun_pixi_env/pyproject.toml:9` (a console script, not a pixi task) |
| 5 | Validate the environment. | `pixi run check-env` [verified] | `pixi.toml:134` |
| 6 | Build and run the native viewer (no wasm needed). | `pixi run rerun` → `cargo run -p rerun-cli --features release_no_web_viewer` [verified] | `BUILD.md:107`, `pixi.toml` |
| 7 | Build the Python SDK into `.venv` (uv + maturin). | `pixi run py-build` [verified] | `BUILD.md:136` |
| 8 | Run a Python example against your build. | `pixi run uvpy examples/python/minimal/minimal.py` | `AGENTS.md`, `BUILD.md:146` |
| 9 | Optional: install all Python example dependencies. | `pixi run py-sync-examples` [verified, `pixi.toml:516`]. **Note:** `BUILD.md:152` and `examples/python/README.md:67` say `py-build-examples`, which **does not exist**. | — |
| 10 | Optional: web viewer. | `pixi run rerun-build-web` (wasm) or `pixi run rerun-web` (CLI with `--web-viewer`) [verified] | `pixi.toml:213-231` |
| 11 | Optional: C++. On Windows you also need VS 2022. | `pixi run -e cpp cpp-build-all`, then `pixi run -e cpp cpp-test` [verified] | `BUILD.md:187-197` |
| 12 | Optional: a JS package. | `pixi run js-install && pixi run js-build-base` | `pixi.toml` |

**Things that differ from a typical web project:**

- There is no database to set up.
- There are no migrations to run.
- You don't need a `.env` file.
- There are no external services to start.
- For catalog work, start the OSS server with `pixi run rerun server` (or `rerun server -d name=path.rrd`), which is `re_server`'s `Args` at `crates/store_app/re_server/src/entrypoint.rs:17`.

**Common pitfalls:**

- **`re_web_viewer_server` fails to build** with "run `pixi run rerun-build-web`". You built a feature set that embeds the web viewer without the wasm present. Either build the wasm first or use `pixi run rerun`, which excludes the web viewer (`crates/top/re_web_viewer_server/build.rs:9-50`).
- **Snapshot tests fail with tiny PNGs.** git-lfs is not installed, so you have LFS pointer files instead of images.
- **H.264/H.265 video doesn't decode.** The `ffmpeg` CLI is a runtime dependency. pixi provides ffmpeg 7 inside its environment; outside pixi you must install it yourself.
- **`cpp` env and macOS web builds.** The `cpp` environment can break web builds on macOS, which is why it isn't the default (`pixi.toml:133-160`, issue #6852).

## 2. Everyday commands

| Goal | Command |
| --- | --- |
| Check before building (fast) | `cargo clippy -p <crate>` (`AGENTS.md`) |
| Format everything you touched | `pixi run rs-fmt`, `pixi run py-fmt`, `pixi run toml-fmt`, `pixi run -e cpp cpp-fmt`; all at once with `pixi run format` |
| Repo-specific lints | `pixi run lint-rerun [file]` (`scripts/lint.py`); `pixi run fast-lint` (changed files only) |
| Full Rust CI checks locally | `pixi run rs-check` (`scripts/ci/rust_checks.py`; groups such as `--only tests`) |
| Python lint/typecheck | `pixi run py-lint` (ty + mypy), `pixi run py-fmt-check`, `pixi run py-check-signatures` (`.pyi` vs runtime) |
| JS lint | `pixi run js-lint` (needs a built web viewer) |
| Regenerate types after editing `*.def.rs` | `pixi run codegen` (`cargo codegen` alias) |
| Regenerate protobuf code after editing `.proto` | `pixi run codegen-protos` |
| Regenerate CLI docs | `pixi run man` |
| After adding, removing or renaming a crate | `pixi run crate-graph` (updates `crate_graph.svg` and the `ARCHITECTURE.md` tables), then `pixi run hakari-generate` |
| Watch mode | `bacon` (default job: clippy). Other jobs: `check`, `test`, `nextest`, `wasm`, `doc`, `run`, `codegen` (`bacon.toml`). Note that the `i` key is bound to a nonexistent `initial` job. |
| Fast dev builds (nightly + Cranelift) | `pixi run rerun-build-fast`, then `pixi run rerun-fast` |

## 3. Testing

### Run tests

| Scope | Command |
| --- | --- |
| One crate (Rust) | `cargo nextest run --all-features --no-fail-fast -p <crate>` (`AGENTS.md`) |
| One test | `cargo nextest run --all-features -p <crate> <name_substring>` or `-E 'test(=name)'` (standard nextest filters) |
| Full Rust suite, as CI | `pixi run rs-check --only tests` → `cargo nextest run --all-targets --all-features --no-fail-fast` + `cargo test --doc` (`scripts/ci/rust_checks.py:387-402`) |
| Python | `pixi run py-test` (builds first) or `pixi run py-test-no-build`; one test: `pixi run uvpy -m pytest -vv rerun_py/tests -k <name>` |
| C++ | `pixi run -e cpp cpp-test` (`RERUN_STRICT=1`) |
| Redap protocol conformance | `cargo nextest run -p re_server --all-features` (runs `re_redap_tests`; `TESTING.md`) |
| Browser integration | `pixi run rerun-test-web` (headless Chrome, `--profile inspection`); `pixi run web-test` |
| `.rrd` back-compat | `pixi run check-backwards-compatibility` |
| Cross-SDK snippet comparison | `docs/snippets/compare_snippet_output.py` (see `docs/snippets/README.md`) |
| Coverage | `pixi run -e coverage rs-coverage <crate>` |
| Watch mode | `bacon nextest`, or `bacon test` (note: that job uses `cargo test`) |

### Snapshots

| Kind | How to update |
| --- | --- |
| Image snapshots (egui_kittest; 1 364 PNGs in LFS) | Re-run with `UPDATE_SNAPSHOTS=1`. To take CI's images instead, run `./scripts/update_snapshots_from_ci.sh`, or use the kitdiff "Update snapshots" button (`.github/workflows/update_kittest_snapshots.yml`). `pixi run snapshots` shows and cleans failures. |
| Text snapshots (insta; 670 `.snap`) | `cargo insta review`, or `pixi run rs-update-snapshot-tests` |
| Python snapshots (syrupy, inline-snapshot) | `pixi run py-update-snapshot-tests` |

Write image snapshot tests with `re_ui::testing::default_snapshot_options_for_ui/_3d`.
`SnapshotOptions::new` is banned by clippy and `::default` by `scripts/lint.py`.
Pixel thresholds are strict only on Linux CI, which renders with lavapipe (`crates/ui_tools/re_ui/src/testing/mod.rs:68-80`).
That means a test can pass locally and fail on CI.

### What's covered and what isn't

**Strong coverage:**
- UI views and components: `re_component_ui` has 314 snapshot PNGs, `re_view_spatial` 227, `re_blueprint_tree` 91.
- The redap protocol (`re_redap_tests`, 14 k LOC).
- Transform resolution (`re_tf`, 2.3 k LOC of tests).
- Chunk store invariants (`re_chunk_store` tests).
- Cross-language snippet round-trips.

**Weaker or absent:**

| Area | Gap |
| --- | --- |
| `rerun_c` C ABI | Tested only indirectly, through the C++ tests |
| SDK fork handling | The `cleanup_if_forked_child` bug at `crates/top/re_sdk/src/global.rs:94` shows the blueprint branch is untested |
| Untrusted-input parsers (MCAP, CDR, URDF/DAE, RRD) | No fuzzing harness exists in this repo (`06-RISKS-AND-TODO.md` #10) |
| Web blueprint persistence | Not implemented (TODO #2579) |
| OSS server maintenance and multipart upload | Not implemented |

### Flaky and slow tests

- Explicitly ignored:
  - `tests/rust/re_integration_test/tests/inspection/internal_catalog.rs:11` ("flaky on CI", RR-5790);
  - a microbench in `re_datafusion`;
  - a test that needs local data in `re_lerobot`.
- Nondeterministic snapshot tests that use custom thresholds:
  - video: `crates/views/re_view_spatial/tests/integration/video.rs:136-144`;
  - gaussian splats;
  - transparency.
- nextest warns after 30 s and kills a test after 2 min (`.config/nextest.toml`).
  - The `ci` profile retries once, but CI never passes `--profile ci` (see `06-RISKS-AND-TODO.md`).

## 4. Branching, PRs, releases, deployment

- **Where code originates.** This public repo is exported from a private monorepo, "reality":
  - Commits carry `Source-Ref:` trailers.
  - There are sync workflows: `pr-trigger-reality-sync.yml` and `notify-reality-sync.yml`.
  - `RELEASES.md` describes merge-back.

  Some process (the changelog-label check, branch protection, the nightly build) therefore lives outside this repo (UNVERIFIED from here).
- **PRs** (`.github/pull_request_template.md`):
  - Sections: Related, **Why**, What.
  - Exactly one changelog label: `exclude from changelog`, `🪳 bug`, `📉 performance`, or `include in changelog`. The last one needs a file in `docs/content/changelog/upcoming/`.
  - LLM-authored PRs must carry an "Agent" disclosure (`AGENTS.md` "Contributing").
  - Fork PRs from `main` or `master` are rejected (`enforce_branch_name.yml`).
  - Maintainers can comment `@rerun-bot full-check` to run the main-branch suite on a PR.
- **CI per PR** (`on_pull_request.yml` → `reusable_checks*.yml`):
  - checks: codegen freshness, crate layering, lints, typos, links;
  - Rust lint and tests on Linux; Python lint and tests; C++;
  - wheel build and test; web and JS builds; per-PR web viewer upload.

  Fork PRs use `on_pull_request_contrib.yml`, which runs without secrets.
- **Main** (`on_push_main.yml`):
  - full checks, including the macOS and Windows tests;
  - docs deploy;
  - builds of the CLI, `rerun_c`, wheels, web and JS, uploaded to `build.rerun.io/commit/<sha>`;
  - a pip index at `build.rerun.io/version/main/simple`.
- **Releases** (`RELEASES.md`, `.github/workflows/release.yml`):
  - **Versioning:** lockstep semver across all SDKs. A minor release ships roughly every 2 weeks and may break things. Pre-releases are `-alpha.N` and `-rc.N`. `main` is always `X.Y.Z-alpha.N+dev`.
  - **Branches:** `prepare-release-0.x.y[-alpha.N]`, where the branch name is the version.
  - **Run:** `release.yml` via workflow dispatch (alpha / rc / final). It bumps versions, publishes to crates.io, PyPI, npm and GCS, and tags the release. A final release force-pushes `latest` and `docs-latest`, which redeploys the docs.
  - Releases are driven by a private `oss-release` skill.
- **Changelog:**
  - Per-PR files go in `docs/content/changelog/upcoming/`.
  - `skills/assemble-changelog` turns them into `docs/content/changelog/changeset-0-xx.md`.
  - `scripts/generate_changelog.py` writes the `CHANGELOG.md` detail sections.

## 5. Code conventions actually used

| Convention | Rule | Where it is defined or enforced |
| --- | --- | --- |
| Generated code | **Never edit** files marked "DO NOT EDIT". Extend them via `*_ext.rs`, `*_ext.py` or `*_ext.cpp`. | `AGENTS.md` "Code generation system"; CI `codegen --force --check` |
| Crate layering | Depend only downward (`crates/README.md`). | `scripts/check_crate_dependencies.py` |
| Prose and strings | `…` not `...`; spaced em dash ` — `; one sentence per line in Markdown; `format!("{x}")` with inline args | `AGENTS.md`, `DESIGN.md`, `scripts/lint.py` |
| Error messages | Error first, file path last | `AGENTS.md` |
| TODOs | `TODO(name)`, `TODO(#issue)` or `TODO(RR-…)` | `scripts/lint.py`; `scripts/zombie_todos.py` (its CI job is disabled) |
| Locks | `parking_lot` only; no upgradable read locks | `clippy.toml` disallowed types and methods |
| Channels and threads | Use `re_quota_channel`; no `std::thread::spawn`, `std::sync::mpsc` or unbounded tokio channels | `clippy.toml` |
| Arrow | Use the `re_arrow_util` helpers instead of `arrow::compute::{concat,filter,take}` and `RecordBatch::try_new` | `clippy.toml` |
| Colors and UI | Design tokens only (no hardcoded `Color32::from_*`); accessible widget names | `clippy.toml`, `crates/ui_tools/re_ui/src/design_tokens.rs`, `DESIGN.md` |
| `unsafe` | Denied workspace-wide; opting out needs `allow(unsafe_code)` plus `// SAFETY:` comments | `Cargo.toml:821,917` |
| Logging | `re_log::{debug,info,warn,error}`, `warn_once!` in per-frame code; `debug_assert!` → `re_log::debug_assert!` | `crates/utils/re_log`, `clippy.toml` disallowed macros |
| Profiling | `re_tracing::profile_function!()` at the top of costly functions | e.g. `crates/store/re_chunk_store/src/writes.rs` |
| wasm-safety | No threads, filesystem, env or `std::time` on wasm; use `re_async`, `web-time` and `cfg(target_arch = "wasm32")` gates | `ARCHITECTURE.md` "Wasm", `clippy.toml` |
| Dependency injection | No DI framework. Functionality is injected through registries (`ViewClassRegistry`, `ComponentUiRegistry`, `FallbackProviderRegistry`, importer registry) and contexts passed down (`ViewerContext`). | `crates/top/re_viewer/src/app/registries.rs:30` |
| Viewer mutations | Never mutate stores mid-frame. Send `SystemCommand::AppendToStore` or `ViewportCommand`, which are applied at the end of the frame. | `crates/viewer_support/re_viewer_context/src/blueprint_helpers.rs`, `re_viewport_blueprint/src/viewport_command.rs` |
| Python docs | MkDocs/mkdocstrings Markdown, never reStructuredText | `AGENTS.md` "Python docstring formatting" |

**Example files to imitate:**

- **A simple view:** `crates/views/re_view_bar_chart/src/view_class.rs` (481 LOC).
- **A visualizer:** `crates/views/re_view_spatial/src/visualizers/points3d.rs`.
- **A new archetype:**
  - a definition in `crates/build/re_type_definitions/rerun/archetypes/`;
  - then `pixi run codegen`;
  - then extensions in `crates/store/re_sdk_types/src/archetypes/*_ext.rs`;
  - and a snippet in `docs/snippets/all/archetypes/`.
- **An importer:** `crates/data_flow/re_importer/src/importer_rrd.rs` for a built-in one, or `examples/rust/external_importer` for an external plugin.
- **Custom views in user code:** `examples/rust/custom_view`, `examples/rust/custom_visualizer`.
