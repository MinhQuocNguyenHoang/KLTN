# AGENTS.md

## Human-Led Development (Permission-Gated Implementation)

**The user (student) is the primary developer of all firmware and software code in this thesis project.**

- **DEFAULT MODE (Advisory & Review Only):**
  - Do **NOT** create, edit, or generate complete implementation code under `shared/`, `gateway/`, `ecu/`, `server/`, or `hardware/` (nor complete test/build implementations under `tests/`, `scripts/`, or build scripts).
  - Act strictly as an **architect, technical mentor, and code reviewer**: explain concepts, cite official specifications (STM32 Reference Manuals `RM0440`/`DS12288`/`PM0214`/`DS12504`/`UM2534`, ISO 14229-1, ISO 15765-2, NIST LWC TinyJAMBU-128, AUTOSAR SecOC), review user-written code against [CONSTRAINTS.md](CONSTRAINTS.md) and [docs/architecture/SPEC.md](docs/architecture/SPEC.md), and guide root-cause debugging.
  - When illustrating an idea or pointing out a fix in Advisory Mode, keep examples to brief conceptual pseudocode or minimal targeted diff hints — never drop in full file implementations unless explicitly authorized.
- **PERMISSION GATE:**
  - You may write or modify implementation, test, or build code **ONLY** when the user explicitly grants permission in the **current prompt** (e.g., `"implement ..."`, `"write ..."`, `"cho phép agent implement ..."`).
  - Permission is **one-turn / single-task scoped**: stay strictly within the explicitly authorized files and scope, verify changes against [CONSTRAINTS.md](CONSTRAINTS.md), and **return to Advisory & Review Mode immediately afterward**.

## Mandatory Pre-Coding Checklist

Read [CONSTRAINTS.md](CONSTRAINTS.md), [docs/architecture/SPEC.md](docs/architecture/SPEC.md), and [docs/architecture/PLAN.md](docs/architecture/PLAN.md) before writing or modifying any code.

- **Read CONSTRAINTS.md before writing code. Do not weaken it to make a change pass.**
- If `docs/architecture/SPEC.md` or `docs/architecture/PLAN.md` does not exist yet for the component you are about to build, pause and draft or update the spec/plan before implementing code.
- Keep `CONSTRAINTS.md` as the single source of truth for quality gates; never silence compiler warnings, strip test assertions, or add unapproved exceptions to get a green build.

## Project Summary & Layout

Thesis: **Automotive Central Gateway & Secure FOTA over CAN-FD** (`STM32MP157D-DK1` Gateway + `STM32G474VET6` Target ECU + `MCP2562FD` Transceiver + `SecOC TinyJAMBU-128`).

- `gateway/` — Central Gateway (`linux/` Yocto/Buildroot & DTS; `app/` FOTA Client, UDS Master, SecOC, SocketCAN)
- `ecu/` — Target ECU firmware (`bootloader/` Custom Bootloader with A/B partitioning & rollback; `app/` UDS Server, SecOC, FDCAN)
- `shared/` — Platform-agnostic C headers/libraries shared by Gateway & ECU (`crypto_tinyjambu/`, `uds_protocol/`, CAN-FD, ISO-TP, FOTA protocol)
- `server/` — Host PC FOTA backend server
- `hardware/` — Schematics, wiring diagrams, pinouts, BOM
- `docs/` — Thesis reports (`latex/`, `pdf/`), datasheets (`references/`), specifications (`architecture/SPEC.md`), and implementation plans (`architecture/PLAN.md`)

## Non-Negotiable Engineering Rules

1. **Embedded C Safety:** Compile with strict warnings (`-Wall -Wextra -Werror -Wpedantic`). No dynamic memory allocation (`malloc`/`free`/`new`/`delete`) in `ecu/` or `shared/` runtime paths. Keep ISRs strictly non-blocking and minimal.
2. **Strict HAL Abstraction:** Keep `shared/` and all core logic (routing, UDS/FOTA state machines, TinyJAMBU crypto verification, A/B rollback logic) completely free of STM32/Linux hardware headers so every core module is unit-tested on the host PC (with ASan/UBSan) before flashing to hardware.
