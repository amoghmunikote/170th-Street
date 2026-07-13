# Sources

### Sources

A comprehensive reference list of all sources, tools, papers, and community resources referenced throughout this documentation.

***

**Primary Research Articles**

**niconiconi — "All GB/s without FLOPS: Nvidia CMP 170HX Review, Performance Lockdown Workaround, Teardown, Watercooling, and Repair"** The foundational article on the CMP 170HX. Covers performance benchmarking, FMA throttle discovery, PCB analysis, teardown, watercooling installation, and repair log. The single most comprehensive public resource on this card. [https://niconiconi.neocities.org/tech-notes/nvidia-cmp-170hx-review/](https://niconiconi.neocities.org/tech-notes/nvidia-cmp-170hx-review/)

**Xing Kangwei — "Exploration of Cryptocurrency Mining-Specific GPUs in AI Applications: A Case Study of CMP 170HX" (arXiv 2505.03782)** Peer-reviewed academic paper confirming the FMA workaround, providing llama-bench LLM inference results, energy efficiency analysis, and theoretical analysis of further improvements. Published April 2025. [https://arxiv.org/abs/2505.03782](https://arxiv.org/abs/2505.03782)

**Hackaday — "Hacking An NVIDIA CMP 170HX Crypto GPU For EM Sim Work"** Feature article covering niconiconi's work, published September 2024. Good entry point for newcomers. [https://hackaday.com/2024/09/11/hacking-an-nvidia-cmp-170hx-crypto-gpu-for-em-sim-work/](https://hackaday.com/2024/09/11/hacking-an-nvidia-cmp-170hx-crypto-gpu-for-em-sim-work/)

**Xing Kangwei — "Microbenchmarking Instruction-Level Tensor Core Throttling in NVIDIA CMP 170HX" (Zenodo 19002983)** Definitive microbenchmarking study characterizing the Tensor Core throttle mechanism. Discovers the 256 fixed-cycle MMA latency and 4-warp-per-SM dispatch limit, constructing a theoretical model that explains the observed 6.3 TFLOPS ceiling. Confirms the throttle is dispatch-level hardware gating, not physical damage. Published March 2026. [https://zenodo.org/records/19002983](https://zenodo.org/records/19002983)

**Xing Kangwei — "Instruction-Level Performance Analysis and Optimization Strategies for Constrained AI Accelerators: A Case Study of CMP 170HX" (Zenodo 18994970)** Proposes a systematic instruction-level framework for analyzing and bypassing restrictions on the CMP 170HX across three dimensions: instruction type, numerical precision, and execution unit. Introduces four operator-level circumvention strategies and validates them on Transformer inference, Stable Diffusion, and Diffusion Transformer workloads, achieving up to \~2× end-to-end improvement. Published March 2026. [https://zenodo.org/records/18994970](https://zenodo.org/records/18994970)

***

**Community Research & Tools**

**dartraiden/NVIDIA-patcher (GitHub)** Driver patcher adding 3D acceleration support for CMP series mining cards including the CMP 170HX. Active issue tracker is one of the best sources of community knowledge on driver compatibility and card behavior. [https://github.com/dartraiden/NVIDIA-patcher](https://github.com/dartraiden/NVIDIA-patcher)

**Issue #73 — "170hx can run higher FP32 flops than before" (dartraiden/NVIDIA-patcher)** The original December 2023 community discovery of the FMA workaround via OpenCL kernel modification. The two-line code diff that started everything. [https://github.com/dartraiden/NVIDIA-patcher/issues/73](https://github.com/dartraiden/NVIDIA-patcher/issues/73)

**ProjectPhysX/FluidX3D (GitHub)** Lattice Boltzmann Method CFD solver in OpenCL by Dr. Moritz Lehmann. The card is officially listed in the FluidX3D benchmark table. The no-FMA modification that unlocked 7,684 MLUPs/s originated here. [https://github.com/ProjectPhysX/FluidX3D](https://github.com/ProjectPhysX/FluidX3D)

**OMGVflash — TechPowerUp Forum Thread (Veii)** Development history and documentation of the Turing VBIOS signature bypass. Essential reading for understanding what has and hasn't been achieved in Ampere firmware security research. [https://www.techpowerup.com/forums/threads/omg-vflash-fully-patched-nvflash-from-x-to-ada-lovelace-v5-780.312601/](https://www.techpowerup.com/forums/threads/omg-vflash-fully-patched-nvflash-from-x-to-ada-lovelace-v5-780.312601/)

**NVflashk (GitHub — notfromstatefarm)** Modified nvflash with permanent mismatch bypass. Used for cross-flashing signed VBIOSes between compatible NVIDIA cards. [https://github.com/notfromstatefarm/nvflashk](https://github.com/notfromstatefarm/nvflashk)

**CAmadeus/falcon-tools (GitHub)** Falcon microcode analysis toolkit based on envydis. Used for disassembling Falcon ucode blobs extracted from VBIOS ROM images, including the DEVINIT scripts and the register `0x14118f78` bit-manipulation sequences documented in the PCIe unlock attempt. [https://github.com/CAmadeus/falcon-tools](https://github.com/CAmadeus/falcon-tools)

**hexkyz — Falcon Security Research Blog** Deep-dive reverse engineering work on NVIDIA Falcon security for Maxwell and Pascal generations. Covers the AES-CMAC key-derivation weakness in Falcon v5 that was exploited in the Thog/vbe0201 "requiem" fake-signing tools — the same class of vulnerability that NVIDIA fixed in Falcon v6+, which Ampere uses. Contextualizes why Maxwell-era Falcon attacks do not transfer to Ampere. [https://hexkyz.blogspot.com/2021/11/](https://hexkyz.blogspot.com/2021/11/)

**Overclock.net — NVflashk Thread (notfromstatefarm)** Primary community discussion thread for nvflashk development. Pages 7 and 10 contain Kefinator's explanation of the "backdoor" bypass mechanism, the complete list of checks the `-6` flag overrides, and the test matrix documenting which cross-generation flash attempts fail at POST vs. boot. [https://www.overclock.net/threads/nvflashk-flash-any-bios-to-nvidia-gpus-safe-board-id-bypass-up-to-4xxx-series-including-founders-edition-cards.1807438/](https://www.overclock.net/threads/nvflashk-flash-any-bios-to-nvidia-gpus-safe-board-id-bypass-up-to-4xxx-series-including-founders-edition-cards.1807438/)

**Overclock.net — "NVIDIA Security Chip Cracked" Thread** Community discussion of Ampere VBIOS security following the OMGVflash release. Contains Veii's direct statements on Ampere certificate chain validation, the "pretty pretty impossible" assessment of going around it without Falcon reprogramming, and Igor's Lab corroboration that gupsterg and Veii "ultimately had to conclude that penetrating the current security mechanisms is very difficult." [https://www.overclock.net/threads/nvidia-security-chip-on-modern-nvidia-video-cards-cracked-modified-and-custom-bioses-allowed-soon.1807471/](https://www.overclock.net/threads/nvidia-security-chip-on-modern-nvidia-video-cards-cracked-modified-and-custom-bioses-allowed-soon.1807471/)

**jonpry/sass\_fma (GitHub)** SASS-level FMA instruction replacement research. Explores low-level binary patching approaches to bypass FMA throttling through instruction substitution. Ongoing research into whether SASS binary patches can overcome the 4-instruction control flow spacing challenge. [https://github.com/jonpry/sass_fma](https://github.com/jonpry/sass_fma)

**theneocorp/cmppatcher (GitHub)** Tools for modifying CMP BIOS configurations and FMA bypass implementations for CMP 90HX (Turing architecture). Branch `cmp-90hx-fma-bypass` contains working FMA bypass for Turing-based CMPs. Reference for understanding strap configuration and VBIOS modification techniques. [https://github.com/theneocorp/cmppatcher](https://github.com/theneocorp/cmppatcher)

**ggml-org/llama.cpp DP2A Patch** Pull request implementing DP2A (2-lane dot product) quantization optimization, achieving approximately 2× end-to-end speedup versus DP4A during decode phase. Critical reference for understanding the performance impact of quantization method choice on CMP 170HX. [https://github.com/ggml-org/llama.cpp/issues/24616](https://github.com/ggml-org/llama.cpp/issues/24616)

**duggasco/bc250-40cu-unlock** Community reverse-engineering work on BC-250 GPU unlocking, documenting VBIOS strap configuration analysis techniques and memory configuration extraction methodologies applicable to 170HX research. [https://github.com/duggasco/bc250-40cu-unlock](https://github.com/duggasco/bc250-40cu-unlock)

**VBIOS Comparison Analysis Gist** Python tools and detailed analysis for extracting and comparing VBIOS versions, identifying strap configurations, and understanding memory controller settings across different NVIDIA cards. [https://gist.github.com/duggasco/819f65d599be39c148b8fcaea42c4d8d](https://gist.github.com/duggasco/819f65d599be39c148b8fcaea42c4d8d)

***

**VBIOS References & Tools**

**TechPowerUp VBIOS Database — CMP Series** Repository of signed VBIOS firmware files for CMP cards. Key variants referenced:
- CMP 50HX: File 262191 (Turing TU102 architecture)
- CMP 170HX variants: Files 267478, 268495, 282151 (different memory configurations)
Authoritative source for understanding VBIOS device ID structures and comparing firmware versions. [https://www.techpowerup.com/vgabios/](https://www.techpowerup.com/vgabios/)

**TechPowerUp VBIOS Viewer (ImHex RTX40x0/Kepler/Ada Pattern)** Reference implementation for VBIOS structure parsing and visualization. Provides patterns and methodology for analyzing VBIOS files in ImHex hex editor. [https://www.techpowerup.com/forums/threads/rtx40x0-vbios-viewer-kepler-ada-nvidia-vbios-visualizer-opensource-pattern-for-imhex.322299/](https://www.techpowerup.com/forums/threads/rtx40x0-vbios-viewer-kepler-ada-nvidia-vbios-visualizer-opensource-pattern-for-imhex.322299/)

***

**Hardware Documentation**

**NVIDIA Tesla A100 GA100-883 P1001-B02 PCB Schematics** Leaked NVIDIA reference schematics for the A100 40GB PCIe. Every component reference designator matches the CMP 170HX PCB exactly. Used for power tree analysis, repair, PCIe capacitor identification, and NVLink research throughout this documentation. Circulates in electronics repair communities — search for "NVIDIA Tesla A100 schematic GA100-883" to find.

**NVIDIA A100 Tensor Core GPU Architecture Whitepaper** Official NVIDIA documentation of the GA100 architecture, SM layout, memory subsystem, and compute capabilities. Authoritative reference for theoretical performance numbers. [https://images.nvidia.com/aem-dam/en-zz/Solutions/data-center/nvidia-ampere-architecture-whitepaper.pdf](https://images.nvidia.com/aem-dam/en-zz/Solutions/data-center/nvidia-ampere-architecture-whitepaper.pdf)

**NVIDIA Ampere Architecture In-Depth (NVIDIA Technical Blog)** Detailed developer-facing explanation of GA100 architecture features including Tensor Cores, NVLink, and SM design. [https://developer.nvidia.com/blog/nvidia-ampere-architecture-in-depth/](https://developer.nvidia.com/blog/nvidia-ampere-architecture-in-depth/)

**NVIDIA Falcon Security Documentation** NVIDIA's official (partial) documentation of the Falcon security processor and its operating modes. Essential for understanding the firmware signing architecture. [https://download.nvidia.com/open-gpu-doc/Falcon-Security/1/Falcon-Security.html](https://download.nvidia.com/open-gpu-doc/Falcon-Security/1/Falcon-Security.html)

**NVIDIA open-gpu-doc (GitHub)** NVIDIA's partial hardware documentation release. Contains register specifications and architectural details useful for driver and firmware research. [https://github.com/NVIDIA/open-gpu-doc](https://github.com/NVIDIA/open-gpu-doc)

**NVIDIA Open-Source Kernel Modules (GitHub)** Partially open-sourced NVIDIA Linux kernel driver, released 2022. Useful for analysis of driver-level behavior including any per-device restrictions. [https://github.com/NVIDIA/open-gpu-kernel-modules](https://github.com/NVIDIA/open-gpu-kernel-modules)

**Linux nova-core — Falcon Architecture Documentation** Kernel documentation for the nova-core driver's Falcon implementation. Covers NS/LS/HS mode transitions, the HS bootrom authentication flow, and DMA-time signature validation on Ampere. Authoritative source for the "verified during DMA transfer, not after" behavior that closes the RAM-patching TOCTOU window. [https://docs.kernel.org/gpu/nova/core/falcon.html](https://docs.kernel.org/gpu/nova/core/falcon.html)

**Linux nova-core — FWSEC Documentation** Kernel documentation for FWSEC (Firmware Security) subsystem. Describes WPR2 creation, FRTS/SB command interface exposed via FalconAppifHdrV1, and the authenticated copy of VBIOS sections into the write-protected region. [https://docs.kernel.org/gpu/nova/core/fwsec.html](https://docs.kernel.org/gpu/nova/core/fwsec.html)

**Linux nova-core — VBIOS Documentation** Kernel documentation covering the VBIOS ROM layout: Type 0x00 PciAt, Type 0x03 EFI GOP, and Type 0xE0 FwSec partitions. Describes the BIT (BIOS Info Table) structure and token 0x70 (BIT\_TOKEN\_ID\_FALCON\_DATA). [https://docs.kernel.org/gpu/nova/core/vbios.html](https://docs.kernel.org/gpu/nova/core/vbios.html)

**Timur Tabi — FalconUCodeDescV2 Support Patch Series (Linux nouveau mailing list)** Patch series from late 2025/early 2026 that revealed GA100 and Turing use FalconUCodeDescV2 rather than V3. V2 returns zero for pkc\_data\_offset, engine\_id\_mask, ucode\_id, signature\_count, and signature\_versions, and uses a split imem\_sec\_base/imem\_sec\_size layout. The nova-core comment "Only used on Turing and GA100, so return None for now" originates here. Critical for understanding why the CMP 170HX's signing infrastructure differs from consumer Ampere. Entry point: [https://www.mail-archive.com/nouveau@lists.freedesktop.org/msg46482.html](https://www.mail-archive.com/nouveau@lists.freedesktop.org/msg46482.html)

**Joel Fernandes — nova-core GSP/SEC2 Patch (patchew.org)** Patch to the nova-core driver adding GSP and SEC2 Falcon support, part of the broader series documenting the Ampere boot chain (Booter → GSP-RM → FWSEC → DEVINIT → SEC2-RTOS). [https://patchew.org/linux/20250503040802.1411285-1-joelagnelf@nvidia.com/20250503040802.1411285-7-joelagnelf@nvidia.com/](https://patchew.org/linux/20250503040802.1411285-1-joelagnelf@nvidia.com/20250503040802.1411285-7-joelagnelf@nvidia.com/)

***

**Tools**

**nvflash 5.867** NVIDIA's official VBIOS flash utility for Linux and Windows. Used for backup and flashing of signed VBIOSes. [https://www.techpowerup.com/download/nvidia-nvflash/](https://www.techpowerup.com/download/nvidia-nvflash/)

**TechPowerUp VBIOS Database — CMP 170HX** Repository of known NVIDIA VBIOS files. The authoritative public source for CMP 170HX firmware versions with verified subsystem IDs. [https://www.techpowerup.com/vgabios/?type=\&model=CMP+170HX](https://www.techpowerup.com/vgabios/?type=\&model=CMP+170HX)

**clpeak** OpenCL benchmark measuring peak compute throughput and memory bandwidth. Primary tool for characterizing the FMA throttle and real-world memory bandwidth. [https://github.com/krrishnarraj/clpeak](https://github.com/krrishnarraj/clpeak)

**mixbench** GPU benchmark for evaluating performance at mixed operational intensities. Used to measure FP16 throughput (\~42 TFLOPS) on the CMP 170HX. [https://github.com/ekondis/mixbench](https://github.com/ekondis/mixbench)

**gpu\_burn** GPU stress testing tool. Used for Tensor Core testing via cuBLAS and thermal stability verification. [https://github.com/wilicc/gpu-burn](https://github.com/wilicc/gpu-burn)

**llama.cpp (ggml-org)** C/C++ LLM inference engine with CUDA support. Primary recommended framework for LLM inference on the CMP 170HX. Build with `-DCMAKE_CUDA_FLAGS="-fmad=false"`. [https://github.com/ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp)

**hashcat** GPU-accelerated password recovery tool. Used as an integer workload benchmark confirming unthrottled INT32 performance at 43,930 MH/s MD5. [https://hashcat.net/hashcat/](https://hashcat.net/hashcat/)

***

**Watercooling & Hardware**

**Bykski N-TESLA-A100-X-V2 Product Page (Bykski.us)** Official US distributor page for the only commercially available full-coverage waterblock compatible with the CMP 170HX. Note: AliExpress is significantly cheaper. [https://www.bykski.us/products/bykski-metal-pom-gpu-water-block-and-backplate-for-nvidia-tesla-a100-40gb](https://www.bykski.us/products/bykski-metal-pom-gpu-water-block-and-backplate-for-nvidia-tesla-a100-40gb)

**Bykski N-TESLA-A100-X-V2 (FormulaMod)** Alternative retailer with international shipping. [https://www.formulamod.com/Bykski-GPU-Block-For-Nvidia-Tesla-A100-40GB-Nvidia-CMP-170HX-Nvidia-Tesla-A30-24G-High-Heat-Resistance-Material-POM-Full-Metal-Construction-With-Backplate-Full-Cover-GPU-Water-Cooling-Cooler-Radiator-Block-N-TESLA-A100-X-V2-p3765067.html](https://www.formulamod.com/Bykski-GPU-Block-For-Nvidia-Tesla-A100-40GB-Nvidia-CMP-170HX-Nvidia-Tesla-A30-24G-High-Heat-Resistance-Material-POM-Full-Metal-Construction-With-Backplate-Full-Cover-GPU-Water-Cooling-Cooler-Radiator-Block-N-TESLA-A100-X-V2-p3765067.html)

***

**Hardware Modifications & Guides**

**PCIe Capacitor Restoration Demonstration (YouTube)** Visual walkthrough of soldering missing PCIe coupling capacitors to restore x16 lane support. Documents the exact capacitor locations and soldering technique. Confirms restoration from x4 to x16 functionality via standard VBIOS without firmware modification. [https://www.youtube.com/watch?v=saPvIIPP1cE](https://www.youtube.com/watch?v=saPvIIPP1cE)

**Fan Cooling Adapter Approach** Alternative to expensive water cooling solutions. 120mm CPU-style fans with high CFM sufficient for thermal management. Documented by community users as viable thermal solution for passive-by-default card.

**RainCandy Driver Work** Community-maintained GPU driver modifications. [https://raincandy.tech/nvidiadrv/](https://raincandy.tech/nvidiadrv/)

***

**Media**

**Linus Tech Tips — "We Spent $5000 on a Crypto Mining GPU for Science…" (YouTube)** First major public coverage of the CMP 170HX. November 2021. Confirms early hardware details and the mining-only restriction at launch. [https://www.youtube.com/watch?v=vfVnxkj8p6I](https://www.youtube.com/watch?v=vfVnxkj8p6I)

**Chinese teardown video — 英伟达CMP 170HX显卡拆解 (YouTube)** Visual teardown guide. Essential reference for the difficult Step 7 board removal procedure. [https://www.youtube.com/watch?v=N1yhvXj\_Do8](https://www.youtube.com/watch?v=N1yhvXj_Do8)

**Second teardown video — 技数犬 CMP 170HX 拆机测试 (YouTube)** Alternative teardown with captions (auto-translatable to English). [https://www.youtube.com/watch?v=kaiB7R4mhTg](https://www.youtube.com/watch?v=kaiB7R4mhTg)

**Other Hardware Modification Videos (YouTube)** Various community demonstrations of cooling solutions, capacitor soldering, and component identification. Search for "CMP 170HX modification" on YouTube for current collection.

***

**Firmware & Security Research Documentation**

**NVIDIA BIOS Information Table (BIT) Specification** Official NVIDIA documentation of the VBIOS BIT (BIOS Information Table) structure. Essential for understanding VBIOS section organization and configuration token meanings. [https://download.nvidia.com/open-gpu-doc/BIOS-Information-Table/1/BIOS-Information-Table.html](https://download.nvidia.com/open-gpu-doc/BIOS-Information-Table/1/BIOS-Information-Table.html)

**Linux nova-core VBIOS Documentation** Kernel documentation covering VBIOS ROM layout, FwSec partitions, and type/token structures. Describes how FwSec sections are identified and loaded. [https://docs.kernel.org/gpu/nova/core/vbios.html](https://docs.kernel.org/gpu/nova/core/vbios.html)

**NVIDIA Falcon Security (Official Documentation)** Official NVIDIA documentation of the Falcon security processor architecture. Essential for understanding firmware authentication and privileged mode restrictions. [https://nvidia.github.io/open-gpu-doc/Falcon-Security/Falcon-Security.html](https://nvidia.github.io/open-gpu-doc/Falcon-Security/Falcon-Security.html)

**GPU Security Breach Analysis** Technical analysis of GPU security vulnerability vectors and attack methodologies. Reference for understanding potential exploit paths and security mechanisms. [https://hackingpassion.com/gpubreach-attack-nvidia-gpu/](https://hackingpassion.com/gpubreach-attack-nvidia-gpu/)

***

**International Resources**

**Habr.com — CMP Modifications (Russian Tech Community)** Russian-language technical forum articles on CMP GPU modifications, fan behavior documentation, and community experiences. Accessible to international audience via Google Translate. [https://habr.com/ru/articles/940226/](https://habr.com/ru/articles/940226/)

**NVIDIA Developer Forums — FBPA Register Discussion** Official NVIDIA developer forum discussion on register permission levels and write accessibility. [https://forums.developer.nvidia.com/t/the-difference-between-fbp-and-fbpa/367237](https://forums.developer.nvidia.com/t/the-difference-between-fbp-and-fbpa/367237)

***

**Image Sources**

All hardware photos unless otherwise noted are sourced from niconiconi's article. Direct image URLs used throughout this documentation:

| Image                      | URL                                                                                                                                                                                |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PCB front                  | [https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-pcb-1.jpg](https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-pcb-1.jpg)               |
| PCB back                   | [https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-pcb-2.jpg](https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-pcb-2.jpg)               |
| PCIe missing caps          | [https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-pcb-3.jpg](https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-pcb-3.jpg)               |
| Chassis exterior           | [https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-teardown-0.jpg](https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-teardown-0.jpg)     |
| Teardown step 1            | [https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-teardown-1.jpg](https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-teardown-1.jpg)     |
| Teardown step 2            | [https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-teardown-2.jpg](https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-teardown-2.jpg)     |
| Teardown step 3            | [https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-teardown-3.jpg](https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-teardown-3.jpg)     |
| Power topology schematic   | [https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-schematics-1.png](https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-schematics-1.png) |
| Power sequencing schematic | [https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-schematics-2.png](https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-schematics-2.png) |
| Repair photo 1             | [https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-repair-1.jpg](https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-repair-1.jpg)         |
| Repair photo 2             | [https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-repair-6.jpg](https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-repair-6.jpg)         |
| Watercooling step 1        | [https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-watercool-1.jpg](https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-watercool-1.jpg)   |
| Watercooling step 2        | [https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-watercool-2.jpg](https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-watercool-2.jpg)   |
| Watercooling complete      | [https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-watercool-3.jpg](https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-watercool-3.jpg)   |
| Roofline model SVG         | [https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/roofline-cmp170hx.svg](https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/roofline-cmp170hx.svg)           |

***

**Further Reading**

**Tom's Hardware — "New NVIDIA BIOS Modding Tools"** News coverage of OMGVflash and NVflashk, with technical commentary on what the bypass achieves vs. what remains blocked. Good secondary source for cross-flash tool capabilities and community reaction. [https://www.tomshardware.com/news/new-nvidia-bios-modding-tools](https://www.tomshardware.com/news/new-nvidia-bios-modding-tools)

**VideoCardz — "NVIDIA GPU BIOS Flashing Tools Can Now Flash Any BIOS to Any GPU"** News coverage of the nvflashk/OMGVflash release cycle. Useful contemporaneous record of what the community understood was achieved. [https://videocardz.com/newz/nvidia-gpu-bios-flashing-tools-can-now-flash-any-bios-to-any-gpu-bypassing-restrictions](https://videocardz.com/newz/nvidia-gpu-bios-flashing-tools-can-now-flash-any-bios-to-any-gpu-bypassing-restrictions)

**repair.wiki — NVIDIA Pascal GPU Diagnosing Guide** General GPU repair methodology. The diagnostic approach applies broadly beyond Pascal. [https://repair.wiki/w/Nvidia\_Pascal\_GPU\_Diagnosing\_Guide](https://repair.wiki/w/Nvidia_Pascal_GPU_Diagnosing_Guide)

**VideoCardz — "NVIDIA CMP 170HX mining card with GA100 GPU has a massive heatspreader"** Early hardware coverage with additional teardown details. November 2021. [https://videocardz.com/newz/nvidia-cmp-170hx-mining-card-with-ga100-gpu-has-a-massive-heatspreader](https://videocardz.com/newz/nvidia-cmp-170hx-mining-card-with-ga100-gpu-has-a-massive-heatspreader)

**TechPowerUp GPU Database — CMP 170HX** Spec database entry. Note documented inaccuracies in the memory size field — real card is 8GB HBM2e. [https://www.techpowerup.com/gpu-specs/cmp-170hx.c3840](https://www.techpowerup.com/gpu-specs/cmp-170hx.c3840)
