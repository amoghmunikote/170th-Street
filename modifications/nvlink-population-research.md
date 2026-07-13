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

**Supporting Components — What Is Missing (Not the Root Problem)**

Even if the fuse-level security could be bypassed, several supporting components are unpopulated on the CMP 170HX. These are documented below for completeness, but their absence is NOT the reason NVLink is disabled — the fuse-level locks are. Analysis of the A100 schematics (Pages 4, 5, 9, 16, 17) reveals the following missing categories:

**1. Physical NVLink Bridge Connectors**

Three connectors labeled `CON_NVLINK_GA10X` — RIGHT, MIDDLE, and LEFT — form the physical NVLink bridge interface. These are the edge connectors that an NVLink bridge card plugs into. All three are unpopulated on the CMP 170HX. Without these, no NVLink bridge can physically connect.

**2. NVHS Power Decoupling Capacitors**

Page 9 (GPU: NVHS POWER) shows that each NVHS group pair (0,1 / 2,3 / 4,5) requires its own bank of power supply decoupling capacitors on the `PEX_DVDD` rail. All of these are missing on the CMP 170HX:

| Value     | Package | Qty per group | Groups | Total |
| --------- | ------- | ------------- | ------ | ----- |
| 1µF X6S   | 0402 4V | 6             | 3      | 18    |
| 4.7µF X6S | 0402 4V | 3             | 3      | 9     |
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

As of 2026, NVLink remains disabled on all known CMP 170HX cards. No successful unlock has been publicly documented. The barrier is the fuse-level security in the GPU silicon, not missing components or firmware limitations.

**Research Status:**
- ❌ **HULK Key Acquisition**: No public access to NVIDIA's HULK cryptographic keys
- ❌ **Silicon Modification**: No publicly documented attempts or successes
- ❌ **Firmware Exploit**: Ampere generation lacks known exploits for fuse-level access
- ⏳ **Component Population Study**: Would be worth doing for completeness, but only after fuse-level barrier is resolved

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
