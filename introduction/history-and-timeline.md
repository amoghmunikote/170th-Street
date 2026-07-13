# History & Timeline

**February 18, 2021 — Official CMP Announcement** NVIDIA publishes a blog post announcing the CMP (Cryptocurrency Mining Processor) product line. Four models are revealed: the 30HX, 40HX, 50HX, and 90HX — all based on existing Turing and lower-end Ampere silicon and manufactured by board partners. The CMP 170HX is not mentioned. NVIDIA's stated reason: "GeForce is made for gaming. CMP is made for mining."

**March 2021 — First Leak** GPU shortage speculation leads to the first credible reports of a dedicated mining GPU built on GA100 silicon. Leaker Kopite7kimi reports NVIDIA is developing a top-end CMP card based on Ampere A100 technology. No official confirmation from NVIDIA.

**September 1, 2021 — CMP 170HX Appears in China** The CMP 170HX surfaces on the Chinese market with no official announcement from NVIDIA. A user on Zhihu (Chinese Q\&A platform) posts the first real-world photographs and benchmark results, showing 164 MH/s on Ethash at 250W. The card is priced at approximately $4,700 USD. Tech outlets notice immediately — this is GA100 silicon, the same die as the A100. The card connects via a single 8-pin connector and is completely passive-cooled.

**November 25, 2021 — Linus Tech Tips Video** Linus Sebastian of Linus Tech Tips purchases a CMP 170HX for $4,700 and publishes a video titled "This $5000 Graphics Card Can't Game." The video reaches over 4.5 million views and is the first major Western coverage of the card. Linus confirms the card uses the GA100-105F GPU, documents the massive copper heat spreader covering the entire interposer area, and notes that no official drivers or software recognize the card. The video establishes that despite listing CUDA capability, the card cannot render in Blender and has no gaming API support. TechPowerUp and VideoCardz.com publish articles documenting the discovery.

{% embed url="https://www.youtube.com/watch?t=707s&v=EcGkF9SBuSo" %}

**2021–2022 — Mining Peak** The CMP 170HX sells at high prices to Ethereum mining farms, primarily in China. At peak ETH prices and electricity rates, miners calculated approximately a 3–6 month payback period. Mining farms rack the cards in server chassis with forced-air cooling, running them 24/7.

**September 15, 2022 — The Merge** Ethereum completes its transition from Proof-of-Work to Proof-of-Stake consensus, known as "The Merge." GPU-based Ethereum mining becomes impossible overnight. Mining farms begin liquidating hardware immediately. Cards originally sold for $4,000–5,000 begin appearing on Chinese secondhand markets for $400–500. Western eBay listings follow at $300–500.

**2022–2023 — Community Exploration Begins** With prices collapsed, researchers and hobbyists begin acquiring CMP 170HX cards and investigating their compute capabilities. Initial attempts to use them for CUDA workloads confirm the severe FP32 throttling. Community members on GitHub and forums begin documenting the behavior and searching for workarounds.

**October 25, 2023 — niconiconi's Definitive Article** A researcher and electronics hobbyist writing as niconiconi publishes "All GB/s without FLOPS — Nvidia CMP 170HX Review, Performance Lockdown Workaround, Teardown, Watercooling, and Repair" on their personal site. This becomes the most comprehensive technical resource on the card to date. The article documents: full performance benchmarks across all precision types, the FMA throttle mechanism, a complete hardware teardown confirming PCB identity to the A100 40GB PCIe, the two-layer PCIe lockdown, a watercooling installation guide using the Bykski N-TESLA-A100-X-V2 waterblock, and a detailed repair log using leaked A100 schematics. The article is featured on Hackaday and Hacker News, reaching a wide technical audience.

{% embed url="https://niconiconi.neocities.org/tech-notes/nvidia-cmp-170hx-review/" %}

**December 2023 — Community FMA Workaround Confirmed** A GitHub user opens issue #73 on dartraiden/NVIDIA-patcher reporting that modifying OpenCL benchmarks to disable FMA instructions using `#pragma OPENCL FP_CONTRACT OFF` raises FP32 performance from 0.395 TFLOPS to 6.285 TFLOPS — a 16× improvement. This independently confirms the throttling is at the instruction level and not in hardware, and establishes that the workaround documented by niconiconi is reproducible. The dartraiden/NVIDIA-patcher project, which had already been providing 3D acceleration patches for CMP cards on Windows, is confirmed to support the CMP 170HX.

**September 11, 2024 — Hackaday Feature** Hackaday publishes a feature article on niconiconi's CMP 170HX work, expanding awareness of the card's potential in the electronics and hardware hacking community. The article emphasizes the PCIe lane capacitor mod, the FMA throttle, and the use of the GPU for electromagnetic field simulations.

**April 30, 2025 — Academic Paper Published** Researcher Xing Kangwei of Jiangsu University publishes "Exploration of Cryptocurrency Mining-Specific GPUs in AI Applications: A Case Study of CMP 170HX" on arXiv (paper 2505.03782). This is the first peer-reviewed academic study specifically examining the CMP 170HX for AI workloads. The paper validates the FMA workaround, reports FP32 performance exceeding 15× original capability after modification, documents LLM inference benchmarks using llama-bench showing over 3× improvement at certain precision levels, and evaluates the card's viability for edge computing and lightweight AI inference. The paper explicitly frames the CMP 170HX as a potential solution to GPU shortages caused by US export controls on high-performance GPUs to regions like mainland China.

{% embed url="https://arxiv.org/abs/2505.03782" %}

**2025 — Present** The 170th Street community forms around the goal of fully unlocking the CMP 170HX's capabilities. Cards are now available on eBay for $200–400 USD. Research into firmware signing and the path to a full FMA unlock continues. The PCIe capacitor mod remains documented but unconfirmed on the 170HX specifically (confirmed on other CMP HX models). The arXiv paper establishes academic credibility for the use case.
