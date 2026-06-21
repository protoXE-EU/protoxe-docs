# protoXE — Documentation & Research

Public technical documentation, white papers, and research notes for **protoXE** — a sovereign, deterministic full-system emulator for ARM64 and RISC-V, built for pre-silicon and high-assurance verification.

🌐 [protoxe.com](https://protoxe.com) · 📖 [CLI Manual](https://protoxe.com/manual.html) · ✉️ [claudio.corbetta@protoxe.com](mailto:claudio.corbetta@protoxe.com)

---

## Contents

- [`whitepaper.md`](./whitepaper.md) — protoXE's architecture, validation methodology, and empirical results
- Defensive publications (prior art, CC0, hosted on Zenodo):
  - Lock-Step ISS-vs-DUT Verification with Automated Test-Case Minimisation — DOI: *[insert]*
  - Pure-Software RTL Co-Simulation via Verilated Peripheral Substitution — DOI: *[insert]*
  - Zero-Allocation Deterministic Snapshot/Restore Enabling Reverse Debugging — DOI: *[insert]*

## About protoXE

protoXE is a full-system emulator for ARM64 and RISC-V written in pure Rust with zero runtime heap allocations. It boots unmodified Linux, RTOSes, Type-1 hypervisors, and bare-metal firmware on a digital twin that is instruction-for-instruction reproducible, validated against real silicon and an independent authoritative reference (QEMU), not just against its own test suite.

Core licensing: AGPL v3, with a commercial exception for organizations unable to comply with its copyleft terms. Source code release: in progress — see [protoxe.com](https://protoxe.com) for current status.

## Design Partner Program

protoXE is currently selecting a limited cohort of teams with a specific pre-silicon integration or verification challenge. Design partners get direct engineering support, a co-developed validation report, and early access to the RTL co-simulation bridge. [Apply →](https://protoxe.com/#contact)

## License

Documents in this repository are licensed under [CC-BY 4.0](./LICENSE) unless otherwise noted. Code, when published, will be licensed separately under AGPL v3 + Commercial Exception.

---

*protoXE — EU-sovereign, deterministic, validated. Domiciled in Belgium.*
