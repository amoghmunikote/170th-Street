# What Works and What Doesn't

**What Works and What Doesn't**

A practical reference guide for deciding whether the CMP 170HX is suitable for your workload.

**Quick Reference**

| Workload                          | Result         | Reason                                |
| --------------------------------- | -------------- | ------------------------------------- |
| LLM inference FP16 (llama.cpp)    | ✅ Good         | Memory-bound decode, FP16 unthrottled |
| LLM inference Q4/Q8 quant         | ✅ Good         | Memory-bound, integer dominant        |
| INT8 inference (quantized models) | ✅ Good         | Integer unthrottled, confirmed working |
| INT4 inference                    | ❓ Unknown      | Not tested, status unclear            |
| FDTD EM simulation                | ✅ Excellent    | Below FMA ridge point, pure BW        |
| FluidX3D LBM CFD (FMA disabled)   | ✅ Excellent    | 90% of A100                           |
| Memory-bound stencil compute      | ✅ Excellent    | Full 1,355 GB/s available             |
| Hashcat / password cracking       | ✅ Good         | Integer dominant                      |
| Sparse matrix operations          | ✅ Good         | Memory-bound                          |
| FP16 matrix multiply (PyTorch)    | ✅ Good         | FP16 unthrottled                      |
| LLM inference FP32                | ⚠️ Poor        | FMA throttle severely impacts prefill |
| FP32 training (standard)          | ❌ Avoid        | FMA throughout training loop          |
| BF16 training or inference        | ❌ Avoid        | Same FMA path as FP32                 |
| Blender CUDA rendering            | ❌ Poor         | FMA-heavy rendering kernel            |
| FP64 scientific compute           | ❌ Avoid        | Throttled and low even without FMA    |
| Models over 8GB VRAM              | ❌ Avoid        | PCIe bottleneck kills offload speed   |
| Multi-GPU PCIe workloads          | ⚠️ Limited     | 0.85 GB/s PCIe is severe bottleneck   |
| Real-time video encoding          | ❌ Not possible | NvEnc not present                     |

**Detailed Assessments**

**LLM Inference — The Sweet Spot**

The CMP 170HX is a legitimate inference accelerator for single-request decode-phase workloads. The card's behavior differs significantly between prefill and decode phases:

**Decode Phase (Token Generation):**
- Purely memory-bandwidth bound at batch size 1
- Single token generation: GPU reads all model weights once, produces one output token
- With 1,355 GB/s bandwidth, delivers competitive token/second rates
- Current performance: Latency-bound rather than compute-bound (batching shows sub-linear throughput improvement)
- Increasing batch size shows diminishing returns for throughput (more latency-bound than compute-bound)

**Prefill Phase (Prompt Processing):**
- More compute-intensive than decode
- Affected by FMA throttle even at FP16
- Noticeably slower than on unthrottled cards, especially for long prompts
- For typical interactive single-request use, still acceptable

**Quantization Approach:**
- Q4/Q8 quantization works well
- DP4A (4-lane dot product) is heavily throttled — use with caution
- DP2A (2-lane dot product) remains unthrottled and can be used for custom quantization schemes (~2× speedup vs DP4A)

**Batching Limitations:**
- Batching on single-request inference frameworks (llama.cpp) has significant limitations
- Decode-specific batching may improve throughput but with trade-offs in latency

Best models for this card: 3B–8B parameter models at Q4\_K\_M or Q5\_K\_M quantization, single-request / batch size 1 for consistent performance.

**Scientific Simulation — The Original Use Case**

Memory-bound physics simulations are where the CMP 170HX is most competitive. Any algorithm with arithmetic intensity below \~1.7 FLOPs/byte (with FMA disabled) will run at near-full bandwidth — competing directly with A100-class hardware at a fraction of the cost. FDTD, LBM, finite element methods with memory-bound kernels, molecular dynamics with simple force calculations — all fit this profile.

**Standard ML Training — Avoid**

The FP32 FMA throttle makes standard training loops impractical. A training iteration involves FMA-heavy forward passes, FMA-heavy backward passes, and FMA-heavy optimizer updates. The combination produces training speeds far below any consumer GPU. Do not use this card for standard ML training without deep framework-level modifications.

**The PCIe Problem**

Several otherwise viable workloads are limited by the 0.85 GB/s PCIe bandwidth rather than GPU compute or memory bandwidth:

* Multi-GPU workloads that require frequent inter-GPU communication via CPU
* Workloads that transfer large datasets from system RAM to GPU repeatedly
* Any pipeline where the CPU and GPU alternate heavily

For workloads that can load all data into VRAM once and stay there, PCIe bandwidth is irrelevant. For streaming workloads that constantly feed new data from system RAM, the PCIe bottleneck is severe.

**Unlock Status and What's Been Validated**

Community testing has validated several aspects of the unlock potential:

- **BAR0 writes work** and persist through driver reload, enabling configuration changes without GPU reboot
- **Tensor core unlock confirmed** at 200 TFLOPS (vs. FMA-throttled throughput) for matrix operations
- **Memory geometry register modifications work** when using direct Falcon register writes (not the indirect engine, which truncates addresses to 16-bit causing writes to wrong offsets)
- **Compute state modifications problematic** — attempts to modify Falcon state directly result in uninitialized driver state rather than the desired behavior
- **Persistence model** — BAR0 changes survive driver reload but not full power cycles or DEVINIT; may require exploit rerun after reboot

**Memory Variants and Constraints**

The CMP 170HX is available in multiple hardware configurations: **8GB** (original, 56 SMs), **10GB** (confirmed production, 70 SMs), and potentially **16GB** (theoretically achievable, not yet confirmed in production). These are hardware binning variants, not just firmware differences. The 8GB variant operates on a 4,096-bit bus while the 10GB variant uses a 5,120-bit bus.

**8GB Configuration (56 SMs):**
Sufficient for:
* Any 3B–7B model at Q4–Q8 quantization
* FDTD grids up to approximately 420³ cells
* FluidX3D domains up to approximately 400³ lattice sites
* Standard computer vision inference (ResNet, ViT, etc.)

Insufficient for:
* 13B+ parameter models at Q4 quantization (require 10–12GB+)
* 70B models in any quantization
* Large batch inference with long context windows
* Multiple models loaded simultaneously

**10GB Configuration (70 SMs):**
- Enables 13B parameter models at Q4/Q5 quantization
- Improved bandwidth (~1,865 GB/s theoretical) vs 8GB due to 5,120-bit bus and 5 HBM2e stacks
- Enables more flexible context windows and batch sizes
- PCIe bottleneck remains unchanged (still Gen 1 x4)
- Additional compute capability (70 vs 56 SMs) benefits matrix operations

**16GB Configuration (Theoretically Achievable):**
- Would support 70B models at INT4 quantization
- Requires matching hardware configuration with additional HBM2e stacks (not confirmed in production)
- Would also require VBIOS modification and GSP firmware signature bypass
- Not yet publicly confirmed as working in production

**Recommended Use Cases by User Type**

_Researcher running physics simulations:_ Excellent choice. FDTD, LBM, and other memory-bound PDE solvers run near A100 performance with the FMA patch applied. The bandwidth-per-dollar ratio is unmatched.

_Developer running local LLM inference:_ Viable for 3B–8B models. Token generation speed is competitive. Prompt processing is slower than consumer cards but acceptable for non-interactive or batch use.

_ML engineer doing model training:_ Not recommended. The FMA throttle makes training loops impractically slow. An RTX 3090 or 4090 is a far better choice.

_Hobbyist experimenting with AI on a budget:_ Good option if you understand the constraints. You get exceptional memory bandwidth, decent FP16 inference, and a fascinating research platform — but you need Linux, a second GPU for display, and patience with software setup.
