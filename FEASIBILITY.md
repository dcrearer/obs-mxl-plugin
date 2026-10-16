# Feasibility: Porting the OBS MXL Plugins to Rust

**Status:** Exploration / decision record · **Date:** 2026-10-16

## TL;DR

Porting both plugins to Rust is **feasible with moderate effort and no blockers.**
The MXL half is nearly free — the official [`dmf-mxl/mxl`](https://github.com/dmf-mxl/mxl)
repo already ships a Rust binding (`mxl` crate) whose API maps 1:1 onto what the
current C++ plugins call. The real work is on the OBS side: reproducing the plugin
module ABI, the output plugin's frontend/menu integration, its native file dialog,
and platform packaging in a hand-rolled Rust `cdylib`.

Recommended path: a Cargo workspace with a shared `obs-sys` FFI crate plus one
`cdylib` crate per plugin, delivered as three PRs (workspace+CI, input, output),
input first.

## Current state

Two C++ OBS Studio plugins (~2,700 LOC total) built with CMake against the MXL
**C API** (`find_package(mxl CONFIG REQUIRED)` → `mxl::mxl`):

| Plugin | LOC | `obs_*` symbols | Notable extras |
|---|---|---|---|
| Input (source) | ~870 | ~16 | video frame output only |
| Output | ~1,700 | ~27 | `obs-frontend-api` (tools menu + event callbacks), native file dialog (Cocoa `.mm` / Linux), video conversion |

MXL C API in use today:

- **Input:** `mxlCreateInstance`, `mxlCreateFlowReader`, `mxlFlowReaderGetGrain`, `mxlFlowReaderGetInfo`, `mxlReleaseFlowReader`, `mxlDestroyInstance`
- **Output:** `mxlCreateInstance`, `mxlCreateFlow`, `mxlCreateFlowWriter`, `mxlFlowWriterOpenGrain`, `mxlFlowWriterCommitGrain`, `mxlFlowWriterCancelGrain`, `mxlGetCurrentIndex`, `mxlReleaseFlowWriter`, `mxlDestroyFlow`, `mxlDestroyInstance`

Known limitations in the current build: macOS/Linux only (Windows WIP), output locked
to 1920x1080, video-only (no audio).

## The MXL side is low-risk

The official repo now contains a Rust workspace under `rust/`:

- `mxl-sys` (v1.1.0) — raw FFI, generated with `bindgen`, builds MXL via `cmake`, loads
  `libmxl` at runtime via `libloading`.
- `mxl` (v0.2.0) — safe wrapper, **edition 2024**, rust ≥ 1.87, tracking the MXL
  **v1.1.0** release. `publish = false`, so consume it as a **pinned git dependency**.
- `gst-mxl-rs` — a working GStreamer plugin on top of `mxl`, i.e. a real reference for
  driving the binding from a media host.

The safe API maps directly onto today's C++ flow:

| Operation | C API today | Rust `mxl` equivalent |
|---|---|---|
| Instance | `mxlCreateInstance` | `MxlInstance::new(api, domain, options)` |
| Reader | `mxlCreateFlowReader` → `mxlFlowReaderGetGrain` | `create_flow_reader` → `to_grain_reader()` → `get_complete_grain` / `get_grain_non_blocking` |
| Reader info | `mxlFlowReaderGetInfo` | `FlowReader::get_info` / `GrainReader::get_info` |
| Writer | `mxlCreateFlowWriter` → `mxlFlowWriterOpenGrain` → `mxlFlowWriterCommitGrain` | `create_flow_writer` → `to_grain_writer()` → `open_grain` (RAII commit/cancel) |
| Timing | `mxlGetCurrentIndex` | `MxlInstance::get_current_index` / `timestamp_to_index` / `index_to_timestamp` |

**Bonus roadmap wins the Rust API enables:** `SamplesReader`/`SamplesWriter` give audio
flow support, and flow definitions are not hardwired to 1080p — so the audio and
arbitrary-resolution roadmap items become reachable as follow-ups rather than rewrites.

## The OBS side is where the effort is

OBS loads native modules that export a C ABI (`obs_module_load`, the `OBS_DECLARE_MODULE`
macro expansion) and register plugins by filling `obs_source_info` / `obs_output_info`
structs whose fields are C function pointers. All of this is reproducible in a Rust
`cdylib` with `#[no_mangle] extern "C"` exports and `extern "C"` callbacks — standard,
well-understood work. We hand-roll a minimal `obs-sys` FFI over only the symbols each
plugin needs (bindgen over the libobs headers), rather than depend on a community OBS
wrapper crate that would likely lag current OBS and need forking anyway.

Three items carry the real cost / risk:

1. **Native file dialog (output plugin)** — the one genuinely awkward piece in Rust
   (`objc2` on macOS, portal/`ashpd` on Linux). *Mitigation:* keep the existing tiny
   `.mm`/C dialog shim and call it from Rust initially; Rust-ify later.
2. **Packaging** — `cargo` emits a bare `.so`/`.dylib`; macOS OBS expects a `.plugin`
   bundle (the repo's `Info.plist` already exists). Needs a post-build CMake/script step.
3. **ABI pinning** — pin the OBS version and the `mxl` crate to a tag/commit (it is early,
   0.2.0) and regenerate bindings per target to avoid ABI drift.

The output plugin additionally needs `obs-frontend-api` (tools-menu item + frontend event
callbacks) and `obs_output_set_video_conversion`.

## Recommended structure

A Cargo workspace (edition 2024, matching the `mxl` crate):

```
rust/
  obs-sys/          # shared minimal libobs + obs-frontend FFI (thiserror, tracing)
  obs-mxl-input/    # cdylib → OBS source
  obs-mxl-output/   # cdylib → OBS output
```

`mxl` consumed as a pinned git dependency. Logging via `tracing`; error types via
`thiserror` in the wrapper crates.

## Delivery plan (branches → PRs)

| PR | Branch | Contents |
|---|---|---|
| 1 | `rust/workspace` | Cargo workspace, `obs-sys` scaffolding, GitHub Actions CI |
| 2 | `rust/input` | Input (source) plugin — smaller, no frontend/dialog; proves the full OBS↔MXL↔cdylib↔bundle path end to end |
| 3 | `rust/output` | Output plugin — frontend menu, native dialog, video conversion |

Input first, deliberately: it exercises the entire toolchain (module ABI, source
registration, MXL reader, packaging) without the output plugin's frontend/dialog
complexity.

## Testing

Real builds/runs happen on the dev EC2 instance (Linux); no OBS/MXL SDK is assumed on
developer laptops.

- **CI (GitHub Actions, ubuntu + macos matrix):** `cargo fmt --check`,
  `cargo clippy -- -D warnings`, `cargo test`, `cargo audit`. CI builds MXL via its
  `cmake` build-script path.
- **Unit tests:** pixel-format mapping (RGBA / V210 / I420 / NV12 ↔ MXL grain layout) and
  the MXL reader/writer logic.
- **Runtime verification (manual, on EC2):** load each plugin in a real OBS against a live
  MXL domain; confirm frame round-trip and (later) audio.

## Effort estimate (rough)

| Work | Estimate |
|---|---|
| Workspace + `obs-sys` + CI | 0.5–1 wk |
| Input plugin | ~1 wk |
| Output plugin (frontend + dialog + conversion) | ~2 wk |

Scope note: do a video-parity port first; treat audio and arbitrary-resolution support as
follow-ups once parity lands.

## Open risks

- `mxl` crate is early (0.2.0) and `publish = false` → pin tightly; expect some API churn.
- OBS version/ABI drift across platforms → pin and regenerate bindings per target.
- macOS `.plugin` bundle packaging from a `cargo`-built dylib needs a wrapper step.
- Native file dialog is the least "pure Rust" area; a C/`.mm` shim is an acceptable
  interim.

## Decision

Green-light. The hard part is not MXL or Rust FFI — it is faithfully reproducing the
output plugin's frontend, native dialog, and packaging. Proceed with PR 1 (workspace +
`obs-sys` + CI), then input, then output.
