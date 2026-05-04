# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`sentrux` — Rust desktop+CLI tool that scans a codebase via tree-sitter, computes a structural quality score (5 root-cause metrics), exposes a live treemap GUI, and serves an MCP server for AI agents. Pure Rust, single binary, no runtime deps. Distributed via Homebrew tap and GitHub Releases. Repo is `github.com/sentrux/sentrux`.

## Workspace layout

Cargo workspace with two members; a third crate (`sentrux-pro`) lives in a **private sibling repo** (`github.com/sentrux/sentrux-pro`) and is only checked out during release builds.

| Crate | Role |
|---|---|
| `sentrux-core/` | All analysis, metrics, renderer, GUI app, MCP server, license validation. Library only. Every module is `pub` so the private `sentrux-pro` crate can consume types (`Tier`, `Snapshot`, `ToolDef`, `McpState`, etc.). |
| `sentrux-bin/` | Thin binary entry — `clap` CLI parsing + `eframe` startup. `src/main.rs` calls `sentrux_bin::run()`; `src/main_impl.rs` is the real entry. `src/lib.rs` re-exports `run()` so `sentrux-pro` can wrap it after setting `Tier::Pro`. |
| `sentrux-pro` *(external, private)* | cdylib loaded at runtime from `~/.sentrux/pro/pro.dylib`. Free binary contains **zero** Pro logic — all Pro features (extra color modes, what-if sim, C4 gen, file-detail panel, refactoring planner, etc.) live in the dylib and register themselves via `pro_registry`. |

`sentrux-core` and `sentrux-bin` share `version` (currently `0.5.7`) — bump together.

## Common commands

```bash
cargo build                                  # debug build (workspace)
cargo build --release                        # release build — produces target/release/sentrux
cargo test                                   # run all tests across workspace
cargo test -p sentrux-core                   # tests in core only
cargo test -p sentrux-core metrics::         # filter by module path
cargo test --test <integration_test_name>    # single integration test
cargo run -- check .                         # run CLI from source
cargo run -- gate --save .                   # save baseline locally
SENTRUX_DEBUG=1 cargo run                    # enable debug_log!() output
SENTRUX_SKIP_GRAMMAR_DOWNLOAD=1 cargo run -- --version   # skip grammar fetch
WGPU_BACKEND=gl cargo run                    # force GL backend if Vulkan flakes
```

There is no Justfile/Makefile/lint task — `cargo` is the whole build surface. CI also does not run `cargo fmt`/`cargo clippy`, so don't assume those gate merges.

### Linux build deps

`libgtk-3-dev libxcb-render0-dev libxcb-shape0-dev libxcb-xfixes0-dev libxkbcommon-dev libssl-dev libvulkan-dev`. ARM64 cross-compile additionally needs `gcc-aarch64-linux-gnu` plus the `:arm64` variants — see `.github/workflows/release.yml`.

## CLI surface (clap, in `sentrux-bin/src/main_impl.rs`)

`sentrux [path]` (default GUI) · `check` · `gate [--save] [path]` · `scan [path]` · `mcp` (also `--mcp` alias) · `plugin {list, add-standard, add, remove, init, validate}` · `analytics {on, off}` · `login` · `pro {activate <key>, status, deactivate, update}`. Add new subcommands by extending the `Command` enum and matching it in the dispatch block in `run()`.

## Runtime data layout (`~/.sentrux/`)

```
plugins/         tree-sitter grammar bundles (per-language dirs with .dylib/.so/.dll + tags.scm + plugin.toml)
pro/             pro.dylib + manifest.json (Pro tier only, watermarked per user)
license.key      Ed25519-signed JSON license
last_update_check, telemetry_pending.json
```

Grammars are downloaded from the most recent release tag that has a `grammars-<platform>.tar.gz` asset (see `ensure_grammars_installed()` in `main_impl.rs`). Embedded plugin TOML/`tags.scm` are synced **after** grammar download and always win, so config drift between binary and tarball is intentionally one-way.

## Pro plugin model (read this before touching license/tier code)

Authoritative spec: `docs/pro-architecture.md`.

- `Tier` enum (`Free < Pro < Team`) lives in `sentrux-core/src/license.rs`. **Do not gate features with `tier.is_pro()` in core code** — that pattern was removed. Use `pro_registry` (`sentrux-core/src/pro_registry.rs`) to look up dynamically registered Pro features. Free binary works the same whether or not the dylib is loaded.
- License validation is offline: parse JSON, verify Ed25519 sig against the embedded `LICENSE_PUBLIC_KEY` in `license.rs`, check `expires`. No network.
- `sentrux_core::license::init()` is called at the top of `run()` and searches multiple paths (handles sudo/system installs). It loads `pro.dylib` via `libloading`, calls `sentrux_pro_init(&ProContext)` so the dylib registers its color modes / MCP tools / panels, then calls `set_tier`.
- Per-user watermark in the dylib is checked against `license.id` — mismatch means a stolen dylib and is silently ignored.
- The release pipeline checks out the private `sentrux-pro` repo (token: `PRO_REPO_TOKEN`) into `../sentrux-pro/` and patches its `../sentrux-fresh` path-dep back to `../sentrux`.

## Architecture reading order (`sentrux-core/src/`)

1. `core/` — `types.rs` (`Snapshot`, `FileNode`, ids), `path_utils`, `settings`, `heat`. The data model the rest of the crate consumes.
2. `analysis/` — scanner pipeline. `plugin/` (per-language grammar loading) → `parser/` (tree-sitter AST → import/call edges) → `graph/` (build dependency graph) → `git.rs`, `entry_points.rs`, `lang_registry.rs`.
3. `metrics/` — quality computation. `arch/` (graph + distance), `dsm/`, `evo/` (git-history-driven), `rules/` (`sentrux check` engine), `whatif/` (Pro), `testgap/` (Pro), `stability.rs`, `root_causes.rs`. Cybernetics theory + formulas: `docs/quality-signal-design.md`.
4. `layout/`, `renderer/` — treemap layout and egui drawing (`rects`, `edges`, `edge_routing`, `heat_overlay`, `minimap`, `badges`).
5. `app/` — eframe app, panels, **`mcp_server/`** (stdio JSON-RPC), `update_check`. The MCP server is what AI agents talk to; new MCP tools are registered here (or via `pro_registry` from the Pro dylib).

The `pro/` directory inside `sentrux-core/src/` (if present) only contains free-tier-side stubs / context types — Pro implementations live in the external crate.

## CI / release pipeline (`.github/workflows/`)

Three workflows with a deliberate ordering — understand before changing tag/release behaviour:

1. **`build-grammars.yml`** — fires on `v*` tags, builds tree-sitter grammar bundles for `darwin-arm64 / linux-x86_64 / linux-aarch64 / windows-x86_64`, uploads `grammars-<platform>.tar.gz` to the release.
2. **`ci-release.yml`** — `workflow_run` triggered after Grammar Build succeeds on the same tag. Downloads grammars from the *current* tag and runs `cargo test` + `cargo build --release`. Avoids the "test needs grammars that don't exist yet" race.
3. **`ci.yml`** — fires on `push` to `main` and PRs (skips tag pushes). Falls back through the most recent 5 release tags to find a grammar bundle, since the new tag's bundle won't exist during the PR build.
4. **`release.yml`** — fires on `v*`, builds the actual binaries on macos-latest / ubuntu-22.04 (x86_64 + aarch64) / windows-latest, and pulls in the private `sentrux-pro` checkout for Pro builds.

Don't add a `cargo test` step that depends on grammars to `ci.yml` without honouring the `for TAG in $(git tag ...)` fallback — fresh forks/branches with no tags will fail.

Windows is intentionally excluded from `ci.yml` test matrix (tree-sitter-scss MSVC build issue, noted in workflow comment).

## Cross-crate invariants worth preserving

- `sentrux-core` exposes everything `pub`. Don't tighten visibility — the private Pro crate breaks if you do.
- `sentrux_bin::run()` is the public entry the Pro crate wraps. Keep its signature `pub fn run() -> eframe::Result<()>`.
- `license::init()` must run **before** any GUI/CLI work in `run()`. Pro feature registration depends on the dylib being loaded first.
- `sync_embedded_plugins()` runs **after** `ensure_grammars_installed()` so embedded configs override stale tarball configs. Don't reorder.
- Free-tier code paths must not assume the Pro dylib is present — every Pro feature lookup goes through `pro_registry` and degrades silently when absent.

## Things to leave alone

- `target/` — large; never read or grep.
- `assets/*.gif` / `assets/logo-*.svg` — used by README, do not regenerate.
- `docs/*.html` — design exploration prototypes, not docs to edit.
- `LICENSE_PUBLIC_KEY` bytes in `license.rs` — paired with an offline private key. Changing it invalidates every issued license.
