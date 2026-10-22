# OpenGPGPU

**An open RISC-V SIMT GPU that draws a Linux desktop.**

OpenGPGPU is the hardware, the Linux graphics stack, and the tools that run and check them. The GPU is Chisel RTL. A Debian ARM64 guest boots it under QEMU, and Weston presents pixels the RTL rendered.

## Current proof

The checked-in proof is the OpenGPU GTK smoke application on a 640×480 Weston desktop. Its GtkGLArea draws through the OpenGPU Mesa/Gallium driver (`opengpu_dri.so`) and OpenGPU RTL in FlashSim, using four shared compute units of 16 lanes each. `gpu_drv.ko` presents that guest-memory scanout in the QEMU window. The path does not replace a failed render with a CPU-painted frame.

![OpenGPU GTK smoke on a 640×480 Weston desktop](https://github.com/OpenGPGPU/gpu/raw/main/docs/images/opengpu-gtk-smoke-640x480.png)

The [2026-10-17 qualification report](https://github.com/OpenGPGPU/gpu/blob/main/docs/performance/packed-white-modulation-main-qualification-2026-10-17.json) records passing idle and active GTK frame checks and scanout verification. Work landed on 2026-10-22 records that presentation gate as passed, distributes compute grids across the shared compute units, and qualifies native SIMT join fault recovery.

```text
Weston / Wayland
        │
Mesa EGL + OpenGPU Gallium (opengpu_dri.so)
        │
Linux gpu_drv.ko  ·  DRM/KMS ioctls, GEM, fences
        │
ARTI full-system model + OpenGPU RTL (FlashSim or Verilator)
        │
guest GEM framebuffer and scanout registers
        │
QEMU display window
```

Broader stock GTK and Qt shader coverage is still follow-up work. The smoke application and the Weston shell are the qualified desktop path.

## Core projects

| Project | Role | Where it stands | Language |
|---|---|---|---|
| **[gpu](https://github.com/OpenGPGPU/gpu)** | RISC-V SIMT GPU, driver, and userspace | Chisel 7.x RTL. RV32 integer, scalar floating point, and selected RVV, with per-warp scheduling, divergence, and reconvergence. Fixed-function clip, raster, interpolation, depth/stencil/blend, texture, and MSAA 1x/2x/4x with resolve. Shared L2, Sv32 translation, and unified compute, render, and DMA commands. Linux DRM/KMS (`gpu_drv.ko`), Mesa/Gallium, and a fatbin loader in `userspace/libopengpu`. Device code comes from the [llvm-project](https://github.com/OpenGPGPU/llvm-project/tree/opengpu) `opengpu` branch. The desktop proof above is the current system gate. | Scala |
| **[llvm-project](https://github.com/OpenGPGPU/llvm-project/tree/opengpu)** | Device compiler | Fork of LLVM/Clang. OpenGPU work is on the `opengpu` branch; `main` tracks upstream. The CU ISA is the RISC-V vendor extension `XOpengpu`, triple `opengpu-unknown-none-elf`. `clang --target=opengpu` lowers through the RISC-V backend, and the device ABI turns a kernel return into `cease`. Assembler coverage includes `barrier`, `tex.sample`, `cease`, `vtex.sample`, `vquad.dfdx` / `dfdy`, the custom vector branches, and `join`. Milestones M1–M5 are done. On 2026-10-19 a Verilator guest ran the compiled `kadd` kernel (`1 + 2 + 3 = 6`). The published branch tip is `4d61da24` (2026-10-20). Codegen does not yet select the vector-branch and `join` forms. RTL, the driver, and the desktop build do not need this tree. | C++ |
| **[FlashSim](https://github.com/OpenGPGPU/FlashSim)** | Cycle-accurate RTL simulator | Imports HW/Comb/Seq through CIRCT and emits C++ that skips idle combinational cones. DUT ports stay cycle-aligned with Verilator. `GpuHostAxi` / `GpuHostSystemAxi` are the QEMU control tops. On the 2026-09-17 OpenGPU probe (3 rounds), mean wall clock was 6.3 s versus 19.0 s for Verilator-linked QEMU (3.0×). On the qualified four-CU desktop model, host dispatch and invalidation changes measured on 2026-10-17 save 8.21% on a fixed Mesa64 trace, and the 640×480 desktop still passes. Scanout costs about 405 ns/tick versus about 11.5 µs/tick for the 8-region Verilator backend. | Python/C++ |
| **[arti](https://github.com/OpenGPGPU/arti)** | RTL into QEMU | Detects AXI4, AXI-Lite, APB, AHB, and AXI-Stream and builds a SystemC/Verilator project or a model compiled into QEMU. IRQ outputs are wired to the GIC; polling backs off when the model advances no ticks. Integration profiles load an external driver and its `.ko` dependencies. The guest image used by the GPU desktop includes Wayland and libffi (2026-10-21), and each concurrent guest gets its own cloud-init seed. An MCP server can inspect RTL, generate a project, and run the Linux boot as a background job. | Python |
| **[chipagent](https://github.com/OpenGPGPU/chipagent)** | EDA tools behind MCP | 56 MCP tools wrapping Yosys, OpenSTA, OpenROAD, and related tools. Synthesis, ASAP7 OpenROAD Flow Scripts through DEF/GDS, post-route STA, simulation and waveform checks, register and driver alignment, and PPA estimates marked as estimates. The GPU physical flow calls this from `scripts/qualify_ppa.py`. Latest landed work (2026-09-20) is post-route STA and robustness fixes on the ASAP7 flow, including substring-targeted high-fanout buffering. Untrusted RTL runs in Docker and fails closed when Docker is missing. | Python |

## GPU architecture

Compute and shading run on the same RISC-V SIMT lanes. Geometry, rasterization, texture sampling, and output merge are dedicated RTL. Colour and depth live in host DRAM; there is no GPU-local VRAM. The product top, `GpuHostSystemAxi`, has one AXI4 control slave, one AXI4 memory master, and one IRQ.

```text
RISC-V SIMT front-end
  scalar ALU/FPU + selected RVV + warp control
              │
              ▼
        unified shader core ─────── shared memory / L2 / Sv32
              │
              ▼
       command router and host AXI
          ┌──────────┴──────────┐
          ▼                     ▼
   compute + DMA jobs      render host
                                │
                                ▼
      transform → clip → rasterize → interpolate → depth/blend → scanout
```

The same command and memory contracts are used by the Scala tests, the small C userspace library, and the Linux driver.

## Getting started

RTL tests need JDK 11+ and sbt. Verilator is required for the simulation-backed suite.

```sh
git clone https://github.com/OpenGPGPU/gpu.git
cd gpu
sbt compile
sbt test
```

The desktop path is separate. It needs sibling `arti` and `FlashSim` checkouts and the GTK Debian image described in the gpu repo's [GTK coverage notes](https://github.com/OpenGPGPU/gpu/blob/main/docs/GTK_COVERAGE.md). For device code, check out the compiler on its OpenGPU branch:

```sh
git clone -b opengpu https://github.com/OpenGPGPU/llvm-project.git
```

Status of that toolchain is in the gpu repo's [LLVM plan](https://github.com/OpenGPGPU/gpu/blob/main/docs/LLVM_TOOLCHAIN_PLAN.md).

```sh
scripts/build_arti_debian_display.sh
bash scripts/build_mesa_opengpu.sh

OPENGPU_AUTO_DISPLAY=weston \
DEBIAN_BASE=../arti-work/debian-gtk-arm64.qcow2 \
DISK="$PWD/arti-weston-current.qcow2" \
REBUILD_DISK=0 \
REBUILD_CLOUDINIT=1 \
GPU_SIM=flashsim \
QEMU_DISPLAY=cocoa \
bash scripts/run_arti_debian.sh
```

FlashSim is cycle-accurate, and a compositor frame can take minutes. An upright Weston clock and launcher in the QEMU window, plus completed fence-backed draws in the serial log, is the success condition. `scripts/qualify_functional.sh` is the local release gate: the Scala suite, host script tests, and both ARTI guest DRM paths.

## Physical design

Status lives in [`timing/README.md`](https://github.com/OpenGPGPU/gpu/blob/main/timing/README.md). A standalone FP32 FMA lane meets 1 GHz at synthesis (`fma-lane-cspipe`: 1193.57 MHz, +162.17 ps). Integrated `GpuHostSystemAxi` does not: the retained synthesis result is 905.83 MHz (−103.97 ps), and the best measured intermediate is 912.80 MHz. `strided-copy` finishes routing with clean DRC, antenna, and hold; post-route timing is still open (committed baseline 505.06 MHz). A finished tool run is not timing closure, and several of the faster rows were measured on uncommitted RTL.

## Where the work is now

- Cut FlashSim wall-clock time on the desktop workload before widening shader coverage. Correctness stays on the same GTK pixel and scanout checks.
- Widen Gallium and shader coverage for stock GTK and Qt clients, and keep RVV, the memory system, and multiple render targets moving.
- Keep multi-CU compute honest: grids now distribute across the shared compute units, and new shader behavior still has to match the single-CU result.
- Close timing on the integrated top. The FMA lane already meets 1 GHz at synthesis; `GpuHostSystemAxi` does not, and the faster synthesis rows still need a clean committed baseline plus post-route STA.
- Keep the `opengpu` LLVM branch matched to the CU ISA. An encoding or shader-ABI change has to land in both repositories.

Start with **[gpu](https://github.com/OpenGPGPU/gpu)** for the hardware, driver, and desktop procedure. The device compiler is [llvm-project](https://github.com/OpenGPGPU/llvm-project/tree/opengpu) (`opengpu`). Full-system boot is [ARTI](https://github.com/OpenGPGPU/arti), fast RTL simulation is [FlashSim](https://github.com/OpenGPGPU/FlashSim), and physical-flow tools are [ChipAgent](https://github.com/OpenGPGPU/chipagent).
