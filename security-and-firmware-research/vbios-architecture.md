# VBIOS Architecture and Encryption

This page documents the complete technical architecture of NVIDIA GA100/CMP 170HX VBIOS firmware, including ROM layout, encryption methods, MAC verification regions, and strap configuration tables.

## ROM Overall Structure

**File Sizes:**
- Standard GA100 variants (A100 PCIe, CMP 170HX): 1,044,480 bytes
- A100 SXM4 and Drive A100: 1,048,576 bytes

**Memory Layout Breakdown:**

```
0x00000–0x02200: NVGI header + RFRD manifest (8.5 KB cleartext)
0x02200–0x05E00: PciAt region (15.5 KB)
0x05E00–0x0C700: FwSec headers (27 KB)
0x0C800–0x13E00: Compressed Falcon code (29.5 KB)
0x14A00–0x20000: AES-128-ECB encrypted firmware A (45.5 KB)
0x20000–0x20700: DMAP cleartext gap (1.8 KB)
0x20700–0x33800: AES-128-ECB encrypted firmware B (76.2 KB)
0x33800–0x41200: Configuration tables (54.5 KB)
0x41200–0x42A00: Strap/training tables (6 KB)
0x43A00–0x47700: Unsigned tail (15.6 KB) - OUTSIDE MAC verification
0x60000–0xA7700: Backup copy of above
0xC0000–0xFFFFF: InfoROM (unsigned)
```

## Encryption: AES-128-ECB

The firmware uses AES-128-ECB (Electronic Code Book) symmetric encryption, NOT RSA asymmetric encryption.

**Key Generation:**
- Symmetric AES key derived from MD5 initialization vector with key number as last byte
- Known plaintext pair verified: `AES_ECB(key, 0xFF×16) = 717d1494eaca317ff106195258b38377`
- Test keys are public; production keys remain proprietary

**Evidence of Symmetric Encryption:**
- 442 identical blocks found across ROM variants when decrypted
- Block-by-block redundancy impossible with asymmetric (RSA-3072) encryption
- Identical plaintext blocks encrypt to identical ciphertext blocks under ECB (ECB weakness)

## MAC Verification: Davies-Meyer Symmetric MAC

**NOT RSA-3072 Signatures** — The firmware uses a Davies-Meyer symmetric MAC, not asymmetric RSA signatures.

**MAC-Verified Ranges (per RFRD manifest at offset 0x200C):**

| Variant | MAC Start | MAC End | Notes |
|---------|-----------|---------|-------|
| 170HX 250W | 0x2200 | 0x43A00 | Configuration outside MAC |
| 170HX 300W | 0x2200 | 0x43C00 | Configuration outside MAC |
| A100 PCIe 40GB | 0x2200 | 0x44400 | Configuration outside MAC |
| A100 SXM4 | 0x2200 | 0x44400 | Configuration outside MAC |
| Drive A100 32GB | 0x2200 | 0x5AC00 | Extended MAC range |

**Security Implication:** Unsigned regions (power limits, CTRL_OPT fuse table, padding) can be modified without MAC forgery. Protected regions (CFG1 memory config, PCIe devinit) require either MAC bypass or valid symmetric key.

## Device Identification

**Primary PCI Device IDs:**

| Product | Device ID | Subsystem ID | Notes |
|---------|-----------|--------------|-------|
| A100 PCIe 40GB | 0x20F1 | 0x134E | Datacenter |
| A100 SXM4 40GB | 0x20B0 | 0x134F | Datacenter |
| CMP 170HX (various) | 0x20C2 | 0x1585 or 0x1557 | Mining card |
| **CMP 170HX alternate** | **0x2082** | **0x1585** | Variant designation |
| Drive A100 | 0x20BB | 0x133A | Drive platform |

The alternate Device ID (0x2082) appears in some CMP 170HX variants, suggesting hardware or firmware-level variance in device enumeration.

## Memory Configuration: CFG1 Strap Tables

Memory capacity and configuration are controlled via strap tables in the VBIOS. These tables specify how much HBM2e is addressable on each die.

**Strap Table Locations:**

| Variant | Primary | Backup |
|---------|---------|--------|
| A100 PCIe/SXM4 | 0x4285A | 0xA285A |
| 170HX 8GB/10GB/16GB | 0x41D41 | 0xA1D41 |
| 170HX 300W | 0x41F41 | 0xA1F41 |
| Drive A100 | 0x3A7D2 | N/A |

**Tier Byte Values (16 entries × 4 bytes each):**

| Value | Addressing | Capacity per Die |
|-------|-----------|-----------------|
| 0x44 | 12-bit row addressing | 2 GB (RESTRICTED) |
| 0x55 | Intermediate | 4 GB (INTERMEDIATE) |
| 0x66 | 14-bit row addressing | 8 GB (NORMAL) |
| 0x77 | 15-bit row addressing | 16 GB (FULL) |

**Strap 4 Configuration (Memory Capacity):**

| Variant | Strap 4 Value | Effective Capacity |
|---------|---------------|-------------------|
| A100 PCIe/SXM4 | 0x66 | Unrestricted (8 GB/die) |
| 170HX 8GB/16GB/300W | 0x44 | **Restricted to 2 GB/die addressing** |
| 170HX 10GB | 0x44 (S4) but 0x77 (S5,S7) | Mixed restrictions |
| Drive A100 32GB | 0x66 | Unrestricted |

**Additional 170HX Restrictions:**
- Strap 5: restricted to 0x55 on 8GB/16GB/300W SKUs
- Strap 7: restricted to 0x44 on 8GB/16GB/300W SKUs
- 10GB variant partially restores straps 5 and 7 to 0x77, but keeps strap 4 nerfed at 0x44

## Power Configuration

**Power Limit Register (Offset 0x45E45 in unsigned tail):**

| SKU | Register Value | Power Limit |
|-----|----------------|-------------|
| 170HX 250W | 0x90D003 | 250W |
| 170HX 300W | 0xE0934 | 300W |

This region is outside MAC-protected areas, making it modifiable via SPI programmer without cryptographic key access.

## License Region: HULK Injection Point

The VBIOS contains a reserved region for HULK cryptographic licenses, relevant if HULK keys were to be injected.

**170HX License Region (0xFE000–0xFEFFF):**
- `0xFE008`: "LU" header (region size 0x1000, version 1)
- `0xFE010`: Slot table directory
- `0xFE4FD`: HLK slot header (0x0460 bytes capacity)
- **`0xFE504`: HLK payload injection point (1113 bytes available)**

**A100 License Region:** Identical structure at 0xFF000–0xFFFFF

This architectural detail is relevant for theoretical HULK key injection scenarios, though no public HULK keys are available.

## Memory Training Tables

**170HX 8GB/10GB Training Table Marker:**
```
0x05BA0240 (12 entries for 8GB/10GB)
0x05BA0240 (20 entries for 16GB/300W)
```

**A100 SXM4 Training Table Markers:**
```
0x40029605, 0x40029e05, 0x6002ca05 (16 entries)
```

**Drive A100 32GB MRS Values (Memory Refresh Settings):**
- MRS_0: 0x00000025 (vs 170HX: 0x00000003)
- MRS_2: 0x00200fc0 (professional) vs 0x002000cf (170HX)

## VBIOS Versions Reference

| Product | Version | Build Date | Notes |
|---------|---------|------------|-------|
| A100 PCIe 40GB | 92.00.90.00.08 | 2022-01-05 | Latest datacenter |
| A100 SXM4 40GB | 92.00.45.00.03 | 2021-06-16 | Original release |
| CMP 170HX 8GB | 92.00.67.00.01 | 2021-05-14 | Original 250W |
| CMP 170HX 10GB | 92.00.66.00.02 | 2021-04-23 | 10GB variant |
| CMP 170HX 300W | 92.00.6D.00.0A | 2022-04-07 | 300W variant |
| Drive A100 32GB | 92.00.A0.00.01 | Unknown | Drive platform |

## Cross-Variant Differences

**170HX 8GB vs 10GB Comparison:**
- Total differing bytes: ~2,637 (0.25% of ROM)
- Device ID change at offset 0x000E
- Strap 5 and 7 values differ
- Training table versions differ (0x0067 vs 0x0066)

**Drive A100 vs A100 PCIe Comparison:**
- Total differing bytes: 468,557 (45% of ROM)
- FwSec body 96 KB larger
- Different firmware build entirely
- Per-unit calibration in InfoROM differs significantly

## Modifiable vs Protected Regions Summary

**Unsigned/Modifiable Regions (via CH341A SPI programmer):**
- Power limit configuration (0x45E45)
- CTRL_OPT fuse table
- FF padding areas
- No MAC verification required

**MAC-Protected Regions (Require MAC Forgery):**
- CFG1 memory configuration (strap tables)
- PCIe speed devinit settings
- Firmware encryption keys
- Core firmware sections

**Potential Modification Paths:**
1. Direct SPI programmer access to unsigned regions
2. DFA attacks targeting AES key derivation
3. DFA attacks targeting MAC forgery capability
4. HULK certificate injection via reserved region
5. Bootloader exploitation (Falcon-level, requires signing bypass)
