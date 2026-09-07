# OpenGPGPU

**构建可验证、可运行 Linux 的 RISC-V SIMT GPU。**

OpenGPGPU 以 GPU 硬件为核心，覆盖从 Chisel RTL、SIMT 计算与图形流水线，
到 Linux DRM 驱动、QEMU 全系统仿真和 EDA 验证的完整开发链路。

## 核心项目

| 项目 | 定位 | 主要内容 |
|---|---|---|
| **[gpu](https://github.com/OpenGPGPU/gpu)** | RISC-V SIMT GPU 主项目 | Chisel 7.x RTL、RV32IMF(+V) 执行、统一着色、固定功能图形流水线、缓存与共享内存、Linux DRM/KMS 驱动 |
| [arti](https://github.com/OpenGPGPU/arti) | RTL 到全系统仿真 | 自动识别 AXI/APB/AHB 等总线，将 Verilated RTL 嵌入 QEMU，让 Linux 通过 MMIO、IRQ 和共享内存直接驱动硬件模型 |
| [chipagent](https://github.com/OpenGPGPU/chipagent) | EDA 工具服务层 | 封装 Verilator、Yosys、OpenSTA、OpenROAD 等真实工具，为仿真、综合、时序、PPA 和物理设计提供可复现反馈 |
| [flashsim](https://github.com/OpenGPGPU/flashsim) | 周期精确 RTL 仿真器 | 基于 CIRCT 导入 RTL 并生成 C++，通过跳过空闲组合逻辑锥加速仿真；与 Verilator 逐周期对齐，GPU 规模设计可获得数倍到数十倍加速，并可替代 Verilator 嵌入 ARTI/QEMU |

## GPU 架构

主 GPU 采用统一着色器与独立固定功能单元相结合的架构：计算和着色程序运行在
同一组 RISC-V SIMT lanes 上，几何处理、光栅化、纹理采样和输出合并由专用 RTL
完成。

```text
Linux / DRM userspace
        │
Linux DRM/KMS driver ── MMIO + IRQ + shared memory
        │
  AXI host interface
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
```

当前实现重点包括：

- 参数化硬件 warp 与 SIMT 分支发散/重汇合；
- RV32I/M、浮点执行和 RVV 向量执行流水线；
- 统一计算/着色执行，以及光栅化、透视插值、纹理和深度测试；
- 软件分配的 command、color、depth 和 texture buffers，无 GPU 本地 VRAM；
- AXI 控制接口、共享内存数据路径、完成中断与 Linux DRM/KMS 驱动；
- 从 RTL 到 QEMU、Linux 驱动和 userspace page flip 的端到端验证。

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
./scripts/run_arti_gpu.sh
```

这条链路会生成 GPU 顶层 RTL，借助 ARTI 构建嵌入 Verilator 模型的 QEMU，启动
Linux，加载 GPU 驱动，并验证渲染、GEM、atomic modeset 与 page flip。

## 开发方向

我们正在围绕以下方向持续推进：

- 扩展 RVV、浮点和 GPU 图形指令覆盖；
- 完善 shader、texture、kernarg 与 GEM 资源绑定；
- 改进任务队列、同步、内存层次和渲染吞吐；
- 推进综合、时序收敛、PPA 优化和物理实现；
- 建立更完整的 Linux 图形软件栈与全系统回归。

想了解实现细节，请从 **[gpu](https://github.com/OpenGPGPU/gpu)** 开始；全系统
集成、EDA 验证与快速 RTL 仿真分别参见 [ARTI](https://github.com/OpenGPGPU/arti)、
[ChipAgent](https://github.com/OpenGPGPU/chipagent) 和
[FlashSim](https://github.com/OpenGPGPU/flashsim)。
