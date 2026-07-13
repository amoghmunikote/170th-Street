# GA100/CMP 170HX Fuse and Register Architecture Reference

This page documents the complete fuse and register architecture of the GA100 GPU, with specific focus on how the CMP 170HX is restricted versus unrestricted variants.

## Fuse Architecture Overview

**Critical Fuses on CMP 170HX:**
- All 9 SM speed-select fuses at maximum throttle: `0x5`
- NVLink completely disabled: `FUSE_NVLINK_DIS = 0x7` (all groups)
- PCIe generation locked: `FUSE_PCIE_GEN23_DIS = 0x1`
- Memory configuration locked: `FUSE_MEM_LOCKED = 0x1`
- **Software override disabled: `FUSE_EN_SW_OVERRIDE = 0x0`**

**Cross-Variant Fuse Patterns:**

| Fuse | Consumer GeForce | Professional/Datacenter | CMP 170HX | Notes |
|------|-----------------|------------------------|-----------|-------|
| SM Speed Select | 0x1 | 0x0 | **0x5** | Throttle level |
| ECC | 0x0 | 0x1 | 0x0 | Disabled on CMP |
| Memory Locked | 0x1 | 0x1 | **0x1** | All variants locked |
| EN_SW_OVERRIDE | 0x1 | 0x0 | **0x0** | Cannot override via firmware |
| NVLink Disable | varies | 0x0 | **0x7** | All groups disabled |
| PCIE_GEN23_DIS | 0x0 | 0x0 | **0x1** | Gen2/3 disabled |

The pattern shows CMP 170HX uses maximum throttle values combined with disabled overrides, creating multi-layered lockdown.

## Speed-Select Fuse Encoding

The SM speed-select system uses 3-bit values (0-7) to represent throttle levels:

| Value | Throttle Level | Effect |
|-------|----------------|--------|
| 0x0 | Unrestricted | Full performance (A100, datacenter) |
| 0x1 | Light throttle | Partial throttle (consumer GeForce) |
| 0x2–0x4 | Medium throttle | Progressive restriction |
| **0x5** | **Heavy throttle** | **FMA/FP32 at 1/32nd throughput (CMP)** |
| 0x6–0x7 | Reserved | Not used in production |

**Affected Fuses (9 total):**
- `FUSE_SS_FFMA`: FMA throttle
- `FUSE_SS_FMLA16`: FMA Low 16-bit
- `FUSE_SS_FMLA32`: FMA Low 32-bit
- `FUSE_SS_IMLA0` through `FUSE_SS_IMLA4`: Integer multiply accumulators

All nine are set to `0x5` on CMP 170HX, creating comprehensive throttling.

## MMIO Register Reference

**Throttle Control Chain:**

| Register | Address | Function | CMP 170HX Value |
|----------|---------|----------|-----------------|
| SM_ISSUE_RATE_MODIFIER | 0x00504204 | Global issue rate | 0x5 (throttled) |
| FECS_FEAT_OVERRIDE | 0x00409664 | Feature override (PLM L3-gated) | Implementation dependent |
| FECS_FEAT_READOUT_1 | 0x00409668 | Throttle state display (RO) | Reflects fuse state |
| FEAT_OVR_SM_SPD[0-8] | 0x0082381C+ | Per-unit speed override (9 total) | Read-only on CMP |

**Memory and FBPA Configuration:**

| Register | Address | Function | CMP 170HX | A100 |
|----------|---------|----------|-----------|------|
| FBPA_CFG1_BROADCAST | 0x009A0204 | Memory geometry config | 0x02449000 | 0x02669000 |
| FUSE_HALF_FBPA_EN | 0x0082049C | Per-FBPA half-capacity mask | Varies | 0x0 |
| FUSE_GPC_DISABLE | 0x00820350 | GPC disable bitmask | Varies | 0x0 |
| FUSE_FBPA_DISABLE | 0x00820368 | FBPA floorsweep (24-bit) | Varies | 0x0 |

**Fuse Controller:**

| Register | Address | Function |
|----------|---------|----------|
| Fuse Controller CMD | 0x00820000 | Command register |
| Fuse Controller STATE | 0x00820004 | State machine |
| EN_SW_OVERRIDE | 0x00820040 | Software override enable (gate) |
| SM_SPEED_SELECT_PLM | 0x008200FC | Power Limit Module (shared fuse interface) |

## Memory Capacity Architecture

The GA100 has a sophisticated HBM2e configuration system that can address different memory capacities on each die independently.

**FBPA (First-Level Buffer Per-Aperture) Configuration:**
- Total FBPAs on full GA100: 24
- Drive A100: 16 active, 8 disabled
- CMP 170HX 8GB: varies by unit (typically 20 active)
- Address configuration via `FBPA_CFG1_BROADCAST` at 0x009A0204

**MRS (Mode Register Set) Values:**

| Parameter | 170HX | A100 Professional | Drive A100 |
|-----------|-------|-------------------|------------|
| MRS_0 | 0x00000003 | 0x00000003 | 0x00000025 |
| MRS_2 | 0x002000cf | 0x00200fc0 | 0x00200fc0 |

**Memory Addressing:**
- Strap configuration determines addressable capacity per die
- CFG1 value selects which tier the GPU uses
- All variants have fuse lock preventing runtime override

## Security Features

**Permanent Locks:**

| Feature | Register/Fuse | CMP Status | Effect |
|---------|--------------|-----------|--------|
| DIS_SW_OVR | FUSE bit | 0x1 (locked) | Prevents firmware override of fuses |
| FEAT_OVR_DIS | Architecture | 0x0 (available) | Override path exists but gated |
| QUADRO_WR_SEC | FUSE bit | 0x1 (locked) | Gates PLM write at 0x823804 |
| Fuse Bank Lock | Hardware | Always locked | Runtime read-only access only |

**Effective Security Model:**
- Fuses blown at manufacturing: permanent silicon-level restrictions
- Firmware cannot override due to EN_SW_OVERRIDE = 0x0
- Register writes to throttle/memory config gates require HS (heavy-secure) Falcon privilege
- HS Falcon code requires RSA signature (not publicly available)

## Hardware Variance Detection

**IEEE 1500 JTAG Interface:**
- Shadow WDR register: per-die unique fingerprint
- CMP 170HX units show variance in I1500_SHADOW_WDR values
- Indicates silicon-level manufacturing differences between units

## Register Accessibility Notes

**BAR0 Aperture Limitations:**
- GA100 has a 256 KB BAR0 on CMP 170HX (vs 16 MB on A100)
- Some fuse addresses return BAR0 placeholder when queried outside valid range
- Direct fuse readback possible, but write access gated through Falcon only

**PROT (Protection Level) Masking:**
- PROT configuration restricts certain register ranges from host access
- PCIe config space registers (link control, capabilities) are PROT-protected
- Cannot be modified from userspace via setpci or nvapeek/nvapoke

## Implications for Unlock Research

**What Cannot Be Modified (Without Exploits):**
- Any fuse value (permanent silicon-level)
- Speed-select throttle (fuse-locked, no SW override)
- Memory capacity addressing (strap-locked, MAC-protected VBIOS)
- PCIe generation (fuse + VBIOS MAC)
- NVLink enablement (fuse-disabled at silicon level)

**What CAN Be Modified:**
- Power limit register in unsigned VBIOS tail
- CTRL_OPT fuse table (currently all zeros, under investigation)
- AC coupling capacitors on PCB (hardware modification → x4 to x16 width)

**Exploit Paths Require:**
1. AES key recovery (Davies-Meyer MAC forgery)
2. Falcon HS signing capability (RSA key or exploit)
3. Hardware-level fault injection (DFA attacks)
4. Physical silicon modification (laser fuse repair)

None of these are currently public for Ampere generation.
