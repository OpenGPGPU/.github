# OpenGPGPU

**构建可验证、可运行 Linux 的 RISC-V SIMT GPU。**

OpenGPGPU 以 GPU 硬件为核心，覆盖从 Chisel RTL、SIMT 计算与图形流水线，
到 Linux DRM 驱动、QEMU 全系统仿真和 EDA 验证的完整开发链路。

## 核心项目

| 项目 | 定位 | 主要内容 | 语言 |
|---|---|---|---|
| **[gpu](https://github.com/OpenGPGPU/gpu)** | RISC-V SIMT GPU 主项目 | Chisel 7.x RTL，RV32IMF + RVV 向量与浮点流水线；统一着色（vertex/fragment 复用 SIMT）+ 固定功能（clip/raster/插值/output merger/depth-stencil-blend）、Sv32 私有 VA/ASID 与图形 TLB、统一命令（render/compute/copy/fill/blit/strided/resolve/invalidate）、L1/L2 + 共享内存；Linux DRM/KMS 含硬件 vblank IRQ、render-to-KMS present（triangle_present/pipe_present + pipe_opengpu Gallium 雏形）、Debian console（64×64 scanout）、MSAA 1x/2x/4x 与 resolve；默认 8 线程 Verilator，支持 FlashSim 双后端 | Scala |
| **[FlashSim](https://github.com/OpenGPGPU/FlashSim)** | 周期精确 RTL 仿真器 | 基于 CIRCT（firtool/circt-opt/circt-verilog）导入 HW/Comb/Seq 并生成可跳过空闲组合锥的 C++；与 Verilator 逐周期对齐，低活度下数倍至数十倍加速（GpuSystem / GpuHostAxi 等切片详见仓库单测表）；通过顶层门需求提升、兄弟 &/\| 前缀 CSE、NBA 右侧 dmux 分块等优化，可替代 Verilator 嵌入 ARTI/QEMU 驱动真实 QEMU+Linux 链路，最新 wall-clock 以仓库 README 为准 | Python/C++ |
| **[arti](https://github.com/OpenGPGPU/arti)** | RTL 到全系统仿真 | 自动识别 AXI4/AXI-Lite/APB/AHB/AXI-Stream 并生成 SystemC/Verilator 与 QEMU 嵌入模型；自动发现 IRQ 并接线至 GIC + 100μs 轮询；支持 guest-memory 动态 scanout（BASE/stride/control/宽高寄存器、refresh 定时、RGBA 转换）、固定 VRAM simple-framebuffer、console 启动；默认 8 线程 Verilator、Debian cloud-init、外部驱动 KO 依赖自动载入、present 自动演示与 ARTI/FlashSim 后端对比 | Python |
| **[chipagent](https://github.com/OpenGPGPU/chipagent)** | EDA 工具服务层 | 56 个 MCP 工具封装 Yosys/OpenSTA/OpenROAD 等真实工具；覆盖综合、ASAP7 ORFS 到 DEF/GDS、post-route STA、仿真/波形事务分析、对齐检查与 PPA 评估；支持 SV 参数 DSE、多文件 RTL、macro GDS/固定布局、按子串定点 high-fanout 拆分、时钟端口自探测与 Docker/host 双执行 | Python |

## GPU 架构

主 GPU 采用统一着色器与独立固定功能单元相结合的架构：计算和着色程序运行在
同一组 RISC-V SIMT lanes 上，几何处理、光栅化、纹理采样和输出合并由专用 RTL
完成。

```text
Linux / DRM userspace ── pipe_opengpu / triangle_present
         │
Linux DRM/KMS driver ── MMIO + IRQ(vblank/completion) + shared memory
         │
   AXI host interface (control) + AXI memory master
         │
┌───────▼──────────────────────────────────────────────┐
│                    OpenGPGPU                         │
│                                                     │
│  Command buffer ──► Geometry ──► Rasterizer         │
│                            │              │          │
│                            ▼              ▼          │
│                RISC-V SIMT shader cores ──► Texture │
│                            │              │          │
│                            └──────────────► Output   │
│                                      merger / depth │
│                                                     │
│           Shared L1/L2 and host physical memory     │
└─────────────────────────────────────────────────────┘
         │
  ARTI guest-memory GraphicHwOps ── QEMU scanout
```

当前实现重点包括：

- 参数化硬件 warp 与 SIMT 分支发散/重汇合，round-robin 调度；
- RV32I/M、F/D/Zfh 浮点（FMA/sign-inject/minmax/compare/classify/bit-move/转换) 与 RVV 向量流水线（含 FP 异常 fflags 累积、物理向量寄存器文件）；
- 统一计算/着色执行，光栅化、透视插值、纹理（可编程 `vtex.sample`）、MSAA 1x/2x/4x 与 resolve、深度/模板测试；
- Sv32 私有 VA 窗口 + ASID、图形与 CU 翻译、TLB 刷新与 ASID 隔离，host vertex→fragment 翻译与指令 fault 恢复；
- 统一命令与描述符获取、DMA（blit/strided/copy/fill/resolve/invalidate）、共享 L1/L2 与 host 物理内存，无 GPU 本地 VRAM；
- AXI 控制从机 + 内存主机、完成/错误中断与硬件 vblank IRQ（SCANOUT 周期门控，30Hz 对齐 ARTI refresh）；
- Linux DRM/KMS 驱动（GEM/fence/scheduler/display takeover、atomic modeset/page flip）、userspace `pipe_opengpu` 与 `triangle_present`/`pipe_present` render-to-KMS 演示；
- 从 RTL 到 QEMU/Linux 驱动与 userspace 的端到端验证，Debian console 启动，FlashSim/Verilator 双后端（默认 8 线程 Verilator）。

物理进展见 [`timing/README.md`](https://github.com/OpenGPGPU/gpu/blob/main/timing/README.md)：独立 FMA lane 已过 1 GHz 综合（1193 MHz, +162 ps），集成 `GpuSystem` 约 913 MHz 仍未收敛，`strided-copy` 后端路由已通。

## 从哪里开始

GPU RTL 使用 Scala/Chisel，基础开发需要 Java 和 sbt：

```bash
git clone https://github.com/OpenGPGPU/gpu.git
cd gpu
sbt compile
sbt test
```

运行完整的 RTL + QEMU + Linux + DRM 集成测试时，将 `gpu` 与 `arti` 放在同一
目录下，然后执行：

```bash
cd gpu
./scripts/run_arti_gpu.sh              # 默认 8 线程 Verilator
GPU_SIM=flashsim ./scripts/run_arti_gpu.sh  # FlashSim 后端（详见 FlashSim README 实测）
./scripts/run_arti_debian.sh           # Debian console + display 演示
```

这条链路会生成 GPU 顶层 RTL，借助 ARTI 构建嵌入 Verilator/FlashSim 模型的 QEMU，启动
Linux，加载 GPU 驱动，并验证渲染、GEM、atomic modeset、page flip 与 display scanout。

## 开发方向

我们正在围绕以下方向持续推进：

- 扩展 RVV/浮点与 GPU 图形指令覆盖，补齐 shader 纹理/kernarg 与 GEM 绑定；
- 完善 MSAA resolve 私有 DMA VA、上下文 ASID 隔离与 translation pending 队列；
- 改进任务队列、同步、内存层次与渲染吞吐，落地 `pipe_opengpu` Gallium 化与 present 路径；
- 推进综合、时序收敛与 PPA（FMA 已达标，集成系统仍需收敛）及物理实现；
- 建立更完整的 Linux 图形软件栈、Debian 演示与全系统回归（含 FlashSim/Verilator A/B 对比）。

想了解实现细节，请从 **[gpu](https://github.com/OpenGPGPU/gpu)** 开始；全系统
集成、EDA 验证与快速 RTL 仿真分别参见 [ARTI](https://github.com/OpenGPGPU/arti)、
[ChipAgent](https://github.com/OpenGPGPU/chipagent) 和
[FlashSim](https://github.com/OpenGPGPU/FlashSim)。
