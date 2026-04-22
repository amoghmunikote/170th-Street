# What Has Been Bypassed — and What Hasn't

**What Has Been Bypassed — and What Hasn't**

This page documents the current state of NVIDIA firmware security research as it applies to the CMP 170HX, separating confirmed achievements from ongoing open problems.

**Confirmed Bypasses**

**FMA Throttle — Bypassed at the Application Layer**

The FMA throttle is the most impactful restriction and it has been effectively bypassed — not at the firmware level, but at the application level. By preventing FMA instructions from being generated in the first place (via `#pragma OPENCL FP_CONTRACT OFF`, `-fmad=false`, or PoCL patching), software can avoid the throttled instruction entirely. The hardware executes unthrottled non-FMA FP32 at 6.25 TFLOPS, delivering a 16× improvement over the throttled 0.39 TFLOPS.

This bypass was first confirmed publicly in December 2023 by community researcher jetcat8848 in the dartraiden/NVIDIA-patcher GitHub repository (Issue #73), and validated by the 2025 arXiv paper (2505.03782). It is now documented, reproducible, and widely used.

**Important:** this is not a firmware bypass — it is a software-level workaround that avoids the throttled instruction rather than removing the throttle. The firmware restriction remains in place; the workaround simply routes around it.

**Driver-Level Patches — NVIDIA-patcher (dartraiden)**

The [dartraiden/NVIDIA-patcher](https://github.com/dartraiden/NVIDIA-patcher) project patches the NVIDIA Linux and Windows driver to add 3D acceleration support for mining cards including the CMP 170HX. The standard NVIDIA driver refuses to enable 3D acceleration (CUDA, OpenCL, Vulkan) for mining-designated cards. The patcher modifies the driver binary to recognize these cards and enable full compute support.

This is a driver-level modification, not a firmware modification. It does not touch the VBIOS and does not require bypassing Falcon. It patches the userspace/kernel driver code that checks device IDs before enabling features.

The patcher supports the CMP 170HX explicitly and is maintained through regular driver version updates. Note: newer driver versions (576.80+) have shown issues with the CMP 170HX — driver version 471.50 (Windows) and the equivalent Linux datacenter driver are the most stable as of mid-2025.

**Cross-Flashing Between Signed VBIOSes — How the Tools Actually Work**

OMGVflash and NVflashk enable flashing of any signed, valid NVIDIA VBIOS onto any compatible card, bypassing the subsystem ID checks that standard nvflash enforces. This is how we flashed `92.00.6D.00.0A` onto our card in place of the original `92.00.67.00.01`.

Both tools are modified builds of NVIDIA's stock `nvflash.exe` (v5.780 base for OMGVflash, up to v5.814.0.k1 for nvflashk). Kefinator's nvflashk README describes the mechanism directly: *"I managed to locate a backdoor of sorts that nVidia implemented to have a 'mismatch bypass', and I have forced that bypass to be enabled at all times when using the -6 parameter."* The `-6` flag has existed in nvflash for years as a PCI Subsystem ID mismatch override; the patched builds simultaneously force bypass paths for:

* PCI Subsystem ID mismatch (the original `-6` behavior)
* Board ID mismatch (the fused PCB+GPU ID that stock nvflash refuses to override from the command line)
* Hierarchy ID mismatch
* InfoROM and XUSB-FW downgrade locks
* Golden-card VendorCert/XOC/MasterCert checks

OMGVflash goes slightly further by sending MUTEX commands directly to Falcon via `powrprof.dll` wake-up, allowing InfoROM/XUSB FW downgrades even when newer VBIOSes have locked parts of the EEPROM. The canonical invocation for a CMP 170HX is `omgvflash --protectoff -6 filename.rom`.

**What these tools cannot do:** flash a modified, unsigned, or custom-authored VBIOS. The bypass exists entirely in host-side validation in `nvflash.exe`; the Falcon HS bootrom still validates the firmware cryptographically at next POST and rejects any tampered ROM. Both authors document this limit explicitly in their repos. Veii's roadmap: *"3000/4000 will require signed bioses... Later we can start and focus on Ampere and Ada"* — that "later" has not arrived.

**What Has Not Been Bypassed**

**Ampere VBIOS Signature Check — Not Bypassed**

No publicly known method exists to flash modified unsigned firmware onto an Ampere GPU and have it boot. OMGVflash's Falcon access breakthrough was specific to Turing. The developer Veii explicitly stated that Ampere's certificate chain validation is a distinct, stronger mechanism that has not been cracked. Flashing a byte-modified VBIOS to the CMP 170HX via any method including SPI programmer results in a non-booting card.

**PCIe Gen 1 Speed Lock — Not Bypassed at Firmware Level**

The PCIe speed lock is enforced in the signed VBIOS. Without a firmware modification, the card cannot negotiate above Gen 1 speed regardless of host slot capabilities.

Every known software-only runtime path has been independently tested and eliminated — `setpci` write to LnkCap2 is rejected, MMIO BAR0 access to the link registers is PROT-protected on GA100, upstream root port retrains don't change what the endpoint advertises, and the Falcon PRIV register that manipulates the speed bits is not host-addressable. The full test methodology and results are documented in [Runtime PCIe Speed Unlock — Attempted & Failed](runtime-pcie-unlock-attempt.md).

**Primary PCI Device ID Reprogramming — Not Demonstrated**

The CMP 170HX's primary PCI Device ID remains fixed as `10DE:20C2` in every publicly documented production-card experiment so far. By contrast, the PCI Subsystem ID is clearly more flexible: cross-flashed signed VBIOSes and engineering-sample firmware work show that subsystem identity can change without altering the silicon's underlying primary Device ID. In other words, the community has demonstrated **rebranding at the subsystem/firmware layer**, but not **arbitrary reprogramming of the GPU's main Device ID**.

The leaked A100 GA100-883 P1001-B02 schematic suggests why. On schematic page 15, the strap group includes `VGA_DEVICE`, `PCIE_CFG`, and `DEVID_SEL`, controlled by `STRAP3`, `STRAP4`, and `STRAP5`. This shows that Ampere boards still use strap resistors for some early-boot configuration, just as older NVIDIA boards did. But the available evidence does **not** support the older Pascal-era model where the full Device ID is simply encoded by a resistor pattern on the PCB and can be rewritten arbitrarily by moving parts around.

The most likely interpretation is narrower: `DEVID_SEL` appears to be a **selector bit** between predefined identities such as "original" and "rebrand," while the actual primary Device ID exposed on the bus is still constrained by silicon state, fuse state, or secure early-boot firmware. That interpretation fits three independent observations:

* engineering-sample reports indicate the Subsystem ID can be changed while the primary Device ID stays hardware-fixed
* the schematic names the strap `DEVID_SEL`, not a multi-bit full PCI ID field
* modern NVIDIA secure boot pushes much of device configuration into signed devinit/Falcon-controlled initialization rather than leaving it entirely to passive board straps

So, **could `DEVID_SEL` distinguish A100 versus CMP 170HX?** Possibly, but this remains unproven. It is a reasonable inference that A100 corresponds to an "original" identity and CMP 170HX to a "rebrand" identity, but no public strap-swap experiment on a retail 170HX has yet shown the card enumerating with an A100-class primary Device ID after changing only the resistor population.

The same caution applies to memory straps. The schematic confirms that `STRAP0`–`STRAP2` select HBM vendor/package configurations, but that does not by itself prove that unused physical capacity can be surfaced just by changing resistor states. Usable VRAM capacity may also depend on firmware training tables, memory init code, and possible fuse limits. Claims that an undocumented strap pattern would expose 32 GB or 40 GB on a 170HX are therefore still speculative.

**Current bottom line:** the primary Device ID appears to be **hardware-rooted and not publicly bypassed**. The strap network likely participates in selecting between a small number of board personas, but there is no confirmed method today to turn a production CMP 170HX into an arbitrary PCI Device ID by resistor changes alone.

**FMA Throttle at Firmware Level — Not Bypassed**

The FMA throttle persists in the firmware. The application-layer workaround routes around it but does not remove it. Any workload that cannot be modified to avoid FMA instructions (standard binaries, closed-source software) will still encounter throttled FP32 performance.

**NVLink — Not Enabled**

Even if the hardware were fully populated (see the NVLink Population page), whether the firmware would activate it is unknown and currently unverifiable without a firmware modification capability.
