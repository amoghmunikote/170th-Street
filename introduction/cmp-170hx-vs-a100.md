# CMP 170HX vs A100

**CMP 170HX vs NVIDIA A100 — What Was Removed and Why**

The CMP 170HX and the A100 PCIe 40GB share the same GA100 die, the same PCB layout, and the same fundamental silicon architecture. Everything that separates them was put there deliberately by NVIDIA. This page documents every difference between the two cards, what was removed, and the reasoning behind each restriction.

**Full Specification Comparison**

**Note:** The CMP 170HX exists in multiple hardware variants. The specifications below are for the 10GB variant (70 SM configuration). The 8GB variant has different core counts. See the Full Specifications page for complete variant details.

| Specification             | CMP 170HX (10GB)                 | A100 PCIe 40GB     | A100 SXM4 40GB |
| ------------------------- | -------------------------------- | ------------------ | -------------- |
| GPU Die                   | GA100-105F-A1                    | GA100-895          | GA100-895      |
| CUDA Cores                | 4,480 (10GB) / 3,584 (8GB)       | 6,912              | 6,912          |
| Tensor Cores              | 280 (10GB) / 224 (8GB)           | 432                | 432            |
| Streaming Multiprocessors | 70 (10GB) / 56 (8GB)             | 108                | 108            |
| Base Clock                | 1,140 MHz                        | 765 MHz            | 765 MHz        |
| Boost Clock               | 1,410 MHz                        | 1,410 MHz          | 1,410 MHz      |
| Max Clock                 | 1,695 MHz                        | 1,410 MHz          | 1,410 MHz      |
| FP32 (theoretical)        | 25 TFLOPS                        | 19.5 TFLOPS        | 19.5 TFLOPS    |
| FP32 (actual, FMA)        | 0.39 TFLOPS                      | 19.5 TFLOPS        | 19.5 TFLOPS    |
| FP32 (actual, no FMA)     | 6.25 TFLOPS                      | 19.5 TFLOPS        | 19.5 TFLOPS    |
| FP16                      | 42 TFLOPS                        | 312 TFLOPS         | 312 TFLOPS     |
| INT32                     | 12.5 TIOPS                       | 19.5 TIOPS         | 19.5 TIOPS     |
| TF32 (Tensor)             | \~6.2 TFLOPS\*                   | 156 TFLOPS         | 156 TFLOPS     |
| Memory                    | 8GB (10GB/16GB variants)         | 40GB HBM2e         | 40GB HBM2e     |
| Memory Bus                | 4,096-bit (8GB) / 5,120-bit (10GB) | 5,120-bit          | 6,144-bit      |
| Memory Bandwidth          | 1,493 GB/s (8GB) / ~1,865 GB/s (10GB) | 1,555 GB/s         | 1,555 GB/s     |
| PCIe Interface            | Gen1 x4 (1 GB/s)                 | Gen4 x16 (64 GB/s) | SXM4 NVLink    |
| NVLink                    | Disabled (fuse)                  | 600 GB/s           | 600 GB/s       |
| TDP                       | 250W                             | 250W               | 400W           |
| Display Outputs           | None                             | None               | None           |
| ECC Memory                | No                               | Yes                | Yes            |
| Compute Capability        | 8.0                              | 8.0                | 8.0            |
| Manufacturing Process     | 7nm TSMC                         | 7nm TSMC           | 7nm TSMC       |
| PCB                       | Near-identical to A100 40GB PCIe | Reference design   | SXM4 module    |
| Original Price            | $4,000–5,000                     | $10,000–11,000     | $15,000+       |
| Current Used Price        | $200–400                         | $2,000–4,000       | $3,000–6,000   |

\*TF32 via cuBLAS workaround only, Tensor Cores not confirmed fully functional

<figure><img src="https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/roofline-cmp170hx.svg" alt=""><figcaption></figcaption></figure>

**What Was Removed and Why**

**CUDA Cores (3,584–4,480 vs 6,912)** The GA100 die contains 128 streaming multiprocessors in its full configuration. The A100 enables 108 of these, and the CMP 170HX exists in multiple hardware variants: the 10GB variant enables 70 SMs (4,480 CUDA cores), while the 8GB variant enables 56 SMs (3,584 CUDA cores). The disabled SMs contain the additional CUDA and tensor cores. This is standard chip binning — dies that don't meet the quality threshold for full A100 production are sold at a lower tier with fewer active cores. This was not done specifically to cripple the CMP; it reflects where these chips sit in NVIDIA's production yield hierarchy.

**FP32 FMA Throttling (0.39 TFLOPS vs 19.5 TFLOPS)** This is the most aggressive and deliberate restriction. Fused Multiply-Add is the core instruction used in virtually every compute workload — machine learning training and inference, physics simulation, rendering, scientific computing. NVIDIA throttled FMA throughput to approximately 1/50th of what the hardware is capable of. Crucially, this throttling is applied at the driver or firmware level, not by disabling hardware. Non-FMA FP32 operations run at 6.25 TFLOPS, and the hardware clearly has far more compute capacity that is being artificially suppressed.

The reasoning is straightforward: Ethereum's Ethash algorithm is memory-bound and integer-heavy, not FMA-heavy. Throttling FMA made the card useless for every workload except mining without affecting its mining performance at all.

**Memory (8GB vs 40GB)** The A100 PCIe 40GB ships with five HBM2e stacks. The CMP 170HX uses only two, giving it 8GB. The Ethereum DAG file at the time was under 5GB, so 8GB was sufficient for mining. Reducing memory also reduces cost and keeps the card useless for large-scale AI training or inference that requires more VRAM.

**PCIe Bandwidth — Two Separate Lockdowns** The PCIe restriction is actually implemented in two independent layers, which means fixing one does not fix the other:

Layer 1 is a firmware-level lock that caps PCIe speed at Gen 1 (2.5 GT/s) regardless of the host slot. Even in a PCIe 4.0 x16 slot, the card negotiates down to Gen 1.

Layer 2 is a physical PCB modification where the AC coupling capacitors for 12 of the 16 PCIe data lanes have been omitted from the board. Without these capacitors, those lanes cannot establish a valid differential signal, forcing the link to downgrade to x4 width. This is trivially reversible by soldering 0402 capacitors onto the empty pads — but because of Layer 1, restoring x16 width still leaves the card at Gen 1 speed (4 GB/s total instead of 1 GB/s).

**NVLink** NVLink is disabled through multiple mechanisms: fuse-level security in the GPU silicon, missing physical components on the PCB, and firmware locks. While the PCB traces routing to the GPU die and edge connectors are present, the three NVLink bridge connectors (RIGHT, MIDDLE, LEFT) are unpopulated, and critical signal-routing resistors are missing. Enabling NVLink would require: (1) populating missing components (small SMD resistors and connectors), (2) bypassing GPU fuse-level locks via HULK cryptographic keys or silicon exploit (neither publicly available), and (3) modifying VBIOS firmware (blocked by signature verification). The fuse-level silicon barrier is the primary obstacle; even full PCB population would not enable NVLink without solving the cryptographic and firmware restrictions.

**ECC Memory** Error-correcting code memory is disabled on the CMP 170HX. For mining this is irrelevant — a bit flip in a hash computation causes the block to fail and mining continues. For scientific computing, ECC is important for result integrity. Disabling it also slightly increases effective memory bandwidth as ECC overhead is eliminated.

**What Was NOT Removed** Despite all of these restrictions, several key capabilities remain fully intact:

The full HBM2e memory bandwidth of 1,493 GB/s is completely unlocked. This is the card's most valuable asset and the foundation of every viable use case. Integer performance (INT32) runs at 12.5 TIOPS with no throttling. FP16 runs at 42 TFLOPS with no throttling. Non-FMA FP32 runs at 6.25 TFLOPS once workloads are modified to avoid the FMA code path. The GA100 die's architectural features — Compute Capability 8.0, the full CUDA feature set, tensor core hardware — are all present.

<figure><img src="https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-pcb-3.jpg" alt=""><figcaption></figcaption></figure>
