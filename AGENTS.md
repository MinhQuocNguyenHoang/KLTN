# AGENTS.md

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
