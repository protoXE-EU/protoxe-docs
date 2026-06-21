# protoXE: Differential Validation as a First-Class Architecture for Pre-Silicon and High-Assurance Verification

**Author:** Claudio Corbetta
**Affiliation:** protoXE
**Date:** June 2026
**License:** Creative Commons Attribution 4.0 International (CC-BY 4.0)
**Contact:** claudio.corbetta@protoxe.com · protoxe.com

---

## Abstract

Existing tools for pre-silicon software verification — QEMU, gem5, and commercial EDA emulation platforms — were not designed for high-assurance engineering. They are non-deterministic, host-dependent, expensive, or simply unable to run in the bare-metal and microkernel environments that certified firmware and hypervisors actually target. We present protoXE, a full-system emulator for ARM64 and RISC-V built in pure Rust with zero runtime heap allocations, organized around a single architectural principle: every component, at every fidelity level, must be differentially validated against an external reference. We describe the platform's design, its three-tier validation methodology (real silicon, an authoritative external reference, and a fast in-house oracle), and report empirical results: over 650,000 instructions cross-validated against real QEMU with zero divergences across both instruction set architectures, a fully validated Type-1 hypervisor substrate at EL2, and four real implementation defects found and corrected purely through the oracle methodology — including one that was silently present in QEMU itself. We argue that this approach — architecture built around the oracle, not validation bolted on afterward — is what makes a deterministic, sovereign emulator viable for certified, mission-critical engineering.

---

## 1. The Problem

Teams building certified firmware, Type-1 hypervisors, secure-world (TrustZone/TEE) software, or RTL IP face a structural gap in the pre-silicon verification toolchain.

**QEMU** is the de facto standard for software-level ARM64/RISC-V emulation, but it was never designed for high-assurance use. It depends on a host operating system, standard libc, and POSIX threading — none of which are available inside the bare-metal or microkernel environments that certified hypervisors and secure firmware actually run on. Its dynamic binary translation introduces execution timing that varies run to run, which makes certain classes of bugs — race conditions, interrupt-timing defects, anything that depends on exact instruction ordering — effectively unreproducible. And because QEMU's correctness is validated primarily through functional and conformance testing rather than continuous comparison against real hardware, latent decode and semantic defects can and do survive for years undetected.

**gem5** offers far higher microarchitectural fidelity but at a cost in speed and complexity that makes it impractical for firmware bring-up or production verification workflows; it is built for computer-architecture research, not for shipping certified software.

**Commercial EDA emulation platforms** (Synopsys, Cadence, Siemens EDA) solve determinism and fidelity at significant cost — both financial and in dependency on non-EU proprietary toolchains, hardware emulation ASICs, or FPGA prototyping infrastructure. None of them are designed to be qualified as a tool under DO-178C or ISO 26262 by a small team, and none of them are sovereign infrastructure in the sense increasingly required by European institutional and defense procurement.

The result: there is no widely available tool that is simultaneously **deterministic**, **bare-metal capable**, **continuously validated against real hardware**, and **open enough to be certified**. protoXE was built to close that gap.

---

## 2. Design Principles

protoXE is built around a small number of non-negotiable invariants.

**Zero allocation in the simulation hot path.** The system topology — CPU, memory, peripherals, bus fabric — is fixed at compile time through Rust's generic type system. No heap allocation occurs anywhere in the instruction execution path. This is not a performance optimization; it is the precondition for the platform's two defining capabilities. Because the entire machine state is a single, statically-sized, contiguous memory region, snapshot and restore reduce to a single `memcpy`, with no serialization and no pointer fixup. That is what makes whole-machine, instruction-granular reverse debugging tractable without the overhead that record/replay normally carries (full technical disclosure of this mechanism is publicly available — see References).

**Determinism is a platform property, not a test-time concern.** Given identical initial state and instruction stream, protoXE produces bit-identical output on every run, on any host. This is what makes timing-dependent bugs — race conditions between cores, interrupt-injection ordering defects — reproducible on demand rather than observed once and never again.

**Differential validation is the organizing architectural principle, not an afterthought.** Rather than writing an emulator and then writing tests for it, protoXE's architecture is built around the assumption that every component must be checkable against an external reference at its interface boundary — and that the cheapest way to find a defect is to compare two independent things and look for where they disagree.

---

## 3. Validation Methodology

protoXE's approach to correctness rests on a three-tier reference hierarchy, used in combination rather than in isolation.

### 3.1 Real silicon as the ground truth

For ARM64, a commodity reference board (Rockchip RK3588S, a heterogeneous octa-core Cortex-A76/A55 SoC) serves as the silicon-as-reference. Instruction streams are executed identically on protoXE and on the physical board; architectural state — general-purpose registers, program counter, and selected system registers — is compared after execution. A divergence is unambiguous: it is either a defect in the emulator or a documented silicon erratum, and it is the only validation tier that cannot itself be wrong about what real hardware does.

### 3.2 An authoritative external reference

QEMU serves as a second, independent reference, used specifically because it is developed independently of protoXE and therefore fails differently. For RISC-V, protoXE drives a real `qemu-system-riscv64` process via its GDB remote protocol, sets identical architectural state on both sides, single-steps, and compares full register state after every instruction. The same technique is applied to `qemu-system-aarch64` for ARM64.

### 3.3 A fast, sovereign in-house oracle

A second, independently developed (clean-room, zero shared code) reference implementation of the instruction set provides a fast, dependency-free oracle suitable for continuous integration and broad coverage sweeps — millions of instructions in minutes, with no external tooling required. Its limitation is explicit and was discovered empirically (see §5): two independent implementations can still share an error if both misread the same specification passage. This is precisely why the in-house oracle and the external QEMU reference are used together rather than as substitutes for one another — each catches a class of defect the other cannot.

### 3.4 Lock-step verification against RTL

The same methodology extends to hardware. A CPU-under-test compiled to a cycle-accurate model via Verilator (open source) is run in lock-step against the same software golden reference, with state compared after every retired instruction and any divergence automatically reduced to a minimal reproducing instruction sequence via delta-debugging (the ddmin algorithm). A separate, pure-software co-simulation bridge allows an arbitrary RTL peripheral or CPU core to be mapped onto the system bus and exercised by real firmware — with no FPGA, no proprietary emulation hardware, and no simulation engine beyond the host CPU. Both mechanisms are implemented and operational; the underlying methodology has been placed in the public domain as prior art (see References, Disclosures 2 and 3).

---

## 4. Empirical Results

### 4.1 Cross-ISA validation against QEMU

| Validation layer | Coverage | Result |
|---|---|---|
| RISC-V RV64IMAC vs. QEMU | 580,000 instructions, 3 random seeds | Zero divergences |
| ARM64 integer + NEON/FP vs. QEMU | 64,000 instructions | Zero divergences |
| ARM64 exception model vs. QEMU | 1,500 cases — FP trap, SVC, ESR/ELR/SPSR | Zero divergences |
| ARM64 MMU vs. QEMU | 6 scenarios — page walk, permission/access-flag faults | Zero divergences |
| ARM64 EL2/EL3 vs. QEMU | Entry/ERET, SMC routing, OP-TEE SMC ABI | Zero divergences |

Across both instruction set architectures, protoXE has been cross-validated against real QEMU on more than 650,000 instructions and exception/system-level scenarios with zero observed divergences.

### 4.2 Real-silicon validation

Against the physical RK3588S reference board, protoXE's GIC and debug UART models match the real device tree bit-for-bit, and nine distinct CPU/SoC behavioral facts have been closed bit-exact against silicon — the physical core-numbering scheme, peripheral base addresses, interrupt count, and clock-controller register behavior among them.

### 4.3 Defects found and corrected by the methodology, not by inspection

The value of differential validation is best demonstrated by what it actually catches. Four real implementation defects were found this way:

**A silently incorrect address-translation instruction.** During OP-TEE boot validation, the `AT S1E1R` instruction (used by secure-world software to verify that a non-secure address is genuinely mapped) was being decoded as a no-op cache-maintenance operation, leaving the result register permanently zero. The defect was present in QEMU as well — a textbook example of why comparison against real silicon is necessary even when an emulator already exists as a reference: two independently maintained emulators agreeing with each other is not evidence of correctness.

**An illegal instruction encoding accepted by two independent implementations.** A RISC-V test generator was emitting an invalid `SRAI`/`SRAIW` encoding. protoXE's interpreter and its independently written in-house reference both decoded the malformed encoding the same (incorrect) way and therefore agreed with each other — only the authoritative QEMU reference rejected it as illegal, exposing the generator defect and confirming, empirically, the precise failure mode that motivates running an external reference alongside an in-house one.

**A misreported hardware capability register.** The GICv3 virtual-interrupt capability register (`ICH_VTR_EL2`) was advertising the architectural maximum of 16 list registers with a reserved value in an adjacent field, rather than the four list registers the target core actually implements. Cross-checked against the Cortex-A76/A55 Technical Reference Manual and corrected to be silicon-exact — a defect that would have silently corrupted any hypervisor relying on the advertised capability to size its own data structures.

**An incorrect assumed core topology.** The big.LITTLE physical CPU numbering was assumed to place the high-performance cores first; the real device tree pulled from the reference board showed the opposite — the efficiency cores are numbered first on real silicon. Corrected once observed; a class of defect that is invisible without a real device tree to compare against.

In every case, the defect was localized to the exact failing instruction or register access by the methodology itself, not discovered through code review.

### 4.4 Hypervisor substrate, validated end-to-end

protoXE implements the architectural substrate required to host a Type-1 hypervisor at EL2 — second-stage memory isolation between guest partitions, hypervisor trap routing for privileged guest operations, and virtualized interrupt delivery. The complete lifecycle has been validated end-to-end with a hand-written EL2 payload that exercises partition isolation, a deliberate access to an unmapped guest-physical address (correctly trapped to the hypervisor), a guest yield via `WFI` (correctly trapped and resumed), and a hypervisor-injected virtual interrupt correctly received, acknowledged, and retired by the EL1 guest — a full 52-instruction lifecycle with no external payload, no QEMU, and three independent in-memory witnesses confirming every stage. We describe the substrate's capability and validation results here; the implementation mechanism is documented separately and is, at the time of writing, available to Design Partners under NDA.

---

## 5. Architecture and Long-Term Direction

protoXE's near-term capability is a fixed, statically composed machine model: an ARM64 or RISC-V core, memory, and a defined set of peripherals, all wired at compile time. Its longer-term direction is a fully composable hardware co-design platform organized around the same validation principle: bring your own CPU, memory, or peripheral model; substitute a single component inside an existing, silicon-validated board and test it *in context*; and validate the whole assembly against a reference, up through a real software stack.

The RTL co-simulation bridge described in §3.4 is the first concrete step in that direction — it already allows a designer's own Verilog or SystemVerilog block to be dropped into a silicon-proven SoC model and exercised by real firmware, with the surrounding system already validated so that any divergence is attributable to the block under test, not to the platform.

---

## 6. Use Cases

**Pre-silicon firmware and RTOS verification.** Validate firmware, drivers, and OS abstractions against a silicon-accurate digital twin before tape-out, with the same diagnostic tooling — symbolized fault reports, MMIO and exception tracing, instruction-coverage triage — available for any instruction set the platform supports.

**Type-1 hypervisor development.** Race conditions between cores and interrupt-injection ordering defects are, by construction, difficult to reproduce on a non-deterministic emulator. protoXE's determinism and instruction-granular reverse execution turn a once-observed, never-reproduced timing bug into a deterministic, replayable, steppable artifact.

**Secure-world and TEE validation.** Full TrustZone EL3/EL1 modeling sufficient to boot OP-TEE unmodified allows secure/non-secure world isolation to be validated before any commitment to silicon.

**Defense and aerospace.** A deterministic, dependency-minimal verification tool, built in Rust with no third-party runtime dependencies in its core, domiciled in the EU, is a meaningfully different starting point for DO-178C or ISO 26262 tool qualification than a general-purpose emulator built for a different purpose and qualified after the fact.

---

## 7. European Sovereignty Context

The European Union's Chips Act, the European Defence Fund, and the NIS2 directive are, collectively, an explicit policy signal that European institutions and defense procurement increasingly require verification and development tooling that does not depend on non-EU vendors or toolchains. protoXE is developed and domiciled in the EU (Belgium), built in Rust with no external runtime dependencies in its validated core, and licensed to keep the validation methodology itself in the public domain as prior art (see References) — available to any team, anywhere, rather than locked behind a single vendor's commercial license.

---

## 8. Licensing and Availability

protoXE's core is released under AGPL v3, with a commercial exception license available for organizations that cannot comply with AGPL's copyleft obligations — typically because their own firmware or integration is proprietary. Three of the platform's core validation techniques have been placed in the public domain as defensive publications (CC0), specifically to keep this methodology open and to prevent any third party from foreclosing it through patent claims. They are listed in the References below.

protoXE is currently selecting a limited cohort of Design Partners — teams with a specific pre-silicon integration or verification challenge. Design partners receive direct engineering support, a co-developed validation report for their own SoC or IP block, early access to the RTL co-simulation bridge, and influence over the roadmap. Details: protoxe.com.

---

## References

1. Corbetta, C. (2026). *Lock-Step ISS-vs-DUT Verification with Automated Test-Case Minimisation.* Defensive publication, Zenodo. DOI: 10.5281/zenodo.20678504
2. Corbetta, C. (2026). *Pure-Software RTL Co-Simulation via Verilated Peripheral Substitution in a Zero-Allocation SoC Model.* Defensive publication, Zenodo. DOI: 10.5281/zenodo.20678643
3. Corbetta, C. (2026). *Zero-Allocation Deterministic Snapshot/Restore Enabling Reverse Debugging in a Compile-Time-Topology Emulator.* Defensive publication, Zenodo. DOI: 10.5281/zenodo.20678682
4. Zeller, A. & Hildebrandt, R. (2002). Simplifying and Isolating Failure-Inducing Input. *IEEE Transactions on Software Engineering*, 28(2), 183–200.
5. Arm Limited. *Arm Architecture Reference Manual for A-profile Architecture* (DDI 0487).
6. Arm Limited. *Arm Generic Interrupt Controller Architecture Specification, GICv3 and GICv4* (IHI 0069).
7. Arm Limited. *Cortex-A76 Technical Reference Manual* (DDI 0526); *Cortex-A55 Technical Reference Manual* (DDI 0484).
8. Verilator open-source Verilog/SystemVerilog simulator. https://github.com/verilator/verilator

---

*This document is licensed under CC-BY 4.0 — you may share and adapt it with attribution. © 2026 Claudio Corbetta / protoXE. For technical questions or Design Partner enquiries: claudio.corbetta@protoxe.com.*
