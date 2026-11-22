# Constraints

**Project:** Automotive Central Gateway & Secure FOTA over CAN-FD (`STM32MP157D-DK1` Gateway + `STM32G474VET6` ECU + `MCP2562FD` + `SecOC TinyJAMBU-128`)  
**Last reviewed:** 2026-11-22 by Nguyễn Hoàng Minh Quốc  
**Escalation level:** Level 1 (Written contract — greenfield scaffold; promote to Level 2 scripted `check:fast` / `check:task` / `check:full` when the first `shared/` module and build system land)

## Enforcement Policy

- **Block** on: Floor violations, compiler warnings (`-Wall -Wextra -Werror`), host sanitizers, HAL/architecture boundary violations, runtime heap-ban violations, and failing host unit tests.
- **Warn** on: Newly added checks whose tooling or target build is not yet wired up (coverage, STM32 Flash/RAM size budgets, `server/` checks). Promote each `Warn` row to `Block` once its first module lands and its command has been verified to fail on a deliberate violation and pass on a clean control.
- **Time budgets:** Edit-loop checks `< 5 s` (changed file / diff-scoped); task-end verification `<= 90 s`; CI unlimited.

---

## Floor (always enforced, no setup required — Block)

- **No new suppression comments:** Do not add `#pragma GCC diagnostic ignored`, `#pragma clang diagnostic ignored`, `NOLINT`, `cppcheck-suppress`, `LCOV_EXCL`, `gitleaks:allow`, `nosemgrep`, `# noqa`, `# type: ignore`, `@ts-ignore`, or `eslint-disable` to silence a check.
- **No unimplemented stubs:** No `TODO`/`FIXME` standing in for required logic, no empty error/catch blocks, and no placeholder functions returning success (`return 0;` / `return true;`) in security verification, CRC, UDS, or rollback paths.
- **No skipped or deleted tests:** Do not skip tests (`TEST_IGNORE`, `GTEST_SKIP`, `@pytest.mark.skip`, `.skip`), delete test files, or strip assertions without an explicit reason in the commit message.
- **No secrets in source:** Never commit real TinyJAMBU pre-shared keys, TLS private keys, certificates, or credentials. Deterministic crypto test vectors belong strictly under test directories and must be labeled as dummy fixtures.
- **No dynamic heap allocation in embedded runtime paths:** `malloc`, `calloc`, `realloc`, `free`, `strdup`, `new`, and `delete` are forbidden in `ecu/` and `shared/`. Use fixed-size static buffers, bounded ring buffers, or caller-owned memory.
- **ISR safety:** Interrupt Service Routines in `ecu/` must be bounded and non-blocking. Never call `printf`/stdio, Flash erase/program routines, blocking delays, mutex/semaphore waits, or full AEAD/UDS parsing inside an ISR — only copy into bounded lock-free ring buffers or update `volatile`/atomic flags.
- **No weakening of this file:** `CONSTRAINTS.md` does not get weakened (thresholds lowered, rules removed, or unapproved exceptions added) to make a change pass.

---

## Architecture & HAL Abstraction Boundaries (Block)

Core protocol, routing, security, and FOTA logic **must** compile and run natively on the host PC (`gcc`) with mock/fake HAL implementations before flashing to hardware.

| Directory / Layer | Allowed Dependencies | Forbidden Dependencies |
|---|---|---|
| `shared/` (`crypto_tinyjambu/`, `uds_protocol/`, CAN-FD, ISO-TP, FOTA headers) | C11 standard freestanding/hosted headers only (`<stdint.h>`, `<stddef.h>`, `<stdbool.h>`, `<string.h>`) | Any MCU, RTOS, or OS-specific header (`stm32*`, `cmsis*`, `FreeRTOS*`, `<linux/can.h>`, `<sys/socket.h>`, `<unistd.h>`, `<pthread.h>`) |
| Core logic (`gateway/app/core/`, `ecu/bootloader/core/`, `ecu/app/core/`) | `shared/`, C11 standard headers, and project HAL interface headers (`*_hal.h` function tables / opaque handles) | Direct register access, STM32 HAL/LL calls, Linux SocketCAN/POSIX syscalls, or filesystem/network I/O outside injected HAL interfaces |
| Thin HAL / Port adapters (`gateway/app/port/`, `ecu/*/hal/`, `ecu/*/port/`) | Target SDK/OS APIs (`stm32g4xx_hal*`, FDCAN, Flash driver, Linux SocketCAN, TLS client) implementing `*_hal.h` | Protocol parsing, UDS/FOTA state-machine logic, routing tables, or TinyJAMBU algorithm logic |
| `gateway/linux/` | Yocto/Buildroot recipes, Linux Device Tree (`.dts`/`.dtsi`), kernel configs | Application C source or duplicated `shared/` protocol headers |
| `server/` | Host backend stack + protocol definitions mirrored or generated from `shared/` | Embedded MCU headers or hardcoded production keys |

---

## Enforced with Numbers

Every enforced dimension pairs a concrete threshold, its rationale, and the command that produces the verdict. External checks (compiler, sanitizers, linker/size, git-diff guards) prevent circular self-grading by the test suite alone.

| Dimension | Rule | Rationale | Checked by | Runs at | Mode |
|---|---|---|---|---|---|
| C/C++ Warnings (Host & Target) | `0` warnings under `-std=c11 -Wall -Wextra -Werror -Wpedantic -Wconversion -Wshadow -Wstrict-prototypes` (C) or `-std=c++17 -Wall -Wextra -Werror -Wpedantic` (C++) | External compiler oracle catches integer truncation, signed/unsigned CAN DLC bugs, and UB before hardware testing | `gcc` (host) / `arm-none-eabi-gcc` (ECU) build target | every edit (`< 5s` single TU), task end, CI | **Block** |
| Memory Safety & UB (Host) | `0` AddressSanitizer or UndefinedBehaviorSanitizer findings | External runtime oracle catches OOB CAN-FD/ISO-TP payload accesses, stack overflows, and unaligned reads on host PC | Host test binary compiled with `-fsanitize=address,undefined -fno-omit-frame-pointer` | task end (`<= 90s`), CI | **Block** |
| Embedded Heap Ban | `0` references to `malloc`, `calloc`, `realloc`, `free`, `strdup`, `new`, `delete` in `ecu/` and `shared/` (and `0` `_sbrk` links in ECU `.elf`) | Guarantees deterministic RAM footprint and zero heap fragmentation on `STM32G474VET6` | `git grep -nE "\b(malloc\|calloc\|realloc\|free\|strdup)\b" -- ecu/ shared/` and `arm-none-eabi-nm` on `.elf` | every edit, task end, CI | **Block** |
| HAL Boundary Isolation | `0` forbidden hardware/OS `#include` directives in `shared/` or core logic directories | Guarantees routing, UDS/FOTA state machines, and TinyJAMBU remain 100% host-testable | `git grep -nE "#\s*include\s*[<\"].*(stm32\|cmsis\|FreeRTOS\|linux/can\|sys/socket)" -- shared/ gateway/app/core/ ecu/bootloader/core/ ecu/app/core/` | every edit, task end, CI | **Block** |
| Host Unit Tests (Core Logic) | `100%` pass rate (`0` failures, `0` skipped) across `shared/` and core logic modules | Proves CAN-FD framing, ISO-TP reassembly, UDS/FOTA transitions, rollback logic, and TinyJAMBU-128 test vectors before flashing | Host test suite (`ctest --output-on-failure` or `make test-host`) | task end (`<= 90s`), CI | **Block** |
| Core Logic Line Coverage | `>= 80%` line coverage on changed/new lines in `shared/` and core logic | High enough to require error/edge-case tests (invalid MAC, power-loss rollback, timeout) without blocking on unreachable defensive paths | `gcov` / `lcov` from host test build (`--coverage`) intersected with `git diff` | task end (`<= 90s`), CI | **Warn** *(Block once first `shared/` module builds)* |
| ECU Bootloader Footprint (`STM32G474VET6`) | Flash `<= 32 KB` (`.text` + `.rodata` + `.data`) and static SRAM `<= 16 KB` (`.data` + `.bss`) | Preserves symmetric A/B slots inside 512 KB Dual-Bank Flash (2 KB/page) and leaves stack headroom in 96 KB contiguous SRAM1+SRAM2 | `arm-none-eabi-size --format=berkeley build/ecu_bootloader.elf` + linker script `.ld` `MEMORY` bounds | target build, CI | **Warn** *(Block once `ecu/bootloader/` builds)* |
| ECU App Footprint (`STM32G474VET6`) | Flash `<= 224 KB` per A/B slot and static SRAM `<= 64 KB` (`.data` + `.bss`) | Fits within one 256 KB bank (after bootloader/metadata reserve) while leaving `>= 32 KB` SRAM1+SRAM2 for call stack and FDCAN buffers | `arm-none-eabi-size --format=berkeley build/ecu_app.elf` + linker script `.ld` `MEMORY` bounds | target build, CI | **Warn** *(Block once `ecu/app/` builds)* |
| Server & Host Tooling Safety | `0` high or critical static/security findings and `0` type/lint errors in `server/` | Prevents FOTA backend transport, signing, or manifest parsing vulnerabilities | `ruff check .` + `mypy src tests` + `pytest` (Python FastAPI stack per `docs/architecture/SPEC.md`) | task end (`<= 90s`), CI | **Warn** *(Block once `server/` is initialized)* |
| Floor Guard | `0` diff-scoped floor violations (suppressions, stubs, skipped/deleted tests, weakened `CONSTRAINTS.md`, untracked exceptions) | Stops agents from lowering the bar instead of fixing failing code | Diff-scoped floor guard against merge base (`scripts/floor-guard`) | task end (`< 5s`), CI | **Block** |

> **Note on Flash Partitioning:** Exact start addresses and page boundaries for Bootloader (48 KB partition at `0x0800_0000`, `<= 32 KB` code budget), Metadata & Freshness NVM (16 KB at `0x0800_C000`), Slot A (224 KB at `0x0801_0000`), and Slot B (224 KB at `0x0804_8000`) on `STM32G474VET6` (512 KB Dual-Bank Flash, 2 KB/page; 128 KB SRAM = 80 KB SRAM1 + 16 KB SRAM2 + 32 KB CCM) are defined in [docs/architecture/SPEC.md](docs/architecture/SPEC.md) and enforced by the linker scripts (`.ld`). See [docs/architecture/PLAN.md](docs/architecture/PLAN.md) for the implementation roadmap.

---

## Lifecycle Placement

| Phase | Target Budget | What Runs | Scope |
|---|---|---|---|
| **BUILD** (every edit) | `< 5 s` | Strict C/C++ compile check, heap-ban grep, HAL-boundary grep, floor check | Changed file / diff only |
| **VERIFY** (task end) | `<= 90 s` | Full host unit-test suite under ASan/UBSan, changed-line coverage (`gcov`/`lcov`), `server/` checks, diff floor guard | Changed modules + related suites |
| **REVIEW** (pre-commit / PR) | Minutes | Full host + cross-compile (`arm-none-eabi-gcc`) builds, `arm-none-eabi-size` footprint checks, full `CONSTRAINTS.md` diff audit | Whole branch diff |
| **SHIP** (CI / pre-flash gate) | Unlimited | All checks above + ratchet direction checks; firmware is never flashed to `STM32G474VET6` or deployed to `STM32MP157D-DK1` with a red host suite | Full repository |

---

## Measured, Not Yet Enforced (Ratchets)

Record baseline measurements on the first commit of each subsystem. Ratchet tolerance is `0.5%` for coverage metrics; size metrics must never exceed the hard budgets above.

| Metric | Today | Direction |
|---|---|---|
| `shared/` overall line coverage | `TBD (baseline on first shared/ PR)` | must not fall |
| `gateway/app/` & `server/` glue/adapter test coverage | `TBD (baseline on first PR)` | must not fall |
| `ecu/bootloader` Flash / static SRAM usage | `TBD (baseline on first build)` | must not grow above `32 KB` / `16 KB` |
| `ecu/app` Flash / static SRAM usage | `TBD (baseline on first build)` | must not grow above `224 KB` / `64 KB` |
| `TinyJAMBU-128` AEAD latency per 64-byte CAN-FD frame | `TBD (baseline on first benchmark)` | must not grow |

---

## Tooling & Wiring Checklist

Detected on current host: `git`, `gcc`, `gcov`, `mingw32-make`, `python`.  
Pending installation / setup before promoting `Warn` rows to `Block`:

- [ ] Add build system (`CMakeLists.txt` or `Makefile`) exposing `check-fast`, `check-task`, and `check-full` targets mirroring the `Checked by` column above
- [ ] Install `arm-none-eabi-gcc` (and `arm-none-eabi-size` / `arm-none-eabi-nm`) for `STM32G474VET6` cross-compilation and footprint gates
- [ ] Add host C unit-test framework (e.g., Unity or Catch2) for `shared/` and `*/core/`
- [ ] Add diff-scoped `scripts/floor-guard` script and verify each enforced command fails on a deliberate violation and passes on a clean control

---

## Exceptions

Exceptions require an explicit owner, a tracked reason, and an expiry date (`<= 90 days` from creation). Agents must never add rows here without user approval.

| ID | Rule | Path | Reason | Owner | Expires |
|---|---|---|---|---|---|
| *(none)* | — | — | — | — | — |
