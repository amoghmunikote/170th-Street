# PCB Analysis

**PCB Analysis**

The CMP 170HX PCB is one of the most interesting aspects of the card from a hardware research perspective. It is not merely similar to the A100 — it is the same board with components selectively omitted.

<figure><img src="https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-pcb-1.jpg" alt=""><figcaption></figcaption></figure>

<figure><img src="https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-pcb-2.jpg" alt=""><figcaption></figcaption></figure>

**PCB Identity**

The CMP 170HX uses a circuit board that is nearly — if not completely — identical to the NVIDIA A100 40GB PCIe reference design. This was definitively confirmed when a researcher used leaked NVIDIA Tesla A100 electrical schematics to diagnose and repair a broken CMP 170HX, finding that every component reference designator on the board matched the schematics exactly. The internal codename "Tesla" appears throughout the schematics, revealing the A100 was still being called by its legacy Tesla name during development.

The only difference at the die level is the GPU model number: GA100-105F-A1 instead of the full GA100-895 used in the A100. This reflects chip binning — the 105F variant has fewer active streaming multiprocessors than the full die.

**What Is Present**

Everything foundational to the A100 is on this board: the same PCIe connector with gold fingers for all 16 lanes, the same NVLink gold finger contacts along the top edge, the same power delivery architecture using the same power management ICs, the same reference designators in the same positions, and the same overall board dimensions and mounting points. This is why all A100 40GB PCIe watercooling solutions — including the Bykski N-TESLA-A100-X-V2 — are directly compatible with the CMP 170HX without modification.

**What Is Omitted**

Several categories of components are unpopulated on the CMP 170HX PCB compared to a full A100:

**VRM phases** — The voltage regulator module for the GPU core uses fewer DrMOS switching transistors and output inductors than the A100. This reduces the maximum sustained current delivery to the GPU, which contributes to the lower power ceiling of the CMP 170HX.

**NVLink** — The physical NVLink connectors and routing are fully present on the CMP 170HX PCB (identical to A100 implementation). However, some interface ICs and support components are unpopulated. More critically, NVLink is disabled at the fuse level in the GPU silicon itself, preventing activation regardless of hardware population. Enabling NVLink would require both hardware population and firmware/fuse-level enabling, which requires HULK-level cryptographic access (not publicly available).

**PCIe AC Coupling Capacitors** — Twelve of the sixteen PCIe data lanes have their AC coupling capacitors omitted from the PCB. These 0402-size capacitors are necessary for valid differential signaling on each lane. Their absence forces the PCIe link to negotiate down to x4 width. The empty pads are visible on the board and can theoretically be populated to restore x16 width, though the firmware-level Gen 1 speed lock remains even after this modification.

<figure><img src="https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-pcb-3.jpg" alt=""><figcaption></figcaption></figure>

**Power Tree**

The board accepts four power inputs that feed different subsystems:

`3V3_PEX` — provided by the PCIe slot, stepped down to 1.8V for auxiliary control circuits via an LDO.

`12V_PEX` — provided by the PCIe slot, branched to: 5V via MP1475 DC/DC converter; 1.35V via MP2988 PWM controller for one VRM phase; 2.5V (HBMVPP) via a second MP1475 for the HBM2e memory controller; and PEXVDD via another MP2988 for PCIe I/O signaling.

`12V_EXT1` and `12V_EXT2` — provided by the single 8-pin power connector (via adapter cable), used to generate the 1.0V GPU core voltage (NVVDD) and HBM memory voltage (HBMVDD) through a multi-phase MP2988 controller driving DrMOS switching transistors.

Power-on sequencing is triggered by applying 12V input, which enables the 5V rail, which enables 3.3V, which in turn enables the core voltages in sequence. The failure of any rail in this chain prevents the GPU from being detected on PCIe — a key diagnostic fact for repair work.

<figure><img src="https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-schematics-1.png" alt=""><figcaption></figcaption></figure>

**HBM2e Interposer**

The GA100-105F-A1 die and two HBM2e memory stacks are mounted on a silicon interposer — a large piece of passive silicon that provides the ultra-wide 4096-bit memory bus between the GPU and HBM stacks. This interposer is the reason the card has such an enormous copper heat spreader: it covers the entire interposer area, which is substantially larger than a typical GPU die alone. This is the same physical structure used in the A100, and the heat spreader is why opening the card requires care — it adheres firmly to the PCB via thermal paste and must be pried away carefully to avoid damage.

**Using A100 Hardware Resources**

Because the PCB is essentially an A100 board, the leaked A100 electrical schematics are a primary resource for anyone working on this card — for repair, modification, or research. These schematics have been used to repair at least one dead CMP 170HX and are the definitive reference for understanding the power delivery, sequencing, and signal routing on the board.
