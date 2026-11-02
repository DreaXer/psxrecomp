# GPU draw timing — reference model and implementation

Status: implemented behind `[timing] gpu_draw_timing` (default off;
`PSX_GPU_DRAW_TIMING=0/1` overrides). Was research-only before. Gathered for Power Rangers
Lightspeed Rescue (SLES-03286), whose jump physics only works at the frame
rate the title reaches on hardware (see the PRLR project's HANDOFF.md).

## Current psxrecomp behaviour

- GP0 commands execute instantly when the GPU linked-list DMA emits them.
- GPUSTAT bits 26 (ready for command) and 28 (ready for DMA block) are
  always 1 (`gpu_read_gpustat`, `runtime/src/gpu.c`).
- The linked-list walker (`runtime/src/dma_gpu_ll.c`) charges 1 guest cycle
  per header and 1 per payload word, no block setup.
- `hold_cpu_for_gpu_linked_list` (`runtime/src/dma.c`) holds the CPU for the
  whole walk, and stops holding when an advance moves no list word.
- So `DrawSync` never waits: the title's frame rate is bounded by CPU cost only.

## DuckStation (stenzek/duckstation @ 7a8335ec, src/core/gpu.cpp, dma.cpp)

GPU ticks run at 2 per system clock: `GPUTicksToSystemTicks(t) = max((t+1)>>1, 1)`.

| Work | GPU ticks |
|---|---|
| Polygon setup, `[quad][shaded][textured]` | tri: 46 / 226 (tex) / 334 (shaded) / 496 (shaded+tex); quad: 82 / 262 / 370 / 532 |
| Triangle pixels (`AddDrawTriangleTicks`) | area of the triangle clamped to the drawing area; x2 if textured; +50% if semi-transparent or mask-check; /2 when interlaced field skip |
| Rectangle | setup 16; per row: width, + width (4bpp), 8bpp/15bpp texture-cache formulas; +50% semi/mask; x height |
| Line / poly-line | setup 16; max(width, height) of the clamped rect |
| Fill (GP0 02) | 46 + ((width/8) + 9) x height |
| VRAM to VRAM copy | width x height x 2 |
| CLUT reload | 16 (4bpp) / 256 (8bpp) |
| Environment commands | 1 |

Consumption: commands run while `pending_command_ticks <= max_run_ahead`
(128); the FIFO holds 16 words. GPUSTAT bit 28 is "FIFO has room for the
current command", bit 26 is idle (`pending_command_ticks <= 0`, FIFO empty).
The DMA request follows bit 28, so a busy GPU stalls the linked-list DMA.
Linked-list DMA costs 8 cycles per header + 5 per non-empty block + 1 per word.

## Beetle PSX / Mednafen (libretro/beetle-psx-libretro @ 71ee81e7, mednafen/psx)

`DrawTimeAvail` is credited 2 per system clock (`gpu.c` update) and debited
by the rasteriser; commands are not taken from the FIFO while it is negative.

| Work | GPU ticks |
|---|---|
| Any non-ss command | 2 |
| Polygon base | 64+18 (tri, or first half of quad), 28+18 (second half) |
| Polygon extra | gouraud+textured 450, gouraud 288, textured 180 |
| Polygon span (per line) | 2w if gouraud or textured, 1.5w if semi or mask-eval, else w; 2 per clipped-out line |
| Sprite (per line) | width, +50% semi/mask; setup 16 |
| Line | 2 per pixel step; setup 16 |
| Fill | 46 + ((w>>3)+9) per line |
| VRAM copy | w x h x 2 |
| CLUT reload / texture cache miss | 16 or 256 / 4 |

GPUSTAT bit 26 = no command in progress, `DrawTimeAvail >= 0`, FIFO empty.
Bit 28 = `CalcFIFOReadyBit()`. The CPU is halted while GPU DMA can write
(`RecalcHalt`, `dma.c`), and runs while the GPU FIFO is full.

## Fit for psxrecomp

DuckStation's closed-form per-command cost fits psxrecomp best: it needs only
the packet and draw state, not the rasteriser's spans (the OpenGL path has no
per-span hook). Beetle's numbers are of the same order (textured span 2/pixel,
semi 1.5) and serve as a cross-check.

Integration points:
1. `gpu.c`: after each GP0 command, add its cost to a guest-cycle
   "busy until" deadline; GPUSTAT bits 26/28 derived from it and a 16-word FIFO.
2. `dma_gpu_ll.c`: before a header or payload read, wait until the GPU
   FIFO has room (new op callback); the wait is a stall, not a word.
3. `dma.c`: `hold_cpu_for_gpu_linked_list` must release the CPU at the first
   stall, before consuming it (hardware: the CPU runs while the FIFO is full).
4. Linked-list cost 8/5/1 instead of 1/0/1.

## Measured on PRLR (level 1 rooftop, 4 frames, DuckStation model)

~583 polygons per frame: setup ~292k GPU ticks, pixels ~367k, other ~20k,
total ~680k GPU ticks = ~340k guest cycles = ~10 ms per frame.
The title's CPU loop already uses ~77% of a 50 Hz frame (~15 ms, gcp_ring).
The GPU draws frame N while the CPU builds frame N+1, so ~10 ms of GPU
work alone keeps the frame under 20 ms: GPU timing by itself would leave the
title at 50 fps. The hardware rate must also involve CPU-side costs that
psxrecomp under-charges. A reference run (DuckStation FPS/VPS and GPU usage
at the same spot) is needed before choosing what to implement.

## Reference run (DuckStation, owner, 2026-11-02)

PAL BIOS, no overclock, level 1 rooftop and a busier spot with an enemy:
**24.9 FPS / 49.8 VPS** (game 25 IPS) at both, and the jump works. psxrecomp
runs the same spots at 50 game frames per second.

psxrecomp side, same spot (gcp_ring, 51 frames): ~27% of guest time inside the
VSync wait (everything with `$ra = 0x80083334`), so the title's CPU work is
~73% of a frame (~14.6 ms). `dma_hold_ring`: one ~6840-word linked list per
frame, kicked right after the swap, so the GPU draws in parallel with the CPU
building the next frame. With the DuckStation GPU cost (~10 ms per frame)
running in parallel, the frame still fits in 20 ms. Reaching 25 fps therefore
needs more than the GPU model: either the CPU side costs ~40% more on
hardware/DuckStation than psxrecomp charges, or the GPU/DMA interaction holds
the CPU longer than assumed. To be measured before implementing.

## Implementation (2026-11-02)

- `gpu.c`: `gpu_draw_command_ticks()` (DuckStation costs) charged in
  `gp0_execute_command()` and per poly-line segment into `s_gpu_busy_until`
  (guest cycles, `(ticks+1)>>1`). GPUSTAT bit 26 = idle, bits 28/25 = at most
  `GPU_DRAW_RUN_AHEAD_CYCLES` (64) outstanding. GP1(00)/(01) clear it.
  Not savestate payload (transient, at most one frame of drawing).
- `dma_gpu_ll.c`: new `stall` op consulted before each payload word; a wait
  sets `stalled` and moves no word. `g_dma_gpu_ll_cycles_left` places the
  word inside the advance (the clock already sits at its end).
- `dma.c`: `gpu_ll_stall` (no wait during render passes);
  `hold_cpu_for_gpu_linked_list` stops holding the CPU at the first stall.
- Config: `[timing] gpu_draw_timing` (config_loader), applied in `main.cpp`.
- Not modelled yet: CLUT reload / texture-cache costs, CPU->VRAM transfer
  pacing, DMA linked-list header/setup cost (8/5), CPU halt during resumed
  asynchronous DMA bursts.

Measured on PRLR, level 1 rooftop: the big linked list now completes
~327,800 cycles (~9.7 ms) after its kick (`dma_state` gpu_ot.cycles_last),
its kick stalls after ~15 words and releases the CPU, VSync wait drops from
~27% to ~22%, and DrawSync never waits. **The title still runs at 50 fps**
(dt 0x1400): as predicted, the GPU finishes before the CPU needs it.
Interrupt handling is ~28k cycles per frame (`irq_exc_stats`, VBlank 24.5k).
The missing ~30% is CPU-side or BIOS-side (DuckStation ran the owner's Sony
BIOS; this build runs OpenBIOS). Next: an A/B with a retail BIOS, or the
Beetle oracle.

## Resolution for PRLR (2026-11-02)

DuckStation with OpenBIOS also runs 25 fps (BIOS ruled out); DuckStation with
a 200% CPU overclock runs 50 fps and breaks the jump the same way. The missing
CPU time was the ENHANCED execution profile's generated CPU spans
(`docs/HLE_EXECUTION.md`): an approximate one-cycle-per-instruction budget
without I-cache or load interlock. With `PSX_EXECUTION_PROFILE=REFERENCE` the
title runs 25 fps (dt 0x2800) and jumps, with or without this GPU timing.
GPU timing stays enabled in that title as the more faithful model.
