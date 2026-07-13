# FP16 Performance

**FP16 Performance**

FP16 (half-precision floating-point) is the second most important compute capability of the CMP 170HX, after memory bandwidth. At approximately 42 TFLOPS scalar (non-tensor) performance, it is unthrottled and represents the primary path for AI inference workloads on this card.

**Scalar vs Tensor FP16 — A Critical Distinction for Inference**

For LLM token generation at batch size 1 (the standard interactive inference scenario), **only scalar FP16 performance matters**. Tensor Cores are not utilized in this regime. This is why the CMP 170HX's scalar FP16 performance directly compares to other GPUs' scalar performance, not their tensor peak performance.

The CMP 170HX has a unique architectural advantage: its FP16 scalar performance (hfma) is **4× its FP32 scalar performance** (fma). Most consumer NVIDIA GPUs artificially cap FP16 hfma to 1:1 with FP32 fma, leaving significant scalar compute unrealized.

**Scalar FP16 Performance Comparison (Token Generation)**

| GPU            | Scalar FP16 Performance | Notes                              |
| -------------- | ----------------------- | ---------------------------------- |
| B300 SXM6      | 75.2 TFLOPS             | Peak scalar (not tensor)           |
| A100 PCIe 40GB | ~50 TFLOPS scalar       | 4× advantage over FP32 (like 170HX)|
| RTX 4090       | 82.6 TFLOPS             | Scalar, peak boost clock           |
| **CMP 170HX**  | **~42 TFLOPS**          | **Locked state, scalar only**      |
| RTX 3090       | 40 TFLOPS               | Scalar, peak boost clock           |
| RTX 3080       | 29.5 TFLOPS             | Scalar, peak boost clock           |
| RTX 2080       | 28.5 TFLOPS             | Scalar, peak boost clock           |

**The 170HX is competitive with modern consumer cards in token generation performance.** With the no-fma and no-dp2a workarounds applied, the locked 170HX matches consumer card token-generation speeds for batch size 1 inference. The comparison to A100's tensor performance (312 TFLOPS) is irrelevant for single-request inference — tensor cores only benefit larger batch sizes or matrix multiply operations.

**mixbench Results**

```
FP16 (half precision, CUDA/SYCL):
  Peak throughput: ~41,869 GFLOPS (~42 TFLOPS) [scalar]
```

Note: FP16 is inaccessible via NVIDIA's OpenCL runtime on this card ("No half precision support" in clpeak). It is accessible via CUDA and SYCL.

**Why FP16 Matters for This Card**

FP16 is the dominant precision for modern AI inference, and the CMP 170HX's scalar FP16 performance is one of its genuine strengths:

* **Token generation performance is competitive with consumer GPUs** — ~42 TFLOPS scalar FP16 matches or beats many desktop cards
* **4× FP16 hfma vs FP32 fma** — Unlike consumer GPUs that artificially cap FP16 to 1:1, the 170HX maintains the full 4× advantage, making scalar FP16 the preferred path
* Most LLM inference frameworks default to FP16 weights
* FP16 uses half the VRAM of FP32, enabling larger models in the 8GB budget
* The memory bandwidth advantage of the CMP 170HX (1,355 GB/s) directly translates to proportional speedup for memory-bound decode workloads

The arXiv paper on the CMP 170HX (2505.03782) confirmed that LLM inference at FP16 precision shows over 3× improvement compared to FP32 inference on this card, because the FP16 path avoids the FMA throttle.

**Tensor Core Status**

The CMP 170HX has 280 third-generation Tensor Cores (4 per SM × 70 SMs). For a long time the community observed \~6.2–6.3 TFLOPS via the cuBLAS Tensor Core path and could not explain why — the same ceiling as non-FMA FP32, suggesting the Tensor Cores were either non-functional or capped by the same mechanism as FMA.

Microbenchmarking research by Xing Kangwei (Zenodo 19002983, March 2026) has now definitively answered this question with three findings:

**Finding 1 — 256 fixed-cycle instruction throttle on MMA instructions**

A single MMA (Matrix Multiply-Accumulate) instruction on the CMP 170HX executes with a fixed latency of 256 cycles. Critically, this latency cannot be hidden through pipeline overlap regardless of how much Instruction Level Parallelism (ILP) is present. On an unrestricted A100, MMA latency can be hidden by overlapping independent instructions in the pipeline. On the CMP 170HX, the 256-cycle stall is absolute — no amount of instruction scheduling helps.

**Finding 2 — Only 4 warps per SM can issue Tensor Core instructions simultaneously**

On the CMP 170HX, only 4 warps per Streaming Multiprocessor can simultaneously issue Tensor Core instructions. An unrestricted A100 allows far more concurrent warp issuance. This 4-warp limit combined with the 256-cycle fixed latency creates the observed throughput ceiling.

**Finding 3 — The theoretical model closes perfectly**

Working from the 256-cycle fixed latency and the 4-warp issue limit, the paper constructs a theoretical model that produces exactly the measured 6.3 TFLOPS — closing the loop from microarchitecture to observed benchmark result. The Tensor Core FP16 realistic computing power is **1/32 of its theoretical peak** as a direct result of these two constraints.

**The throttle is dispatch-level hardware gating**

Multiple controlled experiments — ILP scaling, warp scaling, dependency chain construction, and cross-pipeline interference — confirm the throttle is a **dispatch-level hardware gate**, not physical damage to execution units and not a decoding delay. The Tensor Core execution units are physically intact and functional. The gate simply prevents more than 4 warps from issuing MMA instructions per SM, and enforces the 256-cycle fixed latency regardless of pipeline state.

This is a distinct mechanism from the FP32 FMA throttle. The two restrictions operate at different levels and through different mechanisms — both are present simultaneously on the CMP 170HX.

**Practical implication:** The 6.3 TFLOPS Tensor Core ceiling is now understood to be a hardware-enforced dispatch gate rather than a firmware flag. This makes it significantly harder to bypass than the FMA throttle (which was bypassed at the compiler level by avoiding the instruction entirely). Bypassing the Tensor Core dispatch gate would require either firmware modification or a new hardware-level technique — both of which are unsolved.
