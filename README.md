# NetFPGA Network Processor — EE 533 Lab Sequence

Verilog RTL, C, and CUDA from **EE 533 (Network Processor Design and Programming)** at USC,
Spring 2026. The labs build incrementally toward a data processing unit that pairs a custom
ARM-compatible CPU with a from-scratch SIMD GPU for packet processing and neural-network
inference.

Everything here was designed, synthesized, and — for the NetFPGA labs — run on real
hardware.

**Author:** Emiliano Fernández Cervantes · M.S. Computer Engineering, USC

---

## Platform and toolflow

| | |
|---|---|
| **Board** | NetFPGA-1G |
| **FPGA** | Xilinx **Virtex-II Pro 50** (`xc2vp50-7`, `ff1152`) |
| **Platform clock** | 125 MHz (8 ns `core_clk` constraint) |
| **Testbed** | Five-node USC rack (`nf0`–`nf4`), separate management network, four-port data plane |
| **Synthesis** | Xilinx **ISE 10.1** |
| **GPU work** | NVIDIA CUDA Toolkit, PTX |

The toolflow is deliberately legacy, because the silicon is: **RTL entry and simulation run
in ISE 10.1 inside a Windows XP VM**, synthesis for the NetFPGA runs in a **separate Fedora
VM**, and the resulting bitfile is downloaded to the board over the testbed. Standing that
flow up and keeping it reproducible across two virtualized environments was a real part of
the work.

---

## Results

### Lab 2 — Structural vs. behavioral 32-bit ALU

Two ALUs with **identical 99-pin interfaces**, synthesized to the same Spartan-3A
(`xc3s700a-4-fg484`), differing only in how they were entered: one hand-built structurally
(1-bit → 8-bit → 32-bit ripple-carry adder, shifter, 8-way mux), one rewritten behaviorally
and left to the synthesizer.

| Design | Slices | 4-input LUTs | Combinational path |
|---|---|---|---|
| Structural (`ALU/`) | 48 | 96 | **81.092 ns** |
| Behavioral (`ALU2/`) | 85 | 160 | **15.919 ns** |

**5.1× shorter critical path for 1.8× the area** — the textbook area/delay tradeoff,
measured rather than assumed. Reports: `Lab 2/ALU*/ALU*.twr`, `ALU*_map.mrp`.

### Lab 4 — NetFPGA reference pipeline, timing clean

The reference pipeline built for this board, as the baseline every later design is measured
against:

| Metric | Result |
|---|---|
| Slices | 15,194 / 23,616 (64%) |
| 4-input LUTs | 21,883 / 47,232 (46%) |
| Block RAM | 108 / 232 (46%) |
| `core_clk` min period | **7.987 ns** |
| Timing | **Clean — 0 errors, score 0** |

7.987 ns against an 8 ns constraint, with 13 ps to spare. Reports:
`Lab 4/lab4/synth/nf2_top.mrp`, `nf2_top_par.twr`.

### Lab 4 — Testbed throughput characterization

UDP, 30-second runs, four concurrent flows against a 1 Gbps target, comparing the NetFPGA
**reference NIC** and **reference router** bitfiles:

| Packet size | Reference NIC | Reference router |
|---|---|---|
| 1472 B | 272.4 Mbps | **765 Mbps** |
| 512 B | 157.1 Mbps | 286.0 Mbps |

> These numbers characterize the **vendor reference designs** on this testbed. They are the
> baseline the custom DPU was later measured against — they are not its throughput.

### CUDA lab — Matrix multiplication, four implementations

Same problem, four ways: a CPU baseline, a naive CUDA kernel, a shared-memory tiled kernel,
and cuBLAS.

| N | CPU | Naive CUDA | Tiled CUDA | cuBLAS |
|---|---|---|---|---|
| 256 | 29.1 ms | 0.033 ms | 0.027 ms | 0.010 ms |
| 512 | 241.1 ms | 0.212 ms | 0.176 ms | 0.035 ms |
| 1024 | 2.41 s | 1.618 ms | 1.232 ms | 0.177 ms |
| 2048 | 78.20 s | 12.797 ms | 9.791 ms | 1.307 ms |
| **4096** | **824.83 s** | **95.29 ms** | **72.83 ms** | **10.07 ms** |

At N=4096: tiling buys **1.31×** over the naive kernel, and **cuBLAS is still 7.2× faster
than the tiled version** — a useful reminder of how much distance remains between a correct
shared-memory kernel and a vendor library. Raw CSVs and plots are in
`CUDA Lab/Resulting Files/`.

The lab also covers 2D image convolution (blur, sharpen, Laplacian, Sobel; 3×3 through 7×7
kernels, 256² through 1024² images) on both CPU and GPU, benchmarking coalescing and
shared-memory bank-conflict effects.

---

## Lab map

| Lab | What it is |
|---|---|
| **Lab 1** | C socket programming — TCP and UDP clients/servers, UNIX-domain sockets, forked multi-process servers |
| **Lab 2** | 32-bit ALU in Xilinx ISE, entered structurally and behaviorally, then compared (see above) |
| **Lab 3** | Verilog **mini-IDS** on NetFPGA — 7-byte pattern detector, word matcher, drop FIFO, dual-port 9-byte memory, comparator chain |
| **Lab 4** | NetFPGA reference pipeline build, synthesis to bitfile, and hardware throughput benchmarking |
| **CUDA Lab** | CPU/GPU matrix multiplication and image convolution — naive, tiled, and cuBLAS |

---

## Where the rest of the project lives

This repository holds the individual lab work through Lab 4 and the CUDA lab. The team
project it builds toward is separate:

- **[USC-HW-Engineers/EE533-DPU](https://github.com/USC-HW-Engineers/EE533-DPU)** — the
  full DPU: ARM-compatible CPU, SIMD GPU, and SoC integration on the NetFPGA.
  My contribution there is **61 of 84 commits**: the 5-stage pipelined ARM-compatible
  datapath, 4-way fine-grained multithreading, the BRAM-based convertible FIFO and MMIO
  interface into the packet pipeline, and the CUDA → PTX → custom machine-code toolchain.

- **[AEGIS — hardware demo video](https://www.youtube.com/watch?v=3BBl78dKvIQ)** — the final
  project built on this platform: a grid intrusion-detection system running neural-network
  inference on the NetFPGA, with **850 ns inference latency measured on the board**.
