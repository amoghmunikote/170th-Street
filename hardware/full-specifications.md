# Full Specifications

This page documents every known specification of the NVIDIA CMP 170HX in one place. Where figures differ between sources, the most accurate measured values are used.

<figure><img src="https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-teardown-0.jpg" alt=""><figcaption></figcaption></figure>

**General**

| Property              | Value                                     |
| --------------------- | ----------------------------------------- |
| Model                 | NVIDIA CMP 170HX                          |
| GPU Die               | GA100-105F-A1                             |
| Architecture          | NVIDIA Ampere                             |
| Manufacturing Process | TSMC 7nm N7 FinFET                        |
| Transistors           | 54.2 billion                              |
| Die Size              | 826 mm²                                   |
| Release Date          | September 1, 2021                         |
| Original MSRP         | $4,299 USD                                |
| Current Used Price    | $200–400 USD                              |
| Part Number           | 900-11001-0105-000                        |
| Form Factor           | Dual-slot PCIe add-in card                |
| Cooling               | Passive (server chassis airflow required) |
| Display Outputs       | None                                      |

**Compute**

| Property                  | Value                          |
| ------------------------- | ------------------------------ |
| Streaming Multiprocessors | 70                             |
| CUDA Cores                | 4,480 (64 per SM)              |
| Tensor Cores              | 280 (4 per SM, 3rd generation) |
| Texture Units (TMUs)      | 280                            |
| ROPs                      | 128                            |
| CUDA Compute Capability   | 8.0                            |
| Base Clock                | 1,140 MHz                      |
| Boost Clock               | 1,410 MHz                      |
| Max Clock (observed)      | 1,695 MHz                      |
| L1 Cache                  | 192 KB per SM                  |
| L2 Cache                  | 32,768 KB (32 MB)              |

**Memory**

| Property                       | Value                              |
| ------------------------------ | ---------------------------------- |
| Memory Type                    | HBM2e                              |
| Memory Capacity                | 8 GB (10GB and 16GB variants exist) |
| Memory Bus Width               | 4,096-bit (8GB variant); 5,120-bit (10GB variant) |
| Memory Clock                   | 1,458 MHz (729 MHz effective base) |
| Memory Bandwidth (theoretical) | 1,493 GB/s (8GB); ~1,865 GB/s (10GB estimated) |
| Memory Bandwidth (measured)    | \~1,355 GB/s (8GB, real-world clpeak) |
| Memory Stacks                  | 2 HBM2e stacks (8GB); 5 HBM2e stacks (10GB variant) |
| ECC                            | Disabled                           |
| Resizable BAR                  | Present but limited to 64 MiB      |

**Performance (Theoretical vs Actual)**

| Metric                        | Theoretical  | Actual (measured)           |
| ----------------------------- | ------------ | --------------------------- |
| FP32 (with FMA)               | 25.27 TFLOPS | **0.39 TFLOPS** (throttled) |
| FP32 (without FMA)            | —            | 6.25 TFLOPS                 |
| FP16                          | \~42 TFLOPS  | \~42 TFLOPS (unthrottled)   |
| FP64 (with FMA)               | —            | 0.18 TFLOPS (throttled)     |
| FP64 (without FMA)            | —            | 0.094 TFLOPS                |
| INT32                         | —            | 12.5 TIOPS (unthrottled)    |
| Tensor Core (TF32 via cuBLAS) | —            | \~6.2 TFLOPS                |
| Memory Bandwidth              | 1,493 GB/s   | \~1,355 GB/s                |

**Connectivity**

| Property                    | Value                                      |
| --------------------------- | ------------------------------------------ |
| PCIe Interface (physical)   | x16 slot                                   |
| PCIe Interface (electrical) | Gen 1 x4 (firmware locked)                 |
| PCIe Bandwidth              | \~1 GB/s (0.85 GB/s measured)              |
| NVLink                      | Physical connectors present, disabled via fuse  |
| Power Connector             | 1× 8-pin CPU power connector (via adapter) |

**Power**

| Property                             | Value      |
| ------------------------------------ | ---------- |
| TDP (default)                        | 250W       |
| Max Power Limit (software)           | 300W       |
| Idle Power (typical)                 | \~30–40W   |
| Load Power (FMA-throttled workload)  | \~75–100W  |
| Load Power (integer/memory workload) | \~160–180W |
| Load Power (non-FMA FP32, FluidX3D)  | \~180W     |

**Memory Latency**

| Property | Value | Notes |
|----------|-------|-------|
| Engineering Sample | ~75 ns | Measured on pre-production hardware |
| Production Card | ~152 ns | 2× latency vs engineering samples |
| Root Cause | 2-Hi HBM2e stacking | Fewer activated rows, different page buffer geometry |
| Impact on Inference | Minimal | Transformers are bandwidth-bound (streaming weights sequentially), not random-access bound |

**API Support**

| API           | Supported                      |
| ------------- | ------------------------------ |
| CUDA          | ✅ Yes (Compute Capability 8.0) |
| OpenCL        | ✅ Yes (3.0)                    |
| DirectX       | ❌ No                           |
| Vulkan        | ❌ No                           |
| OpenGL        | ❌ No                           |
| NvEnc / NvDec | ❌ No                           |

<figure><img src="https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-pcb-1.jpg" alt=""><figcaption></figcaption></figure>

**Notes on Specification Discrepancies**

The CMP 170HX exists in multiple memory configurations: **8GB, 10GB, and 16GB variants are confirmed real production cards**, not database errors. The 8GB variant was the original NVIDIA 2021 release. 10GB variants have been verified in circulation with identical physical layout. 16GB is theoretically achievable through VBIOS modifications, with some evidence suggesting different die configurations may support it (similar to A100 40GB/80GB variants). All variants use the same 4,096-bit HBM2e bus. Memory capacity is controlled via GSP firmware in the VBIOS, not physical hardware limitations. Always verify actual capacity using `nvidia-smi` on your specific card.
