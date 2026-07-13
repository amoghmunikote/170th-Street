# What NVIDIA Removed

**What NVIDIA Removed — and Why**

Every restriction on the CMP 170HX was intentional. This page documents each one in detail: what was removed, how it was implemented, what the effect is, and what (if anything) can be done about it.

<figure><img src="https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/roofline-cmp170hx.svg" alt=""><figcaption></figcaption></figure>

**FP32 FMA Throttling — The Core Restriction**

The most significant restriction on the CMP 170HX is the throttling of FP32 Fused Multiply-Add (FMA) instructions to approximately 0.39 TFLOPS — roughly 1/64th of the hardware's theoretical capability of 25 TFLOPS.

FMA is the instruction at the core of nearly every modern compute workload. Matrix multiplication, neural network inference and training, physics simulation, rendering — all rely heavily on FMA. By throttling this single instruction type, NVIDIA made the card incompatible with virtually every piece of ready-made compute software.

This restriction is applied at the driver or firmware level, not by physically disabling hardware. The evidence for this is clear: non-FMA FP32 operations run at 6.25 TFLOPS on the same hardware. The cores are present and functional — NVIDIA is simply enforcing a per-instruction throughput cap in software. This was confirmed by the community when modifying OpenCL programs to use `#pragma OPENCL FP_CONTRACT OFF` and redefining `fma()` as a plain multiply-add raised FP32 performance from 0.39 TFLOPS to 6.25 TFLOPS — a 16× increase from a two-line code change.

The same throttle applies to FP64 FMA, which is reduced to 0.18 TFLOPS. FP64 without FMA runs at 0.094 TFLOPS — useless for scientific double-precision work regardless.

The reasoning is straightforward: Ethereum's Ethash algorithm is memory-bound and integer-heavy. It requires almost no FMA operations. Throttling FMA cost NVIDIA nothing in mining performance while making the card useless for every other compute workload.

**FP16, INT32, and Memory Bandwidth — Untouched**

Notably, NVIDIA did not throttle the instruction types used by Ethereum mining: FP16 runs at its full theoretical \~42 TFLOPS. INT32 runs at its full 12.5 TIOPS. Memory bandwidth delivers its full \~1,355 GB/s. These are the instructions and bandwidth that matter for Ethash — and they are completely unrestricted.

**PCIe Bandwidth — Two Independent Lockdowns**

The PCIe restriction is implemented in two separate layers, each of which would need to be resolved independently:

_Layer 1 — Firmware speed lock:_ The GPU firmware negotiates PCIe at Gen 1 speed (2.5 GT/s) regardless of the host system's capabilities. Even in a PCIe 4.0 x16 slot, the card connects at Gen 1. This is enforced in the VBIOS/firmware and cannot currently be changed due to NVIDIA's Ampere firmware signing.

_Layer 2 — PCB capacitor omission:_ Twelve of the sixteen PCIe data lanes have their AC coupling capacitors removed from the PCB. Without these capacitors, those lanes fail signal integrity checks and the link downgrades to x4 width. The empty 0402 pads are visible on the board. Soldering the missing capacitors would restore x16 width — but because Layer 1 remains, the result would be x16 Gen 1 (\~4 GB/s) rather than x4 Gen 1 (\~1 GB/s). An improvement, but not the game-changer it might seem.

The combined result is approximately 0.85–1 GB/s of PCIe bandwidth — versus 64 GB/s for a full PCIe 4.0 x16 link. This severely limits workloads that require frequent CPU-GPU data transfers.

**VRAM Capacity**

The A100 40GB PCIe ships with five HBM2e stacks on its interposer. The CMP 170HX uses only two, giving it 8GB. For Ethereum mining at the time, this was sufficient — the DAG file was under 5GB. For modern AI and compute workloads, 8GB is a meaningful constraint that limits model size and data set capacity.

**NVLink**

The NVLink physical hardware (connectors and routing) is fully present on the CMP 170HX PCB, identical to the A100. However, NVLink is disabled via fuse-level security in the GPU silicon itself. Some interface ICs and support components are unpopulated, but the root blocker is the fuse disablement, not the component population. Enabling NVLink would require HULK-level cryptographic keys to unlock the fuses — a proprietary NVIDIA license not publicly available. Even if components were populated, the firmware-level security would still prevent activation. NVLink would enable 600 GB/s GPU-to-GPU bandwidth — far beyond the current PCIe bottleneck — but remains inaccessible without a breakthrough in either cryptography or firmware exploitation.

**Display Outputs**

No display output circuitry is present on the board. This is not a firmware restriction — the hardware for driving displays simply does not exist on this card. This is permanent and unmodifiable.

**ECC Memory**

Error-correcting code memory is disabled. For mining, bit flips in hash computations are harmless — the hash simply fails and mining continues. For scientific computing, ECC matters for result integrity. Its absence also provides a marginal increase in effective memory bandwidth since ECC has overhead.

**VBIOS Firmware Signing**

All NVIDIA GPUs since the Turing generation have VBIOS digital signature verification. The GPU checks the cryptographic signature of the firmware before loading it, and rejects any unsigned or incorrectly signed VBIOS. A bypass was found for Turing (RTX 20-series) but it does not apply to Ampere. As a result, all of the above firmware-level restrictions — FMA throttling, PCIe Gen 1 lock — cannot currently be bypassed via VBIOS modification. This is the fundamental barrier to fully unlocking the card, and the focus of ongoing security research documented in the Security & Firmware Research section.
