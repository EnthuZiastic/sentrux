# Project Index: sentrux

Generated: 2026-05-04 · Version: 0.5.7 · License: MIT

Rust desktop+CLI tool — scans codebases via tree-sitter, computes structural quality (5 root-cause metrics), serves a live treemap GUI and an MCP server for AI agents. Single binary, no runtime deps.

Repo: `github.com/sentrux/sentrux` · Pro crate (private): `github.com/sentrux/sentrux-pro`

## Project Structure

```
sentrux/
├── Cargo.toml                  # workspace root (resolver 2)
├── Cargo.lock
├── sentrux-core/               # library crate — analysis, metrics, GUI, MCP
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs              # pub re-exports (consumed by sentrux-pro)
│       ├── license.rs          # Tier enum, Ed25519 sig, dylib loader
│       ├── pro_registry.rs     # runtime registry for Pro features
│       ├── core/               # data model: Snapshot, FileNode, paths, settings
│       ├── analysis/           # scanner: plugin → parser → graph → git
│       ├── metrics/            # arch, dsm, evo, rules, whatif, testgap, root_causes
│       ├── layout/             # treemap layout
│       ├── renderer/           # egui drawing
│       ├── app/                # eframe app + panels + mcp_server/
│       └── queries/            # embedded tree-sitter tags.scm bundles
├── sentrux-bin/                # binary crate — CLI + GUI entry
│   ├── Cargo.toml
│   └── src/
│       ├── main.rs             # → sentrux_bin::run()
│       ├── lib.rs              # re-exports run()
│       └── main_impl.rs        # clap dispatch, license init, GUI launch
├── docs/
│   ├── pro-architecture.md         # authoritative Pro spec
│   ├── quality-signal-design.md    # 5 root-cause metric formulas
│   ├── project-index.md            # detailed structural index
│   └── *.html                      # design exploration (do not edit)
├── .github/workflows/
│   ├── build-grammars.yml      # builds tree-sitter grammar bundles per platform
│   ├── ci-release.yml          # cargo test/build after grammars
│   ├── ci.yml                  # PR/main CI (no Windows tests)
│   └── release.yml             # release binaries (pulls private Pro repo)
├── assets/                     # logos, demo gifs (do not regenerate)
├── README.md (+ de/ja/zh-CN translations)
├── CLAUDE.md                   # repo guidance (read first)
└── LICENSE                     # MIT
```

## Entry Points

| Type | Path | Purpose |
|---|---|---|
| Binary | `sentrux-bin/src/main.rs` | Calls `sentrux_bin::run()` |
| Real entry | `sentrux-bin/src/main_impl.rs` | Clap parse, `license::init()`, grammar sync, GUI/CLI launch |
| Library entry | `sentrux-core/src/lib.rs` | Public module surface for `sentrux-pro` |
| MCP server | `sentrux-core/src/app/mcp_server/mod.rs` | stdio JSON-RPC (`sentrux mcp`) |

### CLI subcommands (`Command` enum in `main_impl.rs`)

```
sentrux [path]                          # GUI (default)
sentrux check                           # rules engine, non-zero on violations
sentrux gate [--save] [path]            # quality gate; --save writes baseline
sentrux scan [path]                     # one-shot snapshot
sentrux mcp                             # stdio JSON-RPC MCP server
sentrux plugin {list|add-standard|add|remove|init|validate}
sentrux analytics {on|off}
sentrux login
sentrux pro {activate <key>|status|deactivate|update}
```

## Core Modules

### `sentrux-core` (lib)
- **`core/`** — `types.rs` (Snapshot, FileNode, ids), `path_utils`, `settings`, `heat`. The data model.
- **`analysis/`** — scanner pipeline. `plugin/` (libloading grammar load) → `parser/` (AST → import/call edges) → `graph/` (dep graph build); plus `git.rs`, `entry_points.rs`, `lang_registry.rs`.
- **`metrics/`** — quality computation. `arch/` (graph + distance), `dsm/`, `evo/` (git-history), `rules/` (`check` engine), `whatif/` (Pro), `testgap/` (Pro), `stability.rs`, `root_causes.rs`, `cross_validation.rs`.
- **`layout/`** — treemap: `treemap_layout.rs`, `rects.rs`, `viewport.rs`, `weight.rs`, `types.rs`.
- **`renderer/`** — egui draw: `rects`, `edges`, `edge_routing`, `heat_overlay`, `minimap`, `badges`, `colors`.
- **`app/`** — eframe app: `state`, `scanning`, `watcher`, `update_loop`, `update_check`, `canvas`, `toolbar`, `status_bar`, `settings_panel`, `breadcrumb`, `channels`, `scan_threads`, `prefs`, `progress`, `draw_panels`.
  - **`app/panels/`** — UI panels: `activity`, `dsm`, `evolution_display`, `file_detail`, `health_display`, `language_summary`, `metrics`, `rules_display`, `whatif_display`, `ui_helpers`.
  - **`app/mcp_server/`** — `mod.rs`, `registry.rs` (ToolDef registration), `handlers.rs`, `handlers_evo.rs`.
- **`license.rs`** — `Tier` (`Free < Pro < Team`), Ed25519 sig verification, `init()`, `pro.dylib` loader (libloading).
- **`pro_registry.rs`** — runtime registry for Pro-supplied color modes / MCP tools / panels. **Use this, never `tier.is_pro()`**.

### `sentrux-bin` (bin + lib)
- `main.rs` → thin shim
- `lib.rs` → re-exports `run`
- `main_impl.rs` → clap CLI, dispatch, GUI bootstrap

### `sentrux-pro` (external cdylib)
- Loaded at runtime from `~/.sentrux/pro/pro.dylib`
- Registers Pro features via `sentrux_pro_init(&ProContext)`
- Per-user watermark validated against `license.id`

## Configuration

| File | Purpose |
|---|---|
| `Cargo.toml` (root) | Workspace: `sentrux-core`, `sentrux-bin`; resolver 2; `eframe`/`egui` 0.31; release profile `opt-level=3`, `lto=thin` |
| `sentrux-core/Cargo.toml` | Core deps: tree-sitter 0.25, git2 0.20, ignore, rayon, dashmap, notify, serde, ed25519-dalek, libloading |
| `sentrux-bin/Cargo.toml` | Bin: clap 4, eframe, egui, dirs |
| `Cargo.lock` | Pinned deps |
| `LICENSE` | MIT |

## Documentation

| File | Topic |
|---|---|
| `README.md` (+ `de`/`ja`/`zh-CN`) | Overview, install, MCP integration, rules engine |
| `CLAUDE.md` | Workspace layout, commands, Pro plugin model, CI ordering, invariants |
| `docs/pro-architecture.md` | Authoritative Pro tier spec — license, dylib, registration |
| `docs/quality-signal-design.md` | 5 root-cause metrics, cybernetics theory, formulas |
| `docs/project-index.md` | Long-form structural index |
| `docs/*.html` | Logo/icon/color design exploration — do not treat as docs |

## Test Coverage

- Source files: **121** Rust files
- Test files: **20** (co-located, no separate `tests/` dir)
- Total LOC: **33,722**
- Test locations:
  - `metrics/{arch,dsm,evo,rules,whatif,testgap}/tests*.rs`, `metrics/mod_tests*.rs`
  - `analysis/{graph,parser}/tests*.rs`, `analysis/parser/ast_import_test.rs`
  - `layout/tests*.rs`
  - `app/scanning_tests.rs`
- Helpers: `metrics/test_helpers.rs`, `analysis/test_helpers.rs`, `layout/test_helpers.rs`

Run:
```bash
cargo test                                  # workspace
cargo test -p sentrux-core                  # core only
cargo test -p sentrux-core metrics::        # filter by path
```

CI does **not** run `cargo fmt` or `cargo clippy` — those don't gate merges.

## Key Dependencies

| Crate | Version | Purpose |
|---|---|---|
| `eframe` / `egui` | 0.31 | GUI framework (wgpu, persistence) |
| `tree-sitter` | 0.25 | Multi-language AST parsing |
| `git2` | 0.20 | Git history walking for evo metrics |
| `ignore` | 0.4 | Gitignore-aware file traversal |
| `rayon` | 1 | Parallel scanning |
| `dashmap` | 6 | Concurrent map |
| `notify` / `notify-debouncer-mini` | 7 / 0.5 | File watching |
| `libloading` | 0.8 | Load `pro.dylib` at runtime |
| `ed25519-dalek` | 2 | License signature verification |
| `clap` | 4 | CLI parsing |
| `serde` / `serde_json` / `serde_yaml` / `toml` / `quick-xml` | — | Config + snapshot serialization |
| `crossbeam-channel` | 0.5 | Thread comms |
| `rfd` | 0.15 | File dialogs (gtk3 on Linux) |
| `thiserror` | 2 | Error derives |
| `streaming-iterator` | 0.1.9 | Tree-sitter cursor support |
| `dirs` | 6 | `~/.sentrux/` resolution |

## Quick Start

```bash
# Build
cargo build --release                       # → target/release/sentrux

# Run from source
cargo run -- check .
cargo run -- gate --save .
cargo run                                   # GUI

# Debug
SENTRUX_DEBUG=1 cargo run
SENTRUX_SKIP_GRAMMAR_DOWNLOAD=1 cargo run -- --version
WGPU_BACKEND=gl cargo run                   # if Vulkan flakes

# Test
cargo test
```

### Linux build deps
`libgtk-3-dev libxcb-render0-dev libxcb-shape0-dev libxcb-xfixes0-dev libxkbcommon-dev libssl-dev libvulkan-dev` (ARM64 cross adds `gcc-aarch64-linux-gnu` + `:arm64` variants).

## Runtime Data (`~/.sentrux/`)

```
plugins/                    tree-sitter grammar bundles per language
pro/pro.dylib               Pro plugin (Pro tier only, watermarked)
pro/manifest.json
license.key                 Ed25519-signed JSON
last_update_check
telemetry_pending.json
```

## Cross-Crate Invariants (Do Not Break)

1. Every `sentrux-core` module is `pub` — Pro crate consumes types directly.
2. `sentrux_bin::run() -> eframe::Result<()>` signature is load-bearing.
3. `license::init()` runs **before** any GUI/CLI work in `run()`.
4. `sync_embedded_plugins()` runs **after** `ensure_grammars_installed()` — embedded configs override tarball.
5. Free-tier code never assumes `pro.dylib` is present — degrade silently via `pro_registry`.
6. `LICENSE_PUBLIC_KEY` bytes in `license.rs` are paired with the offline private key — changing them invalidates every issued license.

## CI Ordering (Read Before Touching Tags)

1. `build-grammars.yml` (tag `v*`) → grammar tarballs per platform
2. `ci-release.yml` (`workflow_run` after grammars) → `cargo test` + `cargo build --release` against current-tag grammars
3. `ci.yml` (push/PR) → falls back through last 5 tags for grammars; Windows tests excluded (tree-sitter-scss MSVC issue)
4. `release.yml` (tag `v*`) → builds binaries; pulls private `sentrux-pro` via `PRO_REPO_TOKEN`

Don't add a `cargo test` step needing grammars to `ci.yml` without honoring the tag-fallback loop.

## Things to Leave Alone

- `target/` — build output
- `assets/*.gif`, `assets/logo-*.svg` — README assets, do not regenerate
- `docs/*.html` — design exploration prototypes, not authored docs
- `LICENSE_PUBLIC_KEY` constant in `license.rs`
- `*-bak/` style directories if present — stale backups
