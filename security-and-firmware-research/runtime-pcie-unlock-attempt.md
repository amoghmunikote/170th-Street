# Runtime PCIe Speed Unlock — Attempted & Failed

**Runtime PCIe Speed Unlock — Attempted & Failed**

This page documents the complete attempt to unlock the CMP 170HX's PCIe speed lock from userspace on a running system, and why every known software-only path is blocked. It is here to save future researchers the time of repeating these tests — the negative result is itself a contribution, because it bounds the problem and rules out otherwise plausible approaches.

**The Goal**

Raise PCIe speed above Gen 1 (2.5 GT/s) without modifying the VBIOS. A successful Gen 4 x16 unlock would raise PCIe bandwidth from \~0.85 GB/s to \~64 GB/s — a 64× improvement that would transform the card's usability for any workload requiring CPU↔GPU data transfer.

**What Was Confirmed About the Lock**

From `lspci -s 0000:21:00.0 -vvv` on a CMP 170HX with VBIOS `92.00.6D.00.0A`:

```
LnkCap:  Port #0, Speed 2.5GT/s, Width x16, ASPM not supported
LnkSta:  Speed 2.5GT/s, Width x4 (downgraded)
LnkCap2: Supported Link Speeds: 2.5GT/s
LnkCtl2: Target Link Speed: 2.5GT/s
```

The card advertises **only 2.5 GT/s** in its Supported Link Speeds Vector (LnkCap2 bits 6:1 = `000001`). During PCIe link training, the Root Complex sees Gen 1 as the highest common speed and negotiates there. The x4 width downgrade is a separate issue caused by the missing PCB AC coupling capacitors (see [PCIe Capacitor Mod](../modifications/pcie-capacitor-mod.md)).

These two restrictions are independent:

* Gen 1 speed lock — enforced in firmware
* x4 width downgrade — enforced by PCB component omission

This page covers attempts to bypass the first restriction only.

**Path 1 — `setpci` Write to LnkCap2**

The Supported Link Speeds Vector lives at PCIe Express Capability offset `0x2C`. Writing all speeds enabled (bits 6:1 = `0x7e` for Gen1 through Gen5 supported):

```bash
# Read current value
sudo setpci -s 0000:21:00.0 CAP_EXP+2C.L
# Returns: 00000002

# Attempt to enable all speeds
sudo setpci -s 0000:21:00.0 CAP_EXP+2C.L=0000007e

# Read back
sudo setpci -s 0000:21:00.0 CAP_EXP+2C.L
# Returns: 00000002 (write silently dropped)
```

**Result: REJECTED.** The Supported Link Speeds Vector register is hardware read-only on this card. Standard PCIe config-space writes are silently discarded — no error, no exception, just no change. The register value is programmed by firmware at boot and then locked.

**Path 2 — Upstream Root Port Retrain**

Using Alex Forencich's [`pcie_set_speed.sh`](https://raw.githubusercontent.com/alexforencich/verilog-pcie/master/scripts/pcie_set_speed.sh) to set the upstream Root Port's Target Link Speed and trigger a retrain via the Retrain Link bit in Link Control. In theory this forces the PCIe LTSSM back into Recovery state, where a new speed can be negotiated.

**Result: No change.** The link retrains and re-enters L0 at Gen 1 because the endpoint (the CMP 170HX) re-advertises Gen 1 only in its TS1/TS2 ordered sets during `Recovery.RcvrLock`. PCIe speed negotiation is a mutual advertisement — both sides must advertise a higher speed for the link to train above Gen 1. No amount of host-side retrain loops changes what the endpoint's firmware-configured LTSSM sends in its training sequences.

**Path 3 — MMIO BAR0 Register Poke via envytools**

The hope here was that MMIO BAR0 might be a back-channel to PCIe config state that bypasses the config-space write protection. On GA100, PCIe config space is mirrored at BAR0 offset `0x88000` (confirmed: reading `nvapeek -c 1 0x88000` returns `20c210de` = device `0x20c2` + vendor `0x10de`).

The PCIe Express Capability sits at PCIe config offset `0x60` (confirmed by `lspci -vvv` header `Capabilities: [60] Express`), which maps to BAR0 MMIO offset `0x88060`. The relevant register offsets are:

| Register | BAR0 MMIO offset | Purpose |
| -------- | ---------------- | ------- |
| Device Ctrl/Status | `0x8806c` | Device-level config (not link-related) |
| LnkCap             | `0x88070` | Link capabilities |
| LnkCap2            | `0x8808c` | Supported Link Speeds Vector |
| LnkCtl2            | `0x88090` | Target Link Speed |

Reading these via envytools' `nvapeek`:

```bash
sudo nvapeek -c 1 0x88000   # Vendor/Device → 20c210de (valid)
sudo nvapeek -c 1 0x8806c   # Device Ctrl/Status → fee00078 (readable)
sudo nvapeek -c 1 0x88070   # LnkCap             → ... (blocked)
sudo nvapeek -c 1 0x8808c   # LnkCap2            → ... (blocked)
sudo nvapeek -c 1 0x88090   # LnkCtl2            → ... (blocked)
```

The `...` output from `nvapeek` indicates the read was suppressed or returned zero. Non-link registers in the same PCIe config region (Vendor/Device ID, Device Ctrl/Status) read back normally. Every link-configuration register returned no data.

**Result: LINK-RELATED REGISTERS ARE PROT-PROTECTED.** NVIDIA's PROT (protection level mask) on GA100 specifically walls off `LnkCap`, `LnkCap2`, and `LnkCtl2` from host MMIO access — both reads and writes. Non-link registers in the same 4KB page remain host-accessible. This is not an accident of MMIO layout — the protection boundary is drawn precisely around the registers we would need to modify.

GA100 is the full datacenter-tier PROT configuration, which is stricter than consumer Ampere. The protection applies to both MMIO BAR0 access and standard PCIe config cycles (tested above in Path 1).

**Path 4 — Falcon PRIV Bus Address Direct Access**

VBIOS disassembly (via `envydis -m falcon -V fuc5`) revealed bit-manipulation code on register `0x14118f78`:

```
ld b32 $r9 D[$r13]      ; read 0x14118f78
mov $r15 0xbfffffff     ; mask clearing bit 30
and $r9 $r15
st b32 D[$r13] $r9      ; write back
```

This register is accessed 20+ times in the VBIOS devinit with various mask operations involving `0x3000`, `0x2000`, `0xc000`, `0x8000` — bit patterns consistent with link speed/capability configuration.

However, `0x14118f78` is in the **Falcon internal PRIV bus address space**, not host BAR0. BAR0 on GA100 is 16 MB (mapped at host physical `0x93000000`–`0x94000000`); address `0x14118f78` would require a 336 MB BAR0 which does not exist on this card. The PRIV bus is the address space a Falcon microcontroller *inside* the GPU sees while executing its own microcode — it is not directly mapped to the host.

**Result: NOT HOST-ACCESSIBLE.** To execute the `and 0xbfffffff` bit-clear instruction against register `0x14118f78`, code must run on a Falcon. Running code on a Falcon in Heavy Secure mode — the mode required to touch PCIe configuration — requires cryptographically signed microcode loaded at boot. Signing requires the Ampere Falcon bypass, which does not exist publicly.

**Why Runtime RAM Patching Also Fails**

A tempting middle path is to let Falcon load and verify the signed firmware from the EEPROM, but patch it in system RAM between validation and execution. This is conceptually similar to the Toyota SecOC bootloader exploit.

On Ampere, this is specifically closed. Falcon validates microcode **during** the DMA transfer from system RAM into its instruction memory (IMEM), not as a separate step afterward. There is no TOCTOU window between "loaded" and "verified" — the signature check and the load happen in the same DMA operation. Modified bytes in system RAM are detected and the firmware is rejected before any patched instruction executes.

The [nova-core](https://docs.kernel.org/gpu/nova/core/firmware/secure-boot.html) documentation describes this explicitly: on Ampere, HS ucode is "verified by the Falcon's Boot ROM" as part of the load process, not after.

**Summary of Attempts**

| Path | Approach | Result |
| ---- | -------- | ------ |
| 1 | `setpci` write to LnkCap2 | Silently rejected (hardware read-only) |
| 2 | Upstream root port retrain | No change (endpoint still advertises Gen1) |
| 3 | `nvapeek`/`nvapoke` on MMIO-mapped PCIe config | Link registers PROT-protected, reads return zero |
| 4 | Direct access to Falcon PRIV register `0x14118f78` | Not in host BAR0 address space |
| — | RAM patching between validation and execution | TOCTOU window closed by DMA-time signature check |

**What This Rules Out**

Every known software-only avenue from Linux userspace has been eliminated:

* No PCIe config-space write path works — the registers are hardware read-only
* No MMIO BAR0 backdoor exists — the link registers are PROT-protected from both paths
* No retrain loop or negotiation trick works — the endpoint's TS ordered sets are firmware-controlled
* No Falcon PRIV address is directly reachable from the host
* No firmware RAM patching path works — signature validation happens inside the DMA

The protection is layered: multiple independent mechanisms each reject the attack, so defeating any single layer would still leave the others in place.

**What Remains Possible**

Three realistic paths still exist, none of which are a quick weekend project:

1. **PCIe retimer with TS modification** — a signal-conditioning chip physically between the motherboard and the card that rewrites the Rate ID byte in the endpoint's TS1/TS2 ordered sets before forwarding them to the Root Complex. Candidate silicon includes Astera Labs Aries and Texas Instruments DS160PR810. This operates below the register level entirely and has no firmware dependency, but requires a custom PCIe interposer PCB.

2. **Ampere Falcon signing bypass** — the master key. Would enable arbitrary VBIOS modification, which would unlock PCIe speed, FMA throttle, and every other firmware-enforced restriction simultaneously. Active research problem with no public progress (see [Open Research Problems](open-research-problems.md)).

3. **A yet-undiscovered hardware vulnerability** — side-channel attacks on signature verification, decapsulation and direct-probe attacks, or an architectural flaw we don't yet know about. Realistically this is multi-year security research territory.

---

**Why Real-Time VBIOS Patching During Boot Is Not Viable**

A theoretical approach has been proposed: use an FPGA or microcontroller-based MUX to intercept the VBIOS during the boot sequence, patching it on-the-fly before Falcon reads it. This is **not viable** for the following reasons:

**Timing Constraint:** The gap between GPU power-on and Falcon signature validation is ~21.9 milliseconds. Even the fastest VBIOS modifications require pre-calculation offline (at least several milliseconds per patch). Real-time LLM-assisted patching introduces 50–100ms network latency alone, which already exceeds the available window.

**Signature Coverage:** The entire VBIOS is signed, including the section headers and configuration. Patching individual bytes requires regenerating the RSA-3072 signature, which requires the private key. Pre-staging a complete modified image bypasses the timing issue but still requires the signature key.

**Complexity:** Even with the timing solved, the MUX interposer must:
1. Detect the exact boot moment (not straightforward without probe access to internal buses)
2. Switch VBIOS source to a pre-staged Golden Image
3. Complete the switch before Falcon begins the signature check
4. Not corrupt the image during handoff (high-speed SPI at 1.8V is signal-integrity sensitive)

**Practical Result:** The most realistic approach would be pre-calculating the patched image *offline*, then using an interposer to swap the VBIOS source at boot time — but this still requires either signing capability or a complete bypass of signature validation (not currently available).

**Conclusion:** Real-time LLM-assisted patching is impractical. VBIOS modification remains blocked by signature validation until a cryptographic breakthrough or Falcon exploit is found.

---

**How to Reproduce the Tests**

For anyone who wants to verify these findings on their own card (or repeat with a newer driver or different VBIOS version), here is the minimal test sequence:

```bash
# 1. Identify your card's PCIe address
lspci | grep -i "CMP 170HX\|GA100"
# Should show something like: 21:00.0

# 2. Confirm the lock from both directions
sudo lspci -s 0000:21:00.0 -vvv | grep -A5 "LnkCap\|LnkSta"
sudo setpci -s 0000:21:00.0 CAP_EXP+2C.L
# Expect: 00000002

# 3. Attempt setpci write
sudo setpci -s 0000:21:00.0 CAP_EXP+2C.L=0000007e
sudo setpci -s 0000:21:00.0 CAP_EXP+2C.L
# Expect: 00000002 (write dropped)

# 4. Check MMIO access via envytools
# Build envytools first: https://github.com/envytools/envytools
sudo nvapeek -c 1 0x88000   # should return vendor/device
sudo nvapeek -c 1 0x8808c   # should return ... (PROT-blocked)
```

If your card shows any different result — for example, LnkCap2 returning a non-`0x00000002` value, or `nvapeek` returning something other than `...` at offset `0x8808c` — please open an issue on the project's GitHub. A card behaving differently is the first clue that might unlock the problem.

**Credits**

Testing performed on an HP Z4 G4 Workstation with a CMP 170HX at PCIe address `0000:21:00.0`, driver 595.58.03, VBIOS 92.00.6D.00.0A, April 2026. VBIOS disassembly performed with [envytools](https://github.com/envytools/envytools) `envydis -m falcon -V fuc5`.
