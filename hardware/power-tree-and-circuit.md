# Power Tree & Circuit

**Power Tree & Circuit**

This page documents the power delivery architecture of the CMP 170HX, based on the leaked NVIDIA Tesla A100 electrical schematics. This information is relevant for repair work, understanding the card's behavior under load, and hardware modification research.

<figure><img src="https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-schematics-1.png" alt=""><figcaption></figcaption></figure>

**Overview**

The CMP 170HX PCB uses the same power delivery architecture as the A100 40GB PCIe reference design. Power enters via four inputs: the PCIe slot's 3.3V and 12V rails, and two 12V lines from the external 8-pin power connector. These are converted to multiple regulated voltages that power different subsystems in a defined sequence.

**Voltage Rails**

`3V3_PEX` — 3.3V from the PCIe slot. Stepped down to 1.8V via a Low-Dropout Regulator (LDO) for auxiliary control circuits.

`5V` — Generated from `12V_PEX` by the MP1475DJ DC/DC buck converter. Powers the logic chips responsible for power sequencing.

`3V3_SEQ` — 3.3V sequential rail. Generated from the 5V rail by the GS7155NVTD LDO. Powers the SN74LV1T08 AND gate logic chips used throughout the sequencing chain.

`1.35V` — Generated from `12V_PEX` via one phase of the MP2988 DC/DC controller. Purpose: auxiliary logic voltage.

`HBMVPP` — 2.5V, generated from `12V_PEX` via a second MP1475 DC/DC converter. Powers the HBM2e memory controller inside the GA100-105F-A1 die.

`PEXVDD` — PCIe I/O voltage, generated from `12V_PEX` via an MP2988 controller with one DrMOS phase.

`NVVDD` — 1.0V GPU core voltage. Generated from `12V_EXT1` and `12V_EXT2` (the external 8-pin connector) via the MP2988 multi-phase controller driving multiple DrMOS switching transistors. This is the primary power rail for the GPU compute cores.

`HBMVDD` — HBM2e memory voltage. Also generated from `12V_EXT1/EXT2` via MP2988 and DrMOS phases. Powers the HBM2e stacks themselves.

**Power-On Sequencing**

The card uses a defined power-on sequence to prevent invalid states and component damage during startup. The sequence is triggered by the presence of 12V input:

1. `12V_F` (filtered 12V input) present → resistive divider generates `5V_PS_EN` signal
2. `5V_PS_EN` enables the MP1475DJ 5V DC/DC converter
3. 5V Power Good signal (`PS_5V_PGOOD`) enables the GS7155NVTD 3.3V LDO via R1200
4. `3V3_SEQ` enables SN74LV1T08 AND gates throughout the board
5. AND gates enable `PEXVDD`, `NVVDD`, `1V35`, and `1V8` in sequence
6. GPU core and memory voltages stabilize
7. PCIe link negotiates and GPU is detected by the host

Failure of any step in this chain results in the GPU not being detected on PCIe. This is the most common symptom of a faulty CMP 170HX and the starting point for any repair diagnosis.

<figure><img src="https://niconiconi.neocities.org/img/nvidia-cmp-170hx-review/cmp-170hx-schematics-2.png" alt=""><figcaption></figcaption></figure>

**Key ICs**

| IC       | Part       | Function                                             |
| -------- | ---------- | ---------------------------------------------------- |
| U?       | MP1475DJ   | 5V DC/DC buck converter                              |
| U?       | GS7155NVTD | 3.3V LDO (sequential)                                |
| U?       | MP2988     | Multi-phase PWM controller (NVVDD, HBMVDD, PEXVDD)   |
| U?       | SN74LV1T08 | Dual-input AND gate (power sequencing)               |
| Multiple | DrMOS      | Integrated gate driver + MOSFET for switching phases |

**CMP 170HX Differences from A100**

The CMP 170HX has fewer populated DrMOS phases for NVVDD and HBMVDD compared to the A100, reducing maximum sustained current delivery. The NVLink power circuitry is entirely absent. Some auxiliary ICs may also be depopulated. Otherwise the power tree is functionally identical.

**Note on R1200**

The 0-ohm jumper resistor R1200, which connects the `PS_5V_PGOOD` signal to the GS7155NVTD LDO enable pin, appears in the schematic but is bypassed by a direct copper trace on the actual PCB. This means removing R1200 does not actually isolate this signal — the trace maintains the connection. This is an important detail for repair work, as removing R1200 to isolate a fault may instead damage the PCB trace.

---

**GSP Firmware Control Layer**

Beyond the fixed power delivery hardware, the CMP 170HX's actual runtime behavior is controlled by the GSP (Graphics System Processor) firmware, which is loaded from the signed VBIOS at boot time.

**What GSP Controls:**

**Memory Configuration:**
- Memory capacity allocation (determines visible VRAM: 8GB, 10GB, or potentially 16GB)
- Memory clock rates and timing parameters
- HBM refresh cycles and power gating behavior
- Memory training procedures during boot

**Clock & Voltage Management:**
- GPU core clock speeds and boost configurations
- Power delivery and frequency scaling tables
- Thermal throttling points

**Instruction Execution Policy:**
- FMA/FMLA/ILMA instruction throughput throttling
- DP4A/DP2A instruction throttling
- Register access permission levels (PLM: Permission Level Mechanism)

**Hardware Feature Gating:**
- NVLink activation (currently disabled via fuse-level security)
- PCIe speed negotiation parameters
- DirectX/Vulkan API enablement

**Modification Barrier:**
The VBIOS firmware containing GSP initialization code is cryptographically signed using RSA-3072 and ECDSA. Any modification to the VBIOS requires either:

1. **HULK License Key** — A proprietary NVIDIA security key required to unlock register access for GSP firmware modification. Licenses are extremely restricted and email-request-only within NVIDIA.

2. **Signature Bypass** — Cracking or compromising the RSA/ECDSA signing mechanism. As of 2026, no publicly known method exists for Ampere GPUs (though OMGVflash achieved this for Turing).

3. **Exploit Path** — Finding an undocumented vulnerability in GSP firmware or the Falcon security processor that allows code execution in a privileged mode (Heavy Secure). Active research ongoing.

**Practical Implication:**
The physical hardware (power rails, memory, compute cores) is fully present and functional, but firmware licensing prevents most users from reconfiguring these subsystems. This is why seemingly simple changes (enabling extra memory, unlocking clock speeds, removing throttles) remain off-limits without a firmware modification breakthrough.
