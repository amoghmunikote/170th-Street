# Open Research Problems

**Open Research Problems**

This page documents the active open problems in CMP 170HX security and firmware research. These are the unsolved challenges that, if cracked, would be the most impactful contributions to the project.

**Problem 1 — Ampere Falcon Signature Bypass**

**Impact if solved:** Full firmware modification capability. Would enable removing the FMA throttle, PCIe speed lock, and any other firmware-enforced restriction directly.

**Current state:** No public progress. The Turing bypass (OMGVflash) exploited a specific vulnerability in Falcon's authentication path for that generation. Ampere uses a certificate chain validation approach that the same techniques do not apply to. Veii (OMGVflash author) has indicated Ampere requires a fundamentally different approach and that "going around it is pretty pretty impossible" without knowing how to reprogram Falcon itself.

**Research directions:**

* **GA100 V2 descriptor analysis (most actionable):** GA100 uses FalconUCodeDescV2, not V3 (see [The Falcon Security Architecture](the-falcon-security-architecture.md)). V2 has received far less public scrutiny than V3 and places the CMP 170HX closer to Turing's signing infrastructure — the generation that OMGVflash ultimately exploited. Dumping the Type 0xE0 FwSec partition from a 170HX ROM and diffing byte-for-byte against a signed A100 40GB PCIe ROM (identical PCB, different fuse state) using `envydis`/`ghidra_falcon` may reveal whether fuse enforcement lives in patchable firmware or in the SM hardware datapath.
* Analysis of Falcon microcode for Ampere — NVIDIA has published some Falcon documentation but key implementation details are proprietary
* Side-channel analysis of the signature verification process
* Review of NVIDIA security bulletins for Ampere firmware vulnerabilities (CVE database)
* Hardware-level analysis (requires decapsulation of the GPU package — extreme difficulty)
* **Comparative trace against A100:** Anyone with access to both a CMP 170HX and a real A100 can dump and compare Falcon devinit code to identify the single conditional branch or constant that decides between "advertise Gen 1" and "advertise Gen 4" — which would directly inform targeted exploit development. The register `0x14118f78` identified in [Runtime PCIe Speed Unlock — Attempted & Failed](runtime-pcie-unlock-attempt.md) is the likely target; comparing the mask values at that address between a 170HX trace and an A100 trace would localize the enforcement.

**Problem 2 — FMA Throttle Mechanism Identification**

**Impact if solved:** Better understanding of exactly where and how the throttle is enforced, potentially revealing alternative bypass paths.

**Current state:** The throttle is widely believed to be OTP fuse-enforced at the silicon level rather than firmware-enforced. niconiconi's fuse analysis notes: *"The strategy was to use fuses to limit the crucial instructions, particularly the FP32 FMA and MAD instructions... Unless of course the fuses aren't really fuses at all and can be reprogrammed with some stolen software. For some chips I worked on there were OTP ROM IPs which were blown at production test. No way to undo that."* The FP32 MAD/FMA throttle ratio is exactly 16:1 (0.395 TFLOPS with FMA vs. 6.285 TFLOPS without), first quantified in December 2023 by jetcat8848 (dartraiden/NVIDIA-patcher Issue #73). Whether enforcement is at the fuse-gated SM datapath or in early boot firmware that configures an SM register is not definitively established — the application-layer workaround routes around the throttled instruction entirely, so the question has been deprioritized.

**Research directions:**

* Differential analysis of NVIDIA driver binaries across versions — does the throttle change across driver versions?
* Analysis of the NVIDIA Linux open-source kernel module (partially released by NVIDIA in 2022) for any per-device instruction throttling logic
* Testing whether the datacenter driver (used for A100) applied to the CMP 170HX produces different FMA behavior

**Problem 3 — PCIe Gen 1 Lock**

**Impact if solved:** 4× improvement in PCIe bandwidth (after capacitor mod) or 64× if Gen 4 x16 can be reached.

**Layer 2 (PCB capacitor omission) — SOLVED** Confirmed April 2026 by Amogh Munikote. Populating the 24 missing 0402 0.22µF AC coupling capacitors on the PCIe lanes upgrades the link from x4 to x16 width. Full details and verified results on the PCIe Capacitor Mod page.

**Layer 1 (firmware Gen 1 speed lock) — Open**

Every known software-only path from Linux userspace has been independently tested and eliminated. The full methodology and results are in Runtime PCIe Speed Unlock — Attempted & Failed. Summary:

* `setpci` write to LnkCap2 → silently rejected (hardware read-only)
* MMIO BAR0 access to link registers at `0x88070`/`0x8808c`/`0x88090` → PROT-protected, reads return zero, writes ignored
* Upstream root port retrain via `pcie_set_speed.sh` → no change, endpoint re-advertises Gen 1 in its TS1/TS2 ordered sets
* VBIOS disassembly identified register `0x14118f78` as the PCIe link speed configuration register manipulated by Falcon devinit, but this address is in the Falcon internal PRIV bus space and is not in host BAR0 aperture
* RAM patching between firmware signature validation and execution → closed by DMA-time verification on Ampere

The protection is layered: multiple independent mechanisms each reject the attack, so defeating any single layer still leaves the others in place. This is stricter than consumer Ampere — GA100 ships with the full datacenter PROT configuration.

**Remaining research directions:**

* PCIe retimer with TS modification — a signal-conditioning chip physically on the PCIe lanes that rewrites the endpoint's TS1/TS2 Rate ID field before forwarding to the Root Complex. No firmware dependency. Candidate silicon: Astera Labs Aries, TI DS160PR810. Requires custom PCIe interposer PCB. Most actionable path that exists today.
* Ampere Falcon signing bypass — solving this unlocks PCIe speed via firmware modification, but also unlocks everything else (see Problem 1).
* Comparative testing with a real A100 — anyone with access to both cards could dump and compare the Falcon devinit code to identify the single conditional branch or constant that decides between "advertise Gen 1" and "advertise Gen 4", which would inform targeted exploit development.

**Problem 3B — Memory Unlock Register Access Strategy**

**Current state — partially solved, critical insight emerged**

Recent community testing identified a critical issue in BAR0 register access that has blocked memory unlock progress:

* The indirect engine (GPU's MMIO indirection through Falcon registers) **truncates 32-bit addresses to 16-bit**, causing writes to registers like `0x9A0204` to land at `0x0204` instead — resulting in writes to wrong MMIO offsets
* **Direct Falcon register writes work correctly** and reach the full 24-bit BAR0 address space
* Memory geometry registers (`FBPA_CFG1` at `0x9A0204`, `LMR` at `0x100CE0`, `CSTATUS`) must be written with matching values for GSP-RM to accept the configuration
* Mismatched geometry values cause GSP-RM to revert the configuration back to locked values during boot
* Persistence model refined: BAR0 writes survive driver reload but not DEVINIT or power cycles

**Remaining research directions:**

* Determining which Falcon register write path is safe to use without causing GSP initialization failures
* Understanding why direct Falcon writes sometimes result in uninitialized driver state
* Validating memory unlock on full power cycle (vs. driver reload) — requires persistent firmware modification or repeated exploit execution at each boot

**Problem 4 — Intermediate Success Measures for ROP Chain Development**

**Impact if solved:** Would enable faster iteration on ROP chain construction without requiring full exploit execution each time.

**Current state:** The ROP chain development has insufficient intermediate validation steps. Researchers have identified a fundamental blocker: each full exploit attempt is required to determine if the ROP chain is correct, with no way to validate partial progress. Additionally:

* Falcon canary values are **randomized each boot**, making it impossible to exfiltrate all canary values in a single exploit run — the canary used to corrupt memory is unknown until the exploit either succeeds or triggers the panic handler
* Multiple stack frame levels and register states must be guessed simultaneously for a successful chain — if any single guess is wrong, the entire chain fails
* The final exit strategy (how to cleanly return control to GSP after ROP payload execution) remains **unclear and untested** — does the chain need to restore the call stack, or can it safely jump to a panic loop?

**Remaining research directions:**

* Developing an instrumented Falcon emulator that can trace ROP chain execution without requiring actual GPU hardware
* Creating intermediate validation measurements (e.g., single-instruction gadget tests) that don't require full exploit success
* Understanding GSP behavior after the ROP payload completes — what happens if execution jumps to an unexpected address?

**Problem 5 — Driver Compatibility with Newer Drivers**

**Impact if solved:** Access to newer CUDA features, better software compatibility, security fixes.

**Current state:** The dartraiden patcher has shown instability with newer NVIDIA driver versions on the CMP 170HX. Driver 471.50 (Windows) and equivalent Linux datacenter drivers are the most stable. Newer drivers (576.80+) have reported issues including misidentification as a "3080 Laptop GPU" and error code 43.

**Research directions:**

* Community testing of intermediate driver versions to identify when the regression was introduced
* Contribution to dartraiden/NVIDIA-patcher with updated patches for newer driver versions
* Investigation of whether the misidentification can be corrected via INF/device ID patches

**Problem 6 — Tensor Core Full Activation**

**Impact if solved:** Access to full theoretical Tensor Core throughput. Current unlock status: tensor compute has been improved to ~200 TFLOPS (from 6.3 TFLOPS), but theoretical maximum (~312 TFLOPS FP16) remains elusive.

**Current state — partially unlocked, limitations identified**

Microbenchmarking research (Zenodo 19002983, March 2026) has definitively characterized the Tensor Core throttle mechanism. Key findings:

* A single MMA instruction executes with a **256 fixed-cycle latency** that cannot be hidden by ILP or pipeline overlap
* Only **4 warps per SM** can simultaneously issue Tensor Core instructions
* These two constraints together produce exactly the measured 6.3 TFLOPS — the theoretical model closes perfectly
* The mechanism is **dispatch-level hardware gating** — confirmed by ILP scaling, warp scaling, dependency chain, and cross-pipeline interference experiments
* The Tensor Core execution units are physically intact — it is the dispatch gate that limits throughput, not damaged or disabled hardware

**Why this is harder to bypass than the FMA throttle**

The FMA throttle was bypassed at the compiler level — by preventing FMA instructions from being generated, the throttled instruction path is simply never invoked. The Tensor Core dispatch gate operates differently: the execution units are present, the instructions are accepted, but the hardware gate limits how many warps can issue them and enforces a fixed latency regardless of pipeline state. There is no compiler-level equivalent of `-fmad=false` for a dispatch-level gate. Bypassing it likely requires either firmware modification (to remove the gate) or a microarchitectural technique that has not yet been identified.

**Research directions**

* Analysis of whether the dispatch gate is enforced in Falcon microcode loaded at boot, in the VBIOS power table, or in a hardwired GPU register — the paper identifies it as hardware gating but the exact enforcement point in the boot chain is not established
* Testing whether any version of the A100 datacenter driver changes the warp issue limit or MMA latency on the CMP 170HX
* PTX-level experiments testing alternative Tensor Core instruction forms (WMMA vs MMA vs warp-level primitives) to see if any path avoids the gate
* Cross-referencing with the Ampere Falcon bypass problem — if custom firmware can be loaded, the dispatch gate configuration may be modifiable

**Contributing to Research**

The most effective contributions right now are:

* **Binary analysis skills:** NVIDIA driver and firmware binary analysis to locate throttle logic
* **Falcon expertise:** Anyone with Falcon microarchitecture knowledge or prior GPU firmware security research experience
* **Systematic testing:** Methodical testing of driver versions, CUDA versions, and compute modes to narrow down where restrictions are enforced
* **Hardware access:** Anyone with both a CMP 170HX and a working A100 can perform comparative analysis that most researchers cannot

All findings should be documented publicly — in issues on dartraiden/NVIDIA-patcher, in the 170th Street community, or as standalone publications. The FMA workaround itself was discovered by a single community researcher posting a code diff to a GitHub issue. The next breakthrough could come the same way.
