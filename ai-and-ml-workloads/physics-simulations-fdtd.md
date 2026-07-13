# Physics Simulations — FDTD

**Physics Simulations — FDTD**

Finite-Difference Time-Domain (FDTD) electromagnetic simulation was the original application that motivated the CMP 170HX purchase documented by niconiconi and the subsequent community research that unlocked the card's potential. It is a pure memory-bandwidth-bound workload — one of the best fits for this hardware.

**Why FDTD Is Perfect for the CMP 170HX**

FDTD simulates the propagation of electromagnetic fields through space and time by discretizing Maxwell's equations on a grid. The kernel updates each grid cell based on its neighbors — reading a small amount of data, performing a small number of arithmetic operations, then writing the result. The arithmetic intensity of a basic FDTD kernel is approximately **0.25 FLOPs/byte**.

This is below the FMA ridge point of 0.29 FLOPs/byte — meaning **the FMA throttle is irrelevant for FDTD**. The algorithm is so memory-bound that the card's compute limitation never comes into play. The CMP 170HX's 1,355 GB/s bandwidth is the only thing that matters, and it delivers that bandwidth in full.

**Measured Results**

From niconiconi's benchmarks with a custom FDTD implementation:

```
FDTD kernel arithmetic intensity: 0.25 FLOPs/byte
Cell update rate: 10,110 million cell updates/second
Achieved compute: 181.98 GFLOPS
Memory bandwidth utilized: ~1,156 GB/s (86% of peak)
```

This performance was achieved **without any FMA workaround** — the standard FDTD kernel just works at full speed. The CMP 170HX delivers approximately 1.5× the performance of an RTX 4090 for this workload, purely due to memory bandwidth advantage.

**What FDTD Is Used For**

* PCB electromagnetic compatibility (EMC) simulation — the original use case
* Antenna design and characterization
* Photonic device simulation (waveguides, resonators)
* Microwave and RF circuit design
* Optical scattering simulation
* RADAR and signal propagation modeling

For PCB EMC simulation specifically, FDTD is the gold standard. The CMP 170HX's bandwidth advantage translates directly to larger simulation domains or faster iteration times.

**Available FDTD Frameworks**

**OpenEMS** — open source FDTD solver, uses OpenCL:

bash

```bash
sudo apt install openems
# No FMA workaround needed — arithmetic intensity below FMA ridge point
```

**meep** — open source FDTD by MIT, primarily CPU but can use CUDA:

bash

```bash
sudo apt install python3-meep
```

**Custom CUDA/OpenCL** — for research use, writing your own FDTD kernel is straightforward. Standard formulation with no FMA dependency runs at full bandwidth automatically.

**Key Optimization Points**

For custom FDTD kernels on the CMP 170HX:

* **No FMA workaround needed** — standard kernels run at full speed
* Optimize memory access patterns for coalesced reads — target 256-bit access widths to hit peak bandwidth
* Use `float` (FP32) rather than `double` (FP64) — FP64 is severely throttled even without FMA
* Grid resolution is limited by VRAM — the calculation below assumes the standard 8GB variant; 10GB and 16GB variants can accommodate proportionally larger grids

**VRAM Capacity for FDTD (8GB Variant)**

A 3D FDTD simulation stores 6 field components (Ex, Ey, Ez, Hx, Hy, Hz) per cell in FP32 (4 bytes each). For a grid of N³ cells:

```
VRAM required = N³ × 6 × 4 bytes

N=256:  ~1.6 GB
N=384:  ~5.4 GB
N=420:  ~8.0 GB  ← practical maximum for 8GB variant
N=448:  ~8.5 GB  ← near limit
N=512:  ~12.8 GB ← exceeds 8GB
```

Maximum practical 3D grid on 8GB variant: approximately **420³ cells** at full FP32. The 10GB variant can accommodate N=480-500, and the 16GB variant significantly larger. For 2D FDTD simulations the memory constraint is far less limiting.
