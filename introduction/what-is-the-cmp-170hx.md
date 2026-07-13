# What is the CMP 170HX?

{% embed url="https://www.youtube.com/watch?t=707s&v=EcGkF9SBuSo" %}

**What is the NVIDIA CMP 170HX?**

The NVIDIA CMP 170HX is one of the most unusual graphics cards ever made — a piece of hardware that sits at a strange intersection of cutting-edge silicon, deliberate sabotage, and untapped potential. On the surface it is a cryptocurrency mining card, purpose-built by NVIDIA in 2021 to mine Ethereum and nothing else. Underneath, it is built on the same GA100 die as the NVIDIA A100, one of the most powerful datacenter GPUs ever produced and a card that costs upward of $10,000.

**What it is at the silicon level**

The CMP 170HX uses the GA100-105F-A1 chip, a binned variant of the full GA100 die that powers the A100. It has 4,480 CUDA cores spread across 70 streaming multiprocessors, 280 tensor cores, and 8GB of HBM2e memory on a 4096-bit bus delivering 1,493 GB/s of memory bandwidth. That bandwidth figure is not a typo — it is nearly 1.5 times faster than the RTX 4090's memory system and essentially matches the A100 PCIe 40GB's 1,555 GB/s. The card runs at a base clock of 1,140 MHz and boosts to 1,410 MHz with a 250W TDP.

The PCB itself is nearly identical to the A100 40GB PCIe reference design. Every component reference designator on the board matches the leaked NVIDIA Tesla A100 electrical schematics exactly — NVIDIA was still calling it Tesla internally when it was being designed. The differences are omissions: fewer VRM phases, no NVLink ICs, no display outputs, and missing AC coupling capacitors on certain PCIe lanes.

<figure><img src="https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-pcb-1.jpg" alt=""><figcaption></figcaption></figure>

**What NVIDIA did to it**

To prevent the card from being used for general compute, NVIDIA implemented multiple layers of restriction:

The most significant is the **FP32 FMA throttle**. FMA (Fused Multiply-Add) is the instruction at the heart of virtually every modern compute and AI workload. NVIDIA throttled FMA throughput to approximately 0.39 TFLOPS — slower than a GPU from 2008, and slower than a modern multi-core CPU. The card's theoretical FP32 ceiling without this restriction would be around 25 TFLOPS, comparable to an A100.

The **PCIe interface** is restricted in two independent ways: a firmware-level lock to PCIe Gen 1 speed, and a hardware-level PCB modification where the AC coupling capacitors for 12 of the 16 PCIe lanes have been omitted, forcing a link downgrade to x4 width. The combined result is approximately 1 GB/s of PCIe bandwidth instead of the 64 GB/s a full PCIe 4.0 x16 connection would provide.

**VRAM** is limited to 8GB (though 10GB and 16GB variants exist) despite the A100 shipping with 40GB or 80GB. **NVLink** physical hardware is fully present but disabled via fuse-level security, preventing activation without specialized access keys. **Display outputs** are absent. And **VBIOS firmware signing** prevents modification of these restrictions through software alone.

**Why it exists**

NVIDIA launched the CMP line in early 2021 during the height of the GPU shortage crisis. Miners were buying every GeForce card available, leaving gamers with nothing and generating massive PR backlash for NVIDIA. The CMP line was NVIDIA's official response — dedicated mining hardware designed to be unattractive for gaming or resale.

The CMP 170HX is unique within the CMP line because it is the only model designed and manufactured directly by NVIDIA themselves. Every other CMP HX model was produced by board partners using existing consumer silicon. The 170HX used GA100 silicon that didn't meet the strict quality thresholds for A100 production — chips with too many defective cores or marginal performance bins that would otherwise be discarded. NVIDIA found a way to monetize these chips at $4,000–5,000 each by selling them to miners.

<figure><img src="https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-teardown-0.jpg" alt=""><figcaption></figcaption></figure>

**Why it's relevant now**

In September 2022, Ethereum completed "The Merge" — its long-planned transition from Proof-of-Work mining to Proof-of-Stake consensus. Overnight, every Ethereum mining GPU became obsolete for its intended purpose. The CMP 170HX, which had no use case other than Ethereum mining, flooded secondhand markets. Cards that originally sold for $4,000–5,000 dropped to $400–500 in China and $200–400 in Western markets.

This created an extraordinary situation: GA100 silicon with 1,493 GB/s of HBM2e memory bandwidth — hardware that in its A100 form costs $10,000+ — was suddenly available for the price of a budget GPU. Researchers, hobbyists, and engineers began asking whether the restrictions NVIDIA imposed could be worked around. The answer, it turns out, is partially yes — and the work to fully unlock this card is ongoing.
