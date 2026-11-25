````markdown
<p align="center">
  <img src="./kelly.png" alt="Kelly logo" width="112">
</p>

## **Kelly** (work in progress)

Kelly is an experimental fork of VIctory, a compatibility layer that runs an Xbox Series X|S title natively on Windows using files you supply. It follows upstream `main` and focuses on getting the world to render correctly and improving performance on the test PC.

The repository contains no game files, code or assets. You need your own copy of the game. Kelly is unaffiliated with the game's developer and publisher.

## Credits

VIctory by Marvisak and its contributors is the base of everything here: the loader, the runtime services, the Direct3D 12 back end and the shader recompiler. Licensed GPL-3.0-or-later; so is this fork. Upstream's history is kept intact below the single commit that carries this fork's changes.

XCL by ivall and its contributors (GPL-2.0-only) was used as a reference for how the same title behaves under another layer. Fixes inspired by it were written anew in VIctory's code.

The guard over every view of a memory page is code by myokex (victory-amd, commit c6dc4d8), which adapts pull request #115 here by Spexivus, its correction by joroyt and a part of VIctory #1122 by tightfall. The read-beside fix for kernel reads into title memory is by Joshua Odom (commit 10ea38e of the same fork family). Both are carried here unchanged apart from one merge conflict.

This fork's changes were developed with Claude Code.

## Kelly status · 2026-11-25

Kelly can boot the title, play the opening cutscene and reach the prologue. Gameplay remains experimental because of low frame rates and rendering issues. All Kelly results below come from one Windows 11 PC with an Intel Core i7-14700KF, an NVIDIA RTX 4070 with 12 GB of VRAM, and 32 GB of RAM. Tests used the Series X profile.

| Stage | Result on the test PC |
| --- | --- |
| Boot, menus and start screen | Work at the title's 30 fps. |
| Opening cutscene | Plays from start to finish, including title cards, credits cards, and lit night and dusk shots. |
| Saves | The game autosaves a checkpoint and loads it again. |
| Prologue | Characters, textures, sky and sunlit exteriors render. The HUD, pause menu and controller input work. |
| First room | Later tests showed walls, textures, characters and lamps. The first shot after the cutscene has sometimes loaded without walls. |
| Lighting | Early tests showed black areas outside direct sunlight. The title's ray-traced lighting and shadows remain under development. |
| Performance | Early world tests ran at roughly 2.5–4 fps. Later first-room tests reached about 4.5–5 fps with defaults, with higher results under experimental settings. |
| Stability | Normal runs stayed up on the test PC after a wave64 barrier regression was fixed. Experimental draw windows have caused device loss. |

## Development notes

### Rendering fixes

- Corrected the buffer view used by meshlet culling so world geometry can draw.
- Fixed per-draw descriptor table exhaustion that could send draws to unallocated slots and hang the GPU.
- Fixed readback of compute-generated lighting data for sky, probes and fog across mip levels and layers.
- Fixed translation for fog reconstruction, shadow masks, hair, blend-shape skinning, fire and several material shaders.
- Fixed gathers that read mip level zero when the shader requested another level, including gathers with offsets.
- Fixed invalid DXIL from implicit-level sampling outside pixel shaders in 730 compute programs.
- Added support for rectangle lists, paired depth and stencil views for compute-written depth textures, and texture readbacks.
- Ray tracing is experimental and off by default. The title's acceleration-structure builder shaders translate, shaders can read title memory by raw address, and the ray-box and ray-triangle tests are emulated in DXIL and checked against a GPU reference. At the recorded checkpoint, 59 of 64 trace programs translated. See [the ray-tracing notes](docs/fork-ray-tracing.md).

### First-room rendering

- Descriptor records copied directly by the title are mirrored before submission. Pull request #305 by Flyander, after upstream VIctory `86a5ed6`, covered about 40,000 records per frame whose host slots had been empty or stale.
- Buffer views whose offsets do not align with an element in the host copy now get their own byte copies. The test saw three to six such views per frame.
- Reserving VRAM for first-frame render targets and buffers allowed 98 buffer copies totalling 945 MB, compared with one copy in earlier runs. CPU-side comparison for shader writes fell from 550–710 MB per frame to about 25 MB.
- An early on-demand mip test used the software adapter. A later hardware run with fixes from pull request #747 by AscilCH avoided allocating about 3.1 GB for mip levels the title did not hold and completed without device loss. Enable it with `VICTORY_LEVELS_ON_DEMAND=1`. It is off by default.
- Opening guarded blocks for storage writes became the default in the later update. Fourteen more shader programs translated. The first-half interpolation fallback is applied only after a related translation failure because applying it globally changed 8,551 working pixel programs.
- A copy between title textures backed by the same host texture is skipped. A five-minute run saw 402 such copies.

### Memory and synchronization

- Memory-value waits, write-watch queries and page queries avoid walking page tables on each call.
- `VICTORY_VIEW_GUARD=0` disables tracking of writes across every mapped view of a physical page. On the test PC, tracking caught 0.8–1.0 million writes per five-minute run that previously left copies stale. Walls appeared in later frames in four of four runs after the fix, compared with two of five before.
- Kernel reads use a temporary buffer before copying into title memory so a block guarded again during the read can still receive the data. Failed page-destination reads are retried. Set `VICTORY_DSTORAGE_RETRIES=0` for the previous behavior.
- Storage writes are counted at the physical pages the title maps. Runs found 10,000–50,000 guarded blocks underneath those writes. `VICTORY_PAGE_WRITES` selects the handling mode. The development notes list `guards`, `1`, `slots` and `all`.
- Large title buffers can have GPU-resident copies, with GPU hashing used to find shader writes. `VICTORY_INPLACE=1` enables experimental use of title memory as a graphics heap. The first stage was verified with a probe on the test GPU.
- The test card used 85–99% of its VRAM budget during world runs. A five-minute run released about 26,000 texture copies and recreated about 3,400. Active render targets used roughly 2.8 GB. Rejected texture slots are deferred briefly so the sweep can reach other slots. `VICTORY_FULL_CARD_PASS=0` restores earlier sweep behavior. `VICTORY_CARD_ROOM_FOR_BUFFERS=1` enables an optional buffer reservation.
- The bindless texture queue skips stale views, the view table is bounded, and the pipeline library is memory-mapped. A fix prevents replacement of the library file while it is mapped, which had caused intermittent device loss.
- Shader cleanup includes dead-code elimination, constant folding and value reuse. It cut host instructions in the shader pack by about 43% and saved about 20 ms of GPU time per frame on the test PC. `VICTORY_SHADER_OPT=0` disables it.

### Performance measurements

- Initial world frames took 3–6 seconds. Early follow-up runs took 420–450 ms with defaults and 240–350 ms with `VICTORY_RELAY=2` or with `VICTORY_LATE_WAIT=3 VICTORY_WIDE_BACK=1 VICTORY_CARD_BUFFERS_MIN_MB=1 VICTORY_CARD_BUFFERS_MB=1024`. In one five-minute run, late waits with relay reduced GPU idle time behind submission from about 136 to 30 ms per frame.
- The first rendering update put the first room at 190–450 ms per frame. Descriptor fingerprinting and leaving draw-table slots to their draws reduced the heap pass from 70–220 ms to about 3 ms per frame. A reader-writer lock and per-page updates also reduced contention. Guarded memory was excluded from write-watch queries following work by Jonirulah in #670 and follow-up work after triippywhale.
- A later first-room session reached 6–7 fps with the settings below enabled together. Defaults measured about 4.5–5 fps in that session.
- Subsequent profiling measured a typical 136 ms frame with windows off. The render thread waited 77 ms inside present, while the submit thread spent 118 ms on work including 41 ms copying title memory and 40 ms on copy-back. GPU utilization was 85–98%. Of roughly 130 ms of GPU work, about 16 ms updated GPU copies, 10 ms ran comparisons, and 102 ms went to the title's lists with per-record clamping. Total frame time tracked draw count at around 32 µs per draw.
- In that later session, draw windows raised average first-room performance from 6.0 to 8.9 fps and opening-cutscene performance from 3.1 to 4.2 fps. Results depend on the scene, build and switches.

| Experimental setting | Effect observed or intended |
| --- | --- |
| `VICTORY_ASYNC_SUBMIT=1` | Moves submission preparation and dispatch to a worker. One profile had the title's render thread inside submission for 257 of 292 ms per frame. |
| `VICTORY_QUEUE2=1` | Moves fetches of shader writes to a copy queue. The copy-back thread's wait fell from around 125 ms to 7 ms per frame. Includes a fix for fetches next to a later in-flight submission. |
| `VICTORY_INDIRECT_WINDOW=1` | Enables the cheaper indirect-draw path from pull request #220. Off by default. See the stability notes below. |
| `VICTORY_TEXTURE_NO_WAIT=1` | Allows texture refreshes during memory remapping. Loading samples above one second fell from 18 to 5. |
| `VICTORY_MERGE_GAP=64 VICTORY_MERGE_RUNS=4096` | Merges small gaps between title writes under in-flight shader writes. The merge is approximate. See the source comment at `merge_gap`. |

Upload fills and read-only buffer copies proceed without waiting by default and are checked against subsequent writes. No mismatch was recorded in the reported runs. `VICTORY_FILL_WAIT=1` restores the wait. Up to 512 idle host lists and texture staging buffers are reused.

Other switches built and unit-tested include `VICTORY_COPY_PLAIN=1`, `VICTORY_REFRESH_ONCE=1`, `VICTORY_VIEW_KNOWN=1`, `VICTORY_BACK_OWN_VIEW=1` and `VICTORY_ASYNC_PRESENT=1`. Their benefit remains unverified. `VICTORY_BVH_HOLD=1` stays off because testing showed a loss of indirect light. `VICTORY_EFFICIENCY_MODE=1` keeps the system's efficiency policy in effect. In one hybrid-CPU run, the title's render and recording threads were sampled on fast cores throughout, and the submit worker was sampled on a slow core about 6% of the time.

### Stability and known issues

- Draw windows originally caused three device losses. `VICTORY_INDIRECT_WINDOW_CHECK=1` found a list sending 90 records of 2²⁴ vertices through a window, where the per-draw path allowed eight. The fix skips records requesting 2²³ or more vertices per instance. After the fix, seven of eight reported runs completed without device loss. The remaining run lost the device 44 seconds into the world and the cause is unknown.
- `VICTORY_DRIVER_WATCH` is on by default and disables draw windows for the remainder of a run after a display driver event. Driver events preceded every observed device loss by 2.7–55 seconds. Set `VICTORY_DRIVER_WATCH=0` to disable it.
- `VICTORY_CARD_FETCH_INFLIGHT=1` enables in-flight fetches from pull request #648 by Jonirulah. It cut copy-back wait from 181 ms to zero in one profile but made the GPU busier and was slower in two runs. Keep it off when testing draw windows. One earlier device loss after the draw-window fix occurred with both enabled.
- `VICTORY_WAVE_WIDE=1` enables the 64-lane barrier form that caused a hang in testing. It is off by default.
- A compute pass requests about 77,500 groups in the X dimension. One dispatch runs 65,535 and the excess is cut off.
- Shadows and indirect lighting remain incomplete. The first room can still load without walls in its first shot.
- Gameplay used around 24 GB of memory. The indirect per-record clamp and transfers between system memory and VRAM remain major costs.
- Some workspace lints are temporarily disabled in `Cargo.toml`. Title- or GPU-specific approximations are marked `shortcut` in the source.

## Getting started

### Requirements

- Windows 10 or 11 x64.
- An AVX2-capable CPU.
- A Direct3D 12 GPU with Shader Model 6.6. The world requires mesh shader support.
- The Rust toolchain pinned in `rust-toolchain.toml`.
- Your own copy of the title and several GiB of available Windows commit at startup.

If `crashes.log` reports Windows error `1455`, check **Committed** under **Task Manager → Performance → Memory**. Close other applications or increase the Windows paging file size before retrying.

### Configure and build

Run these commands from the repository root in Windows Command Prompt. Edit `victory.toml` after copying it and point it to title files stored outside the repository.

```bat
cargo xtask install-hooks
copy victory.example.toml victory.toml
cargo test --workspace
cargo run -p victory -- config
cargo build --release -p victory
```

The hooks check the repository's asset policy on commit. Workspace tests do not require game data.

```bat
target\release\victory.exe run "C:\path\to\game\title.exe" --execute --console series-x
```

The window title shows the frame rate. Saves, settings and `crashes.log` are stored in the data folder, which defaults to `%LOCALAPPDATA%\victory\data`. Derived files are stored in the cache folder.

### Controls

Use an XInput controller or, with no controller connected, the keyboard while the title window has focus.

| Keyboard | Controller |
| --- | --- |
| `W A S D` / `I J K L` | Left / right stick |
| Arrow keys | D-pad |
| `Enter`, `Backspace`, `X`, `Y` | A, B, X, Y |
| `Q` / `E`, `Z` / `C` | Shoulders, triggers |
| `F` / `H` | Stick clicks |
| `Esc`, `Tab` | Menu, View |

### Commands

```text
victory inspect <exe|dll> [--imports] [--exports]
victory census [paths...] [--missing] [--strict]
victory lib-exports <import.lib> [--list]
victory shader-info [paths...] [--verbose] [--disasm]
victory translate [paths...] [--validate] [--pipelines] [--list] [--jobs N]
victory run <exe> [--execute] [--unimplemented trap|zero] [--strict]
            [--console series-x|series-s] [-- <title arguments>]
victory config
```

To pretranslate shaders on the CPU, run `victory translate <shader file> --warm all --jobs N`. This took about 4.5–6.5 minutes on the test PC. The shader cache is keyed to the translator source, so builds can reuse it if that source is unchanged.

## Diagnostics

```bat
cargo build --profile diag -p victory --features diagnostics
```

The executable is written to `target\diag\victory.exe`. Builds append crashes and host GPU failures to `crashes.log`. Release builds log at `info` and above by default. Use `VICTORY_LOG=victory_loader=trace,info` for loader tracing.

| Variable | Effect |
| --- | --- |
| `VICTORY_LOG=<target>=debug,...` | Selects debug or trace output by target. |
| `VICTORY_STACKS=20,60` | Logs every thread's stack at the specified times in seconds. |
| `VICTORY_WATCH=<rva>,...` | Logs changes to selected 32-bit values in title memory. |
| `VICTORY_KEYS=30:Enter,35:X` | Sends timed key presses, including when the window is unfocused. |
| `VICTORY_CRASH_SAVE=<rva>:<len>,...` | Saves selected title memory to the cache folder on a crash. |
| `VICTORY_CAPTURE=<dir>` | Saves presented frames at powers of two by default. Use `VICTORY_CAPTURE_FRAMES=<list>` for specific frames. |
| `VICTORY_DUMP_TARGETS=1` | Saves render targets for the selected captured frames. |
| `VICTORY_D3D12_DEBUG=1` | Enables the Direct3D 12 debug layer. |
| `VICTORY_D3D12_GBV=1` | Adds slow GPU-based validation when the debug layer is on. |
| `VICTORY_D3D12_DRED=1` | Reports affected allocations and in-flight commands on GPU failure. |
| `VICTORY_PIPELINE_WAIT_MS=24` | Sets the wait for background pipelines. The default is 500 ms. A draw can be omitted until its pipeline is ready. |
| `VICTORY_SKIP_PIPELINES=<hash>,...` | Skips draws and dispatches using a listed 16-digit shader hash, including indirect commands. |
| `VICTORY_GPU_TIMING=1` | Runs and times submission lists individually and logs slow lists and commands. |
| `VICTORY_D3D12_WARP=1` | Uses the slower software adapter. |
| `VICTORY_SHADER_LOOP_LIMIT=<n>` | Caps translated shader loops per lane, with separate shader cache entries. Default is 2²⁰. |
| `VICTORY_AUDIO_CAPTURE=<file.wav>` | Records audio from the title process. |
| `VICTORY_XMA_DUMP=<dir>` | Saves context and input for the first 32 XMA voices. |
| `VICTORY_PROFILE=1` | Enables the sampling profiler in diagnostics builds. |
| `VICTORY_FRAME_LOG` | Records frame durations and breaks down typical and long frames. Enabled in diagnostics builds. |
| `VICTORY_CARD_METER` | Uses GPU timestamps to break down time by command-list type. |
| `VICTORY_CAPTURE_WORLD_EVERY=64` | Reduces capture frequency during timed runs. `VICTORY_CAPTURE_WORLD` previously captured every eighth world frame, costing roughly 235 ms per capture. |
| `VICTORY_MERGE_RACE=1` | Checks whether copy-back would overwrite newer title bytes. None were observed across 138,000 copy-backs. |
| `VICTORY_WALLS_TRACE`, `VICTORY_CARD_ACCOUNTING` | Add diagnostics for room rendering and GPU memory. A separate census tracks copies retained by the texture sweep. |

## Upstream reference · VIctory

VIctory runs the title's x86-64 code in-process on Windows and provides the platform services it expects. The layer converts data ahead of time where possible and translates shaders on first use. See [architecture](docs/architecture.md).

### Upstream status · 2026-11-23

On Windows 11 with an AMD RX 6700 XT, upstream reached the first mission with indirect draws. Boot and first-run screens ran at 30 fps, the intro video played with sound, profiles and saves worked, and world loading took about 50 seconds. The mission title card rendered, then the scene remained dark at 0.5–1.8 seconds per frame. Entering the world on a hardware GPU could hang the driver and trigger a Windows display reset. The diagnostics build supports `VICTORY_D3D12_WARP=1` for software rendering.

| Subsystem | Upstream implementation and open work |
| --- | --- |
| Loader | Runs the title executable and DLLs in-process and records the faulting code in `crashes.log`. |
| Runtime services | Provides an offline user, licences, packages, durable saves and a text entry dialog. Speech is silent. The dialog and relaunch still needed live testing. |
| Network | The title runs offline. |
| Files and streaming | Reads archives and streamed data and decodes zlib and BCPack in parallel. Writes into title memory limit loading speed. |
| Input | Supports XInput, keyboard and keyboard controller emulation. Trigger rumble is pending. |
| Audio | Menu, video and loading audio work. XMA and Opus decode. Reverb is silent and Opus still needed verification against the title. |
| GPU menus | Runs draw, compute and presentation on Direct3D 12 and copies changed title memory. One release-build screen showed pacing drift. |
| GPU world | Supports mesh pipelines, shader-written buffers, descriptor lifetimes, clamped indirect execution and background pipeline creation with disk caching. Missing shaders drop draws. Queries, depth bounds, HDR and scaling are pending. |
| Shaders | Recompiles to DXIL with disk caching. Menu coverage was complete. World coverage was 8,191/8,506 mesh, 10,617/12,346 vertex/pixel and 1,643/2,539 compute shaders. Open work includes 64-bit image atomics, merged geometry stages and ray-tracing builder shaders. Some compute shaders need 64-lane waves. |
| Textures | Handles tiled layouts and detiling used by the title. Render-target metadata planes are pending. |
| Series S | Reaches the loading screen through the menus. World rendering remains pending. |

## License

Kelly is licensed under GPL-3.0-or-later. See [LICENSE](LICENSE).
````
