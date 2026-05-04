# Sentrux Project Index

Generated structural index of the `sentrux` repository — entry points, module map, CLI/MCP surface, and cross-references.

> Source of truth for *what is*: this file. Source of truth for *why/how*: `CLAUDE.md`, `docs/pro-architecture.md`, `docs/quality-signal-design.md`. If they conflict, the design docs win.

## 1. Workspace

Cargo workspace, two members. Both pinned to **0.5.7** — bump together.

| Crate | Path | Type | Role |
|---|---|---|---|
| `sentrux-core` | `sentrux-core/` | lib | Analysis, metrics, renderer, GUI app, MCP server, license |
| `sentrux` (`sentrux_bin`) | `sentrux-bin/` | bin + lib | Clap CLI + eframe entry; `run()` re-exported for `sentrux-pro` |
| `sentrux-pro` | *external — `github.com/sentrux/sentrux-pro`* | cdylib | Pro features loaded at runtime from `~/.sentrux/pro/pro.dylib` |

Feature flag `pro` exists on both crates but is activated only by the external `sentrux-pro` crate during release builds.

## 2. Entry points

- `sentrux-bin/src/main.rs` → calls `sentrux_bin::run()`
- `sentrux-bin/src/lib.rs` → re-exports `run` from `main_impl`
- `sentrux-bin/src/main_impl.rs` → clap dispatch, `license::init()`, `ensure_grammars_installed()`, `sync_embedded_plugins()`, GUI/CLI launch

Invariant: `license::init()` runs **first** so the Pro dylib can register features before any GUI/CLI work.

## 3. CLI surface

Defined in `sentrux-bin/src/main_impl.rs` (`Command` enum).

| Command | Purpose |
|---|---|
| `sentrux [path]` | Launch GUI (default) |
| `check` | Run rules engine, exit non-zero on violations |
| `gate [--save] [path]` | Quality gate; `--save` writes baseline |
| `scan [path]` | One-shot scan, print snapshot |
| `mcp` (`--mcp`) | Start stdio JSON-RPC MCP server |
| `plugin {list, add-standard, add, remove, init, validate}` | Tree-sitter grammar plugin management |
| `analytics {on, off}` | Telemetry opt-in/out |
| `login` | Pro account auth |
| `pro {activate <key>, status, deactivate, update}` | License lifecycle |

Add new subcommands by extending the `Command` enum and the dispatch match in `run()`.

## 4. Module map (`sentrux-core/src/`)

```
lib.rs              — pub re-exports of all modules (public surface for sentrux-pro)
license.rs          — Tier enum, Ed25519 sig verification, init(), pro.dylib loader
pro_registry.rs     — Runtime registry for Pro-supplied color modes / MCP tools / panels
queries/            — Embedded tree-sitter queries (tags.scm bundles)

core/               — Data model
    types.rs        — Snapshot, FileNode, ids
    path_utils.rs
    settings.rs
    heat.rs

analysis/           — Scanner pipeline
    plugin/         — Per-language grammar loading (libloading)
    parser/         — Tree-sitter AST → import/call edges
    graph/          — Dependency graph build
    git.rs          — Git history walking
    entry_points.rs — Detect entry points
    lang_registry.rs

metrics/            — Quality computation
    arch/           — Architectural distance, graph metrics
    dsm/            — Design Structure Matrix
    evo/            — Evolutionary metrics from git
    rules/          — `sentrux check` rule engine
    whatif/         — Pro: what-if simulation
    testgap/        — Pro: test coverage gaps
    stability.rs
    root_causes.rs  — 5 root-cause aggregator
    cross_validation.rs

layout/             — Treemap layout (rects, viewport, weight, treemap_layout)
renderer/           — egui drawing
    rects, edges, edge_routing, heat_overlay, minimap, badges, colors

app/                — eframe GUI app
    state.rs, mod.rs, scanning.rs, watcher.rs, update_loop.rs, update_check.rs
    canvas.rs, toolbar.rs, status_bar.rs, settings_panel.rs, breadcrumb.rs
    channels.rs, scan_threads.rs, prefs.rs, progress.rs, draw_panels.rs
    panels/         — activity, dsm, evolution, file_detail, health,
                      language_summary, metrics, rules, whatif, ui_helpers
    mcp_server/     — stdio JSON-RPC server
        mod.rs, registry.rs, handlers.rs, handlers_evo.rs
```

## 5. MCP server

- Location: `sentrux-core/src/app/mcp_server/`
- `registry.rs` — `ToolDef` registration; Pro tools register dynamically via `pro_registry`
- `handlers.rs` — Free-tier tool handlers
- `handlers_evo.rs` — Evolution-metric tool handlers
- Entry: `sentrux mcp` (stdio JSON-RPC)

## 6. Pro architecture (summary)

Authoritative: `docs/pro-architecture.md`.

- `Tier` enum (`Free < Pro < Team`) in `license.rs`
- **Never** gate features with `tier.is_pro()` — use `pro_registry` lookup
- Free binary works identically with or without dylib
- `pro.dylib` validated by Ed25519 signature against embedded `LICENSE_PUBLIC_KEY`
- Per-user watermark check vs `license.id` — mismatch silently ignored
- Release pipeline pulls private repo via `PRO_REPO_TOKEN`

## 7. Runtime data layout

```
~/.sentrux/
    plugins/                    — Tree-sitter grammar bundles
    pro/pro.dylib               — Pro plugin (Pro tier only)
    pro/manifest.json
    license.key                 — Ed25519-signed JSON
    last_update_check
    telemetry_pending.json
```

## 8. Tests

In-tree co-located test modules (no separate `tests/` dir):

- `metrics/{arch,dsm,evo,rules,whatif,testgap}/tests*.rs`, `metrics/mod_tests*.rs`
- `analysis/{graph,parser}/tests*.rs`, `analysis/parser/ast_import_test.rs`
- `layout/tests*.rs`
- `app/scanning_tests.rs`
- Shared: `metrics/test_helpers.rs`, `analysis/test_helpers.rs`, `layout/test_helpers.rs`

Run: `cargo test`, `cargo test -p sentrux-core`, `cargo test -p sentrux-core metrics::`.

## 9. CI/CD (`.github/workflows/`)

| File | Trigger | Purpose |
|---|---|---|
| `build-grammars.yml` | tag `v*` | Build grammar tarballs (darwin-arm64 / linux-x86_64 / linux-aarch64 / windows-x86_64) |
| `ci-release.yml` | `workflow_run` after grammars | `cargo test` + `cargo build --release` against current-tag grammars |
| `ci.yml` | push to `main`, PRs | Falls back through last 5 release tags for grammars; Windows excluded |
| `release.yml` | tag `v*` | Build binaries; checks out private `sentrux-pro` |

## 10. Design docs

| File | Topic |
|---|---|
| `docs/pro-architecture.md` | Pro tier, dylib loading, license validation |
| `docs/quality-signal-design.md` | 5 root-cause metrics, cybernetics theory, formulas |
| `docs/*.html` | Logo / icon / color exploration prototypes (do **not** edit as docs) |

## 11. Cross-crate invariants (do not break)

1. Every `sentrux-core` module is `pub` — Pro crate consumes types directly.
2. `sentrux_bin::run() -> eframe::Result<()>` — Pro crate wraps this; signature is load-bearing.
3. `license::init()` runs before GUI/CLI work in `run()`.
4. `sync_embedded_plugins()` runs **after** `ensure_grammars_installed()` — embedded configs must override tarball.
5. Free-tier code never assumes `pro.dylib` is present; degrade silently via `pro_registry`.
6. `LICENSE_PUBLIC_KEY` bytes in `license.rs` are paired with the offline private key — changing them invalidates every issued license.

## 12. Things to leave alone

- `target/` — build output
- `assets/*.gif`, `assets/logo-*.svg` — README assets
- `docs/*.html` — design exploration, not authored docs
- `LICENSE_PUBLIC_KEY` constant
