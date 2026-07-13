# NVLink — Current Understanding and Why It Remains Disabled

**NVLink on the CMP 170HX — Why It Cannot Be Enabled**

Despite containing all the necessary physical hardware, NVLink on the CMP 170HX is disabled via fuse-level security in the GPU silicon itself. This page clarifies the current understanding of this barrier, documents what hardware IS present, and explains why component population alone is insufficient to enable NVLink.

**What NVLink Would Provide**

NVLink on the A100 provides 600 GB/s of GPU-to-GPU bidirectional bandwidth — 9.4× faster than the current PCIe Gen 1 x4 bandwidth of the CMP 170HX. For multi-GPU compute workloads this would be transformative: it would allow two CMP 170HX cards to act as a single logical GPU with combined HBM2e bandwidth, enabling larger model sizes and dramatically faster inter-GPU communication.

The full A100 NVLink implementation consists of 6 NVHS (NVIDIA High Speed) groups, each carrying 4 lanes of TX and 4 lanes of RX — 24 NVLink lanes total, forming 12 NVLink links at 2 lanes each. This is what the gold fingers on the CMP 170HX PCB are wired to support.

**The Root Blocker: Fuse-Level Disablement**

The fundamental barrier to NVLink on the CMP 170HX is **fuse-level security** — the GA100-105F-A1 GPU die has fuses blown at the silicon level that disable the NVLink interface. These fuses are protected by NVIDIA's HULK cryptographic key system. Unlocking them would require:

1. **HULK License**: A proprietary NVIDIA cryptographic license not publicly available
2. **Fuse Blown in Silicon**: Once blown, cannot be "re-enabled" via software alone — requires either:
   - Access to the blown fuse bit masks and ability to reprogram (not possible without HULK keys)
   - Physical silicon modification (silicon-level laser fuse repair — not feasible for most)

Even if all supporting components were populated on the PCB, the GPU firmware would still refuse to enable NVLink because the fuse-level locks would prevent it. This is a hardware security barrier, not a firmware limitation.

**What IS Present on the PCB**

The physical NVLink infrastructure on the CMP 170HX is comprehensive:

- **NVLink gold finger contacts** along the top edge of the PCB, identical to A100 40GB PCIe
- **PCB traces** routing from the GA100-105F-A1 die's NVHS signal pads to the connector pads
- **Physical routing** for all 24 NVLink lanes (6 NVHS groups × 4 TX + 4 RX each)
- **Power delivery** infrastructure (12V_PWR, 3V3_PROT rails) for NVLink power

This complete infrastructure exists because the CMP 170HX uses the same PCB as the A100 — removing these would have required a custom board revision. The PCB was never modified; components were simply omitted.

**Supporting Components — Missing but Secondary to Fuse Lock**

Even if the fuse-level security could be bypassed, several supporting components are unpopulated on the CMP 170HX. These are documented below for completeness, but their absence is NOT the reason NVLink is disabled — the fuse-level locks are the fundamental barrier. Analysis of the A100 schematics and community investigation reveals the following:

**1. Physical NVLink Bridge Connectors**

Three connectors labeled `CON_NVLINK_GA10X` — RIGHT, MIDDLE, and LEFT — form the physical NVLink bridge interface. These are the edge connectors that an NVLink bridge card plugs into. All three are unpopulated on the CMP 170HX. Without these, no NVLink bridge can physically connect.

**2. NVLink Power and Signal Resistors**

Community analysis of the A100 schematics identified five resistors physically missing on the CMP 170HX that route power and signals to the middle NVLink connector:

| Reference | Location             | Purpose                    | A100 Status | CMP 170HX Status |
| --------- | -------------------- | -------------------------- | ----------- | ---------------- |
| R976      | Ball F51 (under die) | Signal routing             | Populated   | Missing          |
| R1030     | Page 4, Ball G1      | Signal routing             | Populated   | Missing          |
| R1029     | Page 5, Ball F1      | Signal routing             | Populated   | Missing          |
| R1024     | Page 17              | Connector routing          | Populated   | Missing          |
| R238      | Page 17              | Connector routing          | Populated   | Missing          |
| R236      | Page 17              | Connector routing          | Populated   | Missing          |

These resistors are physically small and traceable with careful inspection. R976's placement under the GPU die makes it the most difficult to access, potentially requiring micro-soldering or thin wire routing to ball F51 at the die edge.

**3. NVHS Power Decoupling Capacitors**

Page 9 (GPU: NVHS POWER) shows that each NVHS group pair (0,1 / 2,3 / 4,5) requires its own bank of power supply decoupling capacitors on the `PEX_DVDD` rail. All of these are missing on the CMP 170HX:

| Value     | Package | Qty per group | Groups | Total |
| --------- | ------- | ------------- | ------ | ----- |
| 1µF X6S   | 0402 4V | 6             | 3      | 18    |
| 4.7µF X6S | 0402 4V | 3             | 3      | 9     |

**4. NVLink Bridge Active Electronics**

The NVLink bridge itself contains active components including a microcontroller and ROM chip. These are NOT on the CMP 170HX PCB — they are part of the separate physical NVLink bridge card that connects between two GPUs. A functional NVLink bridge card can be sourced from used A100 or other compatible systems, but must be purchased separately.

---

## Community Research and Practical Attempts

**NVLink as a Path Forward**

The community recognizes NVLink as a superior alternative to PCIe for multi-GPU scaling. Without NVLink or high-speed PCIe (Gen 4+), multi-GPU configurations are severely bandwidth-limited — tensor parallelism across 3+ cards is impractical due to PCIe bottlenecks.

**Active Development**

The community is actively exploring practical solutions:

- **Custom Single-Slot Cooling with 8-way NVLink:** Compact thermal solutions specifically designed for single-slot NVLink-enabled configurations are under development. This would allow building denser multi-GPU systems with full NVLink bandwidth without requiring dual-slot spacing per GPU.
- **PCIe-based Alternatives:** PCIe switches and bridges are being evaluated as an interim solution while fuse-level NVLink enablement remains blocked. A 4-way NVLink mesh configuration provides approximately 200 GB/s of interconnect bandwidth.

**Scalability Perspective**

For practical multi-GPU systems:
- **2 GPUs:** 600 GB/s (full bidirectional NVLink) — excellent for tensor parallelism
- **4 GPUs:** Full mesh topology possible, though total bisection bandwidth is shared
- **8 GPUs:** Single-slot form factor being prototyped; requires specialized bridges and cooling

**Feasibility Assessment**

Populating the missing resistors and capacitors is physically feasible for experienced solderers, but **does not solve the fuse-level security barrier**. Even a fully populated and component-complete CMP 170HX would still be unable to enable NVLink without:

1. HULK cryptographic keys (not publicly available)
2. Fuse reprogramming capability (requires specialized equipment or silicon modification)
3. GSP/devinit firmware modifications (blocked by VBIOS signing)

The physical component population effort would only make sense in conjunction with solving the fuse-level lock — either through an exploit or through direct GPU modification.
| 10µF X6S  | 0603 4V | 3             | 3      | 9     |
| 22µF X6S  | 0603 4V | 2             | 3      | 6     |

**Total decap capacitors required: 42**

**3. Per-PLL Decoupling Capacitors**

Pages 4–5 (GPU: NVHS 0,1,2,3,4,5) show that each NVHS group has a dedicated PLL voltage supply (`NVHS0_PLL_HVDD`, `NVHS1_PLL_HVDD`, `NVHS2_PLL_HVDD`) with its own decoupling network. Each PLL supply uses small capacitor pairs visible in the schematic. Approximately 18 additional capacitors are required across all 6 PLL supplies.

**4. NVHS Termination Resistors**

Each NVHS group has a `NVHS_TERMP` termination resistor network — 6 total, one per NVHS group. These provide the correct impedance termination for the high-speed differential pairs. Without them, signal integrity on the NVLink lanes will be severely compromised regardless of other components.

**5. ID ROM Circuit**

Page 17 (IO: NVHS x16, CON3, MIDDLE connector) shows an ID ROM circuit on the NVLink bridge connector. This is a small I2C EEPROM (with resistors R1024/R1025 visible in the schematic) that stores identification data for the NVLink bridge. The A100 uses this to identify what type of NVLink bridge is attached. This circuit needs to be populated and programmed with appropriate data.

**6. Connector Control Signals**

Each of the three NVLink connectors (RIGHT, MIDDLE, LEFT) has associated control signals that need supporting circuitry:

* `PWR_EN` — power enable for each connector, driven via resistor divider from `3V3_PROT`
* `NVHS_ADDR` — NVLink address assignment per connector
* `RASTERSYNCH` — synchronization signal
* `SWAPREADY` — lane swap readiness signal
* `PEX_RST_BUF*` — reset signal buffered from PCIe reset

Each connector also requires `12V_PWR` and `3V3_PROT` power routing with appropriate filtering.

**The Complete Component List**

Based on schematic analysis, the theoretical complete component list for NVLink population is:

| Category                 | Components                             | Approx Qty | Source                                   |
| ------------------------ | -------------------------------------- | ---------- | ---------------------------------------- |
| NVLink bridge connectors | CON\_NVLINK\_GA10X (edge connector)    | 3          | Salvage from A100 or industrial supplier |
| NVHS decap capacitors    | 1µF/4.7µF/10µF/22µF (various packages) | \~42       | Mouser/DigiKey/LCSC                      |
| PLL decap capacitors     | Small value caps per PLL               | \~18       | Mouser/DigiKey/LCSC                      |
| Termination resistors    | NVHS\_TERMP networks                   | 6          | Per schematic values                     |
| ID ROM                   | Small I2C EEPROM                       | 1          | Standard parts                           |
| Control signal resistors | Pull-ups, dividers per connector       | \~20       | Per schematic values                     |

**Why Component Population Is Not Sufficient**

The common misconception is that populating the missing components would enable NVLink. This is not the case. Here's why:

Even if all missing components were populated perfectly:
- NVHS power decoupling capacitors (42 total)
- PLL decoupling capacitors (~18)
- NVHS termination resistors (6)
- ID ROM circuit and supporting resistors
- NVLink bridge connectors

...the GPU firmware would still refuse to enable NVLink because the fuse-level security barrier at the silicon level would prevent it.

The fuses are one-way: once blown, they cannot be electronically "unblown" without the HULK cryptographic keys. This was NVIDIA's security-by-design approach to ensure that mining-specific GPUs remain mining-specific, even if someone physically reconstructs all the removed components.

**Potential Paths Forward (Theoretical)**

Several research directions have been explored or proposed:

1. **HULK Key Acquisition** — If NVIDIA's HULK keys could be obtained (through legal, technical, or research channels), the fuses could theoretically be unlocked. This is the only path that would not require silicon modification.

2. **Silicon-Level Fuse Repair** — Specialized equipment (laser fuse rewriting) could theoretically reprogram the blown fuse bits. This requires: 
   - De-lidding the GPU die (destructive, voids warranty)
   - Access to specialized laser fuse-writing equipment
   - Knowledge of NVIDIA's fuse bit structure (not public)

3. **Firmware Exploit** — If a firmware vulnerability were discovered that allows bypassing fuse checks at runtime, NVLink could be enabled without HULK keys. This is the most practical research direction but has not yet yielded results for Ampere generation.

4. **Silicon Die Replacement** — Replace the GA100-105F-A1 die with a full GA100-895 die that doesn't have NVLink fuses blown. This requires:
   - De-lidding the existing package
   - Removing the current die (removes solder, destroys original die)
   - Sourcing and micro-soldering a replacement die
   - Re-applying thermal interface and re-liding
   - This is extremely high-risk and likely not practical

**Current Status**

As of June 2026, NVLink remains disabled on all production CMP 170HX cards due to fuse-level security. However, active research efforts are underway in the community:

**Ongoing Development:**
- 🔧 **PCIe NVLink Bridge Development**: Community members (notably jonpry_77715) are actively designing and building custom NVLink bridge adapters for PCIe cards. PCB designs are in progress with target completion expected by end of June 2026.
- 📊 **Schematic Analysis**: Complete A100 schematics are being used to guide PCIe bridge development. Reverse engineering of existing commercial NVLink bridges is ongoing.
- 🎯 **Configuration Research**: The primary technical challenge identified is "configuring the nvlink" — the bridge hardware can be built, but enabling it on the 170HX requires solving the fuse-level barrier or finding a firmware workaround.

**Research Status:**
- ❌ **HULK Key Acquisition**: No public access to NVIDIA's HULK cryptographic keys
- ❌ **Silicon Modification**: No publicly documented attempts or successes
- ❌ **Firmware Exploit**: Ampere generation lacks known exploits for fuse-level access
- 🔧 **Component Population Study**: Community members have populated all missing NVLink components; the limiting factor remains the fuse-level security barrier, not component availability
- 🚧 **PCIe Bridge Hardware**: Active development, not yet publicly released

The missing components are a secondary concern — valuable to document and understand, but solving the fuse-level disablement is the prerequisite to making component population meaningful.

**Schematic Reference Pages**

All page references are from the NVIDIA A100 GA100-883 P1001-B02 reference schematic:

| Page | Title                   | Relevance                                                            |
| ---- | ----------------------- | -------------------------------------------------------------------- |
| 4    | GPU: NVHS 0,1,2 and 3   | GPU die NVLink signal pins, PLL supplies, termination                |
| 5    | GPU: NVHS 4 and 5       | GPU die NVLink signal pins continued                                 |
| 9    | GPU: NVHS POWER         | All NVHS power decoupling component values and reference designators |
| 16   | IO: NVHS x16, CON1/CON2 | RIGHT and LEFT NVLink connectors, lane routing, control signals      |
| 17   | IO: NVHS x16, CON3      | MIDDLE NVLink connector, ID ROM circuit, power routing               |

**Resources for Research**

* NVIDIA A100 GA100-883 P1001-B02 reference schematics (see Repair Guide for sourcing)
* NVIDIA A100 NVLink product brief
* NVLink bridge connector physical specifications
* NVIDIA CUDA Multi-Process Service documentation (relevant for multi-GPU NVLink topology)

**Call for Contributors**

Enabling NVLink would effectively double the usable compute capacity of a two-card CMP 170HX system and unlock multi-GPU workloads previously impossible on this hardware. This remains one of the highest-impact unsolved problems for the CMP 170HX.

**What Would Actually Help:**

* **HULK Key Research**: If you have cryptographic expertise and are interested in exploring NVIDIA's HULK key system or potential vulnerabilities
* **Firmware Exploit Research**: Knowledge of Ampere GPU firmware internals, VBIOS parsing, or PTX modification techniques
* **Silicon/Hardware Expertise**: Experience with GPU die-level work, fuse structures, or hardware security
* **Documentation Contributions**: If you have reverse-engineered any aspect of the NVLink fuse structures or have technical details about Ampere security mechanisms

**What Won't Help (at this point):**

* Component population work — valuable for documentation but won't enable NVLink while fuses remain blown
* VBIOS modification — firmware-level restrictions aren't the root cause
* PCB rework techniques — the physical hardware is ready, the silicon barrier isn't

If you have insights into any of these areas, please document your findings and submit them to the community.
