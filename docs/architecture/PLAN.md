# Implementation Plan: Automotive Central Gateway & Secure FOTA over CAN-FD (V1)

**Source spec:** [docs/architecture/SPEC.md](SPEC.md)  
**Quality contract:** [CONSTRAINTS.md](../../CONSTRAINTS.md)  
**Workflow rule:** [AGENTS.md](../../AGENTS.md) *(Human-Led Development — student implements every task; agent acts as architect, mentor, and auditor unless explicitly granted one-task implementation permission)*  
**Last updated:** 2026-11-22  

---

## 1. Overview

This plan breaks the V1 **Automotive Central Gateway & Secure FOTA over CAN-FD** thesis prototype into **23 small, dependency-ordered tasks across 7 phases**, tailored for a student developer implementing every module hands-on. The build progresses strictly bottom-up: **Phase 0 (Toolchain & Gates)** → **Phase 1 (`shared/`)** → **Phase 2 (`server/`)** → **Phase 3 (`gateway/`)** → **Phase 4 (`ecu/bootloader/`)** → **Phase 5 (`ecu/app/`)** → **Phase 6 (SIL `vcan0` → HIL Hardware & Benchmarks)**. Every platform-agnostic core module is written in strict C11 and verified on the host PC under ASan/UBSan before a single line is flashed to `STM32G474VET6` or `STM32MP157D-DK1`.

---

## 2. Key Architecture & Workflow Decisions

- **Human-Led Iterative Loop (Per Task):** Read prerequisite specs → implement the task's deliverables → run self-verification commands (tests, sanitizers, `CONSTRAINTS.md` greps) → invoke the **Review Checkpoint** with the agent in Advisory Mode → commit only when green.
- **Host-First Verification:** All protocol, crypto, routing, UDS, and A/B rollback logic (`shared/` and `*/core/`) compiles natively with `gcc` + Unity + ASan/UBSan + `gcov` before cross-compiling for ARM.
- **Buildroot Environment (Closed §13 #3):** `gateway/linux/` Buildroot builds, ARM Cortex-A7 cross-compilation, and `vcan0` SIL runs execute in **WSL2 (Ubuntu on Windows)** using a native ext4 build tree (`~/kltn-buildroot`) while tracking clean configs/DTS/overlays in Git under `gateway/linux/`.
- **Deferred Hardware Pins & Crystal (Closed §13 #1 & #2):** `STM32G474VET6` `FDCAN1` pins (`PA11/PA12` or `PD0/PD1`, avoiding `PB8`/`BOOT0`) and external `HSE` crystal frequency (`8 MHz` or `24 MHz`) are isolated in `ecu/bootloader/port/board_config.h` and locked at **Task 4.4** during physical board inspection, keeping Phases 0–3 and Tasks 4.1–4.3 completely unblocked.
- **Progressive Constraint Promotion:** `CONSTRAINTS.md` starts at Level 1 (`Warn` for unbuilt targets). Phase Checkpoints explicitly promote each `Warn` gate to `Block` once its first module lands and passes a deliberate-failure sanity check.

---

## 3. Dependency Graph

```text
Phase 0: Host C11 CMake + Unity + Floor Guard (Task 0.1)
   │
   ▼
Phase 1: shared/ — Pure C11 Core Libraries (Tasks 1.1 → 1.6)
   ├── 1.1 Common Types / Endian / CRC32
   ├── 1.2 Pure C HAL Interfaces (*_hal.h)
   ├── 1.3 TinyJAMBU-128 AEAD Core (NIST KAT)
   ├── 1.4 SecOC Framing & Anti-Replay FV
   ├── 1.5 CAN-FD Helpers & ISO-TP (ISO 15765-2)
   └── 1.6 UDS Codec (ISO 14229-1) + .fota & boot_metadata_t Structs
        │
        ├──────────────────────────────┬───────────────────────────────┐
        ▼                              ▼                               ▼
Phase 2: server/ (2.1 → 2.2)   Phase 3: gateway/ (3.1 → 3.4)   Phase 4: ecu/bootloader/ (4.1 → 4.5)
  2.1 PKI + Ed25519 Signer       3.1 Default-Deny Router          4.1 Dual Metadata + Freshness NVM (Host)
  2.2 FastAPI HTTPS + Telemetry  3.2 UDS Master FOTA FSM          4.2 A/B Boot & Rollback FSM (Host)
                                 3.3 SocketCAN + HTTPS Port       4.3 UDS Programming Server (Host)
                                 3.4 Buildroot + DTS (WSL2)       4.4 HW Bring-Up + Linker/Clock (TBD Lock)
                                                                  4.5 G474 Flash/FDCAN/RNG/IWDG/Jump Port
                                                                       │
                                                                       ▼
                                                               Phase 5: ecu/app/ (5.1 → 5.2)
                                                                 5.1 App UDS + SecOC + Boot Confirm (Host)
                                                                 5.2 Slot A/B Linker Scripts + G474 Port
        │                              │                               │
        └──────────────────────────────┴───────────────────────────────┘
                                       ▼
                        Phase 6: Integration & Thesis Verification (6.1 → 6.3)
                          6.1 End-to-End SIL (FastAPI + Gateway + Sim ECU over vcan0)
                          6.2 HIL Bench Bring-Up (MP157D-DK1 + MCP2562FD + G474VET6)
                          6.3 Benchmarking (TinyJAMBU Cycles, Throughput, Footprint)
```

*(Note: After Phase 1 completes, Phases 2, 3.1–3.2, and 4.1–4.3 are mutually independent and can be interleaved if you want variety, though following 1 → 2 → 3 → 4 → 5 → 6 minimizes context switching.)*

---

## 4. Progress Checklist (Master Index)

- **Phase 0 — Toolchain & Gates:** `[ ] 0.1`
- **Phase 1 — `shared/`:** `[ ] 1.1` `[ ] 1.2` `[ ] 1.3` `[ ] 1.4` `[ ] 1.5` `[ ] 1.6` *(Checkpoint 1)*
- **Phase 2 — `server/`:** `[ ] 2.1` `[ ] 2.2` *(Checkpoint 2)*
- **Phase 3 — `gateway/`:** `[ ] 3.1` `[ ] 3.2` `[ ] 3.3` `[ ] 3.4` *(Checkpoint 3)*
- **Phase 4 — `ecu/bootloader/`:** `[ ] 4.1` `[ ] 4.2` `[ ] 4.3` `[ ] 4.4` `[ ] 4.5` *(Checkpoint 4)*
- **Phase 5 — `ecu/app/`:** `[ ] 5.1` `[ ] 5.2` *(Checkpoint 5)*
- **Phase 6 — Integration & HIL:** `[ ] 6.1` `[ ] 6.2` `[ ] 6.3` *(Final V1 Sign-Off)*

---

## Phase 0: Workspace Toolchain, Build Skeleton & Quality Gates

### [ ] Task 0.1: Host C11 CMake + Unity Test Harness & Floor Guard
**Dependencies:** None | **Estimated scope:** Medium (5 files)

1. **Goal & Deliverables:**
   - Set up the top-level host build system with strict C11 compiler warnings (`-std=c11 -Wall -Wextra -Werror -Wpedantic -Wconversion -Wshadow -Wstrict-prototypes`), optional ASan/UBSan and `gcov` coverage flags, a Unity C unit-test harness, and a diff-scoped floor-guard script implementing [CONSTRAINTS.md](../../CONSTRAINTS.md).
   - Files to write:
     - `CMakeLists.txt` (top-level host build & CTest registration)
     - `cmake/compiler_flags.cmake` (strict C11 flags, sanitizer & coverage toggles)
     - `tests/CMakeLists.txt` (Unity integration via vendored `unity.c`/`unity.h` or CMake `FetchContent`)
     - `tests/test_sanity.c` (minimal passing Unity smoke test)
     - `scripts/floor_guard.py` (checks diff for suppression comments, `TODO`/`FIXME` stubs, skipped tests, `malloc`/`free` in `ecu/`/`shared/`, forbidden OS/HW headers in `shared/`/`*/core/`, and unauthorized edits to `CONSTRAINTS.md`)
2. **Prerequisite Concepts & Official References:**
   - CMake `INTERFACE` libraries, `CTest`, GCC warning flags (`-Wconversion`, `-Wshadow`, `-Wstrict-prototypes`), GCC AddressSanitizer (`-fsanitize=address,undefined`), and `gcov` instrumentation (`--coverage`).
   - Project docs: [CONSTRAINTS.md](../../CONSTRAINTS.md) (§Floor, §Enforced with Numbers, §Tooling & Wiring Checklist), [docs/architecture/SPEC.md](SPEC.md) (§6 Commands, §8 Code Style).
   - Tooling check: verify `gcc`, `cmake`, `gcov`, `python`, `arm-none-eabi-gcc` (install ARM GNU Toolchain if not yet present), and WSL2 Ubuntu availability.
3. **Acceptance & Self-Verification Criteria:**
   - [ ] `cmake -B build/host -DCMAKE_BUILD_TYPE=Debug -DENABLE_SANITIZERS=ON -DENABLE_COVERAGE=ON && cmake --build build/host` succeeds with `0` warnings. *(Note: on MinGW Windows where ASan may be unavailable, run ASan/UBSan tests inside WSL2 `gcc` and verify both environments cleanly.)*
   - [ ] `ctest --test-dir build/host --output-on-failure` executes `test_sanity` and passes (`100%`).
   - [ ] Deliberate-failure test: temporarily insert a `#pragma GCC diagnostic ignored` or `malloc(1)` in a test diff and confirm `python scripts/floor_guard.py` exits non-zero; revert and confirm it exits `0`.
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to audit `cmake/compiler_flags.cmake` and `scripts/floor_guard.py` against every single row of [CONSTRAINTS.md](../../CONSTRAINTS.md) to verify no warning flag, forbidden header pattern, or floor rule was missed.

#### Checkpoint 0: Toolchain Ready
- [ ] Host C11 build + Unity + `floor_guard.py` verified green on a clean tree.
- [ ] WSL2 Ubuntu shell confirmed working with `gcc`, `cmake`, `python3`, `git`.

---

## Phase 1: `shared/` — Platform-Agnostic C11 Core Libraries (`shared-proto`)

### [ ] Task 1.1: Common Status Codes, Endianness Helpers & IEEE CRC32
**Dependencies:** Task 0.1 | **Estimated scope:** Medium (5 files)

1. **Goal & Deliverables:**
   - Implement the foundational C11 primitives shared by Gateway and ECU: unified `fota_status_t` error codes, safe unaligned big-endian/little-endian read/write helpers (eliminating strict-aliasing/alignment UB), and IEEE 802.3 CRC32 (for `boot_metadata_t` integrity).
   - Files to write:
     - `shared/uds_protocol/fota_types.h` (`fota_status_t`, `fota_slot_id_t`, `fota_slot_state_t`, constants matching SPEC §3.2)
     - `shared/uds_protocol/endian_utils.h` (`read_be16/32/64`, `write_be16/32/64`, `read_le16/32`, `write_le16/32`)
     - `shared/uds_protocol/crc32.h`
     - `shared/uds_protocol/crc32.c` (bitwise or 16-entry nibble table to stay Flash-lean for Bootloader)
     - `tests/test_common_crc32.c` (standard `"123456789"` → `0xCBF43926` vector, empty input, incremental update, endianness round-trips)
2. **Prerequisite Concepts & Official References:**
   - C11 strict aliasing, integer promotions (`-Wconversion` pitfalls when shifting `uint8_t`), unaligned memory access on ARM Cortex-M4 vs x86.
   - Local thesis notes: [docs/pdf/02_CInEmbeddedSystem.pdf](../pdf/02_CInEmbeddedSystem.pdf).
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§3.2, §8 Code Style).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] `crc32_compute("123456789", 9)` returns `0xCBF43926u`; incremental chunked CRC matches single-pass CRC.
   - [ ] All endianness helpers pass round-trip tests with boundary values (`0`, `UINT32_MAX`, `UINT64_MAX`, odd buffer offsets).
   - [ ] `ctest --test-dir build/host --output-on-failure` passes with ASan/UBSan (`0` warnings, `0` UB).
   - [ ] `git grep -nE "\b(malloc|calloc|realloc|free|strdup)\b" -- shared/` returns empty.
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to review `endian_utils.h` and `crc32.c` for signed-shift UB, `-Wconversion` edge cases, and Flash vs speed trade-offs for the 32 KB Bootloader budget.

### [ ] Task 1.2: Pure C HAL Interface Contracts (`shared/hal_if/`)
**Dependencies:** Task 1.1 | **Estimated scope:** Medium (5 files)

1. **Goal & Deliverables:**
   - Define the opaque-context function-pointer HAL interfaces (`*_hal_t`) so `gateway/app/core/`, `ecu/bootloader/core/`, and `ecu/app/core/` never include STM32 or Linux headers.
   - Files to write:
     - `shared/hal_if/can_hal.h` (CAN-FD send/receive non-blocking interface using `canfd_frame_t`)
     - `shared/hal_if/flash_hal.h` (page erase, 8-byte double-word write, read — matching SPEC §8 example)
     - `shared/hal_if/timer_hal.h` (`now_ms`, watchdog feed/arm, backup-register read/write, system reset callbacks)
     - `shared/hal_if/rng_hal.h` (fill `N` cryptographically random bytes for UDS `0x27` Seed)
     - `tests/mocks/mock_hal_compile_test.c` (compile-time & null-callback contract sanity test)
2. **Prerequisite Concepts & Official References:**
   - Dependency Injection / Hardware Abstraction Layer pattern in bare-metal C (function-pointer vtables + `void *ctx`).
   - Local thesis notes: [docs/pdf/04_PointerAndMCUPeripherals.pdf](../pdf/04_PointerAndMCUPeripherals.pdf), [docs/pdf/06_RTOSvsLinuxAppDevelopment.pdf](../pdf/06_RTOSvsLinuxAppDevelopment.pdf).
   - Spec & Contract: [CONSTRAINTS.md](../../CONSTRAINTS.md) (§Architecture & HAL Abstraction Boundaries), [docs/architecture/SPEC.md](SPEC.md) (§8 Example C11 HAL-Abstracted Core Module Pattern).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] Every header under `shared/hal_if/` compiles standalone with `-std=c11 -Wall -Wextra -Werror -Wpedantic` and includes only `<stdint.h>`, `<stddef.h>`, `<stdbool.h>`, and `shared/` headers.
   - [ ] `git grep -nE "#\s*include\s*[<\"].*(stm32|cmsis|FreeRTOS|linux/can|sys/socket)" -- shared/` returns empty.
   - [ ] `ctest --test-dir build/host --output-on-failure` passes.
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to verify whether the HAL signatures cleanly cover every hardware operation required by Gateway SocketCAN, Bootloader Flash/RNG/IWDG/BKP0R, and ECU App without leaking STM32 or POSIX types.

### [ ] Task 1.3: TinyJAMBU-128 AEAD Core & Constant-Time Tag Verification
**Dependencies:** Task 1.1 | **Estimated scope:** Small (3 files)

1. **Goal & Deliverables:**
   - Implement the NIST LWC Finalist **TinyJAMBU-128** authenticated encryption algorithm (128-bit key, 96-bit nonce, 64-bit tag, 128-bit state on `uint32_t s[4]`, zero S-box tables) plus a constant-time tag comparison helper (`ct_memcmp`) and streaming/incremental full-image tag API (needed by Bootloader to verify 224 KB Flash slots without buffering the whole image in RAM).
   - Files to write:
     - `shared/crypto_tinyjambu/tinyjambu128.h`
     - `shared/crypto_tinyjambu/tinyjambu128.c`
     - `tests/test_tinyjambu_kat.c` (official NIST LWC TinyJAMBU-128 Known-Answer Test vectors + tamper tests)
2. **Prerequisite Concepts & Official References:**
   - Hongjun Wu & Tao Huang, *TinyJAMBU: A Family of Lightweight Authenticated Encryption Algorithms (Version 2)*, NIST Lightweight Cryptography (LWC) submission specification (state update `P_640` / `P_1024`, frame bits `0x10`/`0x30`/`0x50`/`0x70`, partial-block byte ordering, little-endian 32-bit word packing).
   - NIST LWC `LWC_AEAD_KAT_128_96.txt` test vectors for TinyJAMBU-128.
   - Constant-time byte comparison (`diff |= a[i] ^ b[i]`) to prevent timing side-channels.
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§4.6).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] Passes official NIST LWC TinyJAMBU-128 KAT vectors covering empty PT/AD, partial (<4 B) blocks, exact 4-byte multiples, and large (>64 B) payloads (all labeled as dummy test fixtures per `CONSTRAINTS.md`).
   - [ ] Decrypting with any 1-bit flip in ciphertext, tag, nonce, key, or AD returns `FOTA_ERR_AUTH_FAILED` and zeros the plaintext output buffer (never leaks unauthenticated plaintext).
   - [ ] Streaming/incremental API produces identical tags to one-shot API on a 2048-byte & 64 KB buffer.
   - [ ] `ctest --test-dir build/host -R tinyjambu --output-on-failure` passes with ASan/UBSan and `0` warnings.
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to audit `tinyjambu128.c` for: (a) bit/endianness fidelity to TinyJAMBU v2 spec, (b) constant-time tag verification (confirming compiler cannot short-circuit), (c) zeroing plaintext on failed decrypt, and (d) stack footprint (`<= 256 B` stack usage).

### [ ] Task 1.4: SecOC Framing, 96-bit Nonce Builder & Monotonic Freshness Anti-Replay
**Dependencies:** Task 1.3 | **Estimated scope:** Small (3 files)

1. **Goal & Deliverables:**
   - Implement the SecOC layer on top of TinyJAMBU-128: 96-bit Nonce builder (`sender_id || boot_epoch || fv`), Wire Format 1 (single-frame CAN-FD `<= 64 B` AuthOnly & AEAD-Encrypted frames with 48-bit truncated FV reconstruction), Wire Format 2 (UDS `0x36` AEAD block up to `2048 B` plaintext), and strictly monotonic anti-replay state (`fv_rx > fv_last_accepted`, updated **only after** tag passes).
   - Files to write:
     - `shared/crypto_tinyjambu/secoc.h`
     - `shared/crypto_tinyjambu/secoc.c`
     - `tests/test_secoc.c`
2. **Prerequisite Concepts & Official References:**
   - AUTOSAR *Specification of Secure Onboard Communication (SecOC)* (Freshness Value Manager, truncated FV reconstruction, Authentic I-PDU vs Secured I-PDU).
   - Nonce uniqueness invariant in AEAD (why `(Key, Nonce)` reuse breaks confidentiality/integrity and how `boot_epoch` prevents post-reset reuse).
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§4.6 Wire Format 1, Wire Format 2, Verification Invariant).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] Wire Format 1 encodes/decodes `0..48` byte payloads within `<= 64` bytes, verifying Associated Data includes `can_id || flags_ver || payload_len || full_fv`.
   - [ ] Wire Format 2 encodes/decodes `1..2048` byte UDS `0x36` blocks with AD `ecu_id || target_slot || bsc || plaintext_len || fv`.
   - [ ] Replaying a previously accepted frame (`fv_rx == fv_last`), an older frame (`fv_rx < fv_last`), or a valid `fv_rx > fv_last` with a bad MAC fails without mutating `fv_last_accepted`.
   - [ ] Truncated 48-bit FV rollover boundary test passes.
   - [ ] `ctest --test-dir build/host -R secoc --output-on-failure` passes under ASan/UBSan.
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to audit `secoc.c` for replay-window edge cases (e.g., forged high FV DoS protection — verify `fv_last` never advances on bad MAC), buffer length bounds (`payload_len > 48`), and exact byte alignment with SPEC §4.6.

### [ ] Task 1.5: CAN-FD Frame Helpers & ISO-TP (ISO 15765-2:2016) Engine
**Dependencies:** Task 1.1, Task 1.2 | **Estimated scope:** Medium (4 files)

1. **Goal & Deliverables:**
   - Implement CAN-FD DLC↔byte-length conversion with `0xCC` padding and a full non-blocking, zero-heap ISO-TP (ISO 15765-2:2016) TX/RX state machine supporting CAN-FD 64-byte frames and SDUs up to `4095` bytes (needed for `2066`-byte UDS `0x36` blocks).
   - Files to write:
     - `shared/uds_protocol/canfd_frame.h` (`canfd_frame_t`, DLC/length lookup & `0xCC` padding helpers)
     - `shared/uds_protocol/isotp.h`
     - `shared/uds_protocol/isotp.c` (SF `1..7 B` 1-byte PCI, CAN-FD SF `8..62 B` 2-byte PCI, FF 12-bit length, CF `SN` wrap `1..15, 0..15`, FC `CTS`/`WAIT`/`OVFLW`, `BS=8`, `STmin=1 ms`, `N_As`/`N_Bs`/`N_Cr = 1000 ms` timeouts, static `4096 B` buffers)
     - `tests/test_isotp.c` (back-to-back TX↔RX loopback over a mock CAN queue: 1 B, 7 B, 8 B, 62 B, 63 B, 2066 B, 4095 B, wrong SN abort, FC `OVFLW`, `N_Bs`/`N_Cr` timeouts)
2. **Prerequisite Concepts & Official References:**
   - ISO 15765-2:2016 *Road vehicles — Diagnostic communication over Controller Area Network (DoCAN) — Part 2: Transport protocol and network layer services* (CAN-FD SF escape byte `0x00`, FF/CF/FC PCI layout, timing parameters `N_As`, `N_Bs`, `N_Cr`, `STmin`).
   - Bosch CAN-FD Specification v1.0 (discrete DLC values `0..8, 12, 16, 20, 24, 32, 48, 64`).
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§4.1, §4.4).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] Single-frame boundary tests (`len = 1, 7, 8, 62`) emit exact 1-byte vs 2-byte PCI and pad unused bytes with `0xCC` to the next valid CAN-FD DLC.
   - [ ] Multi-frame loopback transfers a `2066`-byte SDU and `4095`-byte SDU across multiple `BS=8` Flow Control cycles with exact byte match.
   - [ ] Out-of-sequence `CF` (`SN` mismatch), oversized `FF` (`> 4095`), and simulated `N_Bs`/`N_Cr` timeouts (`> 1000 ms` via mock `timer_hal_t`) cleanly reset the link and report error codes.
   - [ ] `ctest --test-dir build/host -R isotp --output-on-failure` passes under ASan/UBSan (`0` OOB accesses, `0` `malloc`).
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to audit `isotp.c` for ISO 15765-2:2016 corner cases: `SF_DL = 0` malformed frames, `FF_DL` smaller than CAN-FD SF threshold, `STmin` encoding (`0x00..0x7F` ms vs `0xF1..0xF9` 100–900 µs), and timer wrap-around (`(uint32_t)(now - start) >= timeout`).

### [ ] Task 1.6: UDS (ISO 14229-1) Protocol Codec, `.fota` Package & `boot_metadata_t` Structs
**Dependencies:** Task 1.1, Task 1.5 | **Estimated scope:** Medium (5 files)

1. **Goal & Deliverables:**
   - Define all V1 UDS service IDs, sub-functions, DIDs (`0xF195`), Routine IDs (`0x0203`, `0xFF00`, `0x0202`), standard NRCs, safe serializers/deserializers, and exact binary layouts for `fota_pkg_header_t` (128 B) and `boot_metadata_t` (128 B, 8-byte aligned) with `_Static_assert` size checks.
   - Files to write:
     - `shared/uds_protocol/uds_defs.h` (SIDs, NRCs, timing constants `P2_server=50 ms`, `S3_server=5000 ms`, CAN IDs `0x101..0x7E9`)
     - `shared/uds_protocol/uds_codec.h`
     - `shared/uds_protocol/uds_codec.c` (encode/decode for `0x10`, `0x11`, `0x22`, `0x27`, `0x31`, `0x34`, `0x36`, `0x37`, `0x3E`, `0x7F`)
     - `shared/uds_protocol/fota_format.h` (`fota_pkg_header_t`, `slot_meta_t`, `boot_metadata_t`, serialize/parse + magic/bounds validation)
     - `tests/test_uds_format.c`
2. **Prerequisite Concepts & Official References:**
   - ISO 14229-1:2020 *Unified Diagnostic Services (UDS) — Application Layer* (positive response `SID + 0x40`, negative response `0x7F <SID> <NRC>`, suppressPosRspMsgIndicationBit `0x80`, RequestDownload `0x34` `addressAndLengthFormatIdentifier = 0x44`).
   - C11 `_Static_assert` and alignment (`__attribute__((aligned(8)))` vs explicit byte serialization).
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§3.2, §4.2, §4.5, §5.2).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] `_Static_assert(sizeof(boot_metadata_t) == 128)` and `_Static_assert(sizeof(slot_meta_t) == 24)` compile cleanly.
   - [ ] Every UDS request/response codec round-trips valid payloads and returns `FOTA_ERR_INVALID_LEN` on truncated inputs (fuzz/boundary length tests from `0` to `max+1`).
   - [ ] `.fota` header parser validates `magic == 0x41544F46`, `header_version == 1`, `header_size == 128`, `target_slot in {1, 2}`, and `1 <= image_size <= 229376`.
   - [ ] `ctest --test-dir build/host -R uds_format --output-on-failure` passes under ASan/UBSan.
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to cross-check every byte offset in `fota_format.h` and `uds_codec.c` against SPEC §3.2, §4.5, and §5.2 so Python (`server/`) and C (`gateway/` + `ecu/`) agree byte-for-byte.

#### Checkpoint 1: `shared/` Complete — Gate Promotion
- [ ] Full host suite passes (`100%`, `0` failures, `0` skipped) under ASan/UBSan.
- [ ] `gcov`/`lcov` reports `>= 80%` line coverage across all `shared/` `.c` files.
- [ ] `python scripts/floor_guard.py` exits `0`; `0` heap refs and `0` forbidden OS/HW includes in `shared/`.
- [ ] **Constraint Update (Ask User/Agent):** Promote `Core Logic Line Coverage` in [CONSTRAINTS.md](../../CONSTRAINTS.md) from `Warn` to `Block` and record `shared/` overall line coverage baseline in the Ratchets table.

---

## Phase 2: `server/` — Host PC FOTA Backend & Offline Ed25519 Signer (`fota-server`)

### [ ] Task 2.1: Dev PKI Keygen & Offline `.fota` Package Signer CLI
**Dependencies:** Task 1.6 | **Estimated scope:** Medium (4 files)

1. **Goal & Deliverables:**
   - Initialize the Python 3.11+ `server/` project (`ruff`, `mypy --strict`, `pytest`), write a development key/TLS certificate generator (outputting strictly to git-ignored `server/secrets/` and `server/certs/`), and write the offline `.fota` firmware signing CLI producing packages matching `fota_pkg_header_t` (128 B + raw `.bin`).
   - Files to write:
     - `server/pyproject.toml` (dependencies: `fastapi`, `uvicorn`, `cryptography`, `pydantic`; dev: `ruff`, `mypy`, `pytest`, `pytest-cov`, `httpx`)
     - `server/tools/keygen_dev.py` (generates dev Root CA, localhost/LAN TLS server cert+key, and Ed25519 signing keypair into ignored directories)
     - `server/tools/sign_firmware.py` (packs 128-byte little-endian header, computes `SHA-256(payload)` and `Ed25519_Sign(header[0x00..0x3F])`, writes `.fota`, updates `server/storage/releases.json`)
     - `server/tests/test_sign_firmware.py` (tests valid signing/verification, 1-byte header tamper detection, 1-byte payload tamper detection, invalid slot/oversized `> 224 KB` rejection)
2. **Prerequisite Concepts & Official References:**
   - RFC 8032 (*Edwards-Curve Digital Signature Algorithm — Ed25519*), Python `struct.pack("<...")` little-endian binary packing, `cryptography.hazmat.primitives.asymmetric.ed25519` & X.509 dev certs.
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§5.1, §5.2, §5.3, §6 command #5).
   - Verify `.gitignore` excludes `server/secrets/`, `server/certs/`, `*.pem`, `*.key`.
3. **Acceptance & Self-Verification Criteria:**
   - [ ] `cd server && ruff check . && mypy src tools tests && pytest tests/test_sign_firmware.py` passes with `0` errors.
   - [ ] Cross-language compatibility check: generate a dummy `.fota` binary with `sign_firmware.py` and verify your C11 `fota_pkg_header_parse()` (from Task 1.6) parses all fields (`magic`, `target_ecu_id`, `target_slot`, `fw_version`, `image_size`, `image_sha256`, `ed25519_sig`) with exact match.
   - [ ] `git status` shows zero private keys or certificates tracked.
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to compare `struct.pack` format string in `sign_firmware.py` field-by-field against `shared/uds_protocol/fota_format.h` and audit `.gitignore` to ensure secrets cannot be accidentally committed.

### [ ] Task 2.2: FastAPI HTTPS REST API & FOTA Telemetry Store
**Dependencies:** Task 2.1 | **Estimated scope:** Medium (4 files)

1. **Goal & Deliverables:**
   - Implement the FastAPI FOTA backend serving signed release manifests, streaming `.fota` binaries per `(fw_version, target_slot)`, and logging Gateway FOTA status telemetry.
   - Files to write:
     - `server/src/fota_server/models.py` (Pydantic v2 models for manifest response, health, and telemetry POST payload)
     - `server/src/fota_server/storage.py` (safe path-traversal-proof release catalog loader & append-only JSONL telemetry log)
     - `server/src/fota_server/main.py` (FastAPI endpoints: `GET /api/v1/health`, `GET /api/v1/firmware/latest`, `GET /api/v1/firmware/{fw_version}/slot/{target_slot}/package`, `POST /api/v1/telemetry/fota-status`)
     - `server/tests/test_api.py` (FastAPI `TestClient` tests for all endpoints, `404` on unknown ECU/slot, `422` on malformed telemetry, path-traversal attempt rejection)
2. **Prerequisite Concepts & Official References:**
   - FastAPI `FileResponse` / `StreamingResponse`, Pydantic v2 strict validation, Uvicorn TLS configuration (`--ssl-keyfile`, `--ssl-certfile`), OWASP path-traversal prevention.
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§5.3).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] `cd server && ruff check . && mypy src tools tests && pytest --cov=src --cov-report=term-missing` passes (`0` lint/type/test failures, `>= 80%` coverage).
   - [ ] Manual HTTPS check: start Uvicorn on `https://127.0.0.1:8443` with dev certs from Task 2.1 and verify `curl --cacert server/certs/ca.crt https://127.0.0.1:8443/api/v1/health` returns `{"status":"ok"}`.
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to review `main.py` and `storage.py` for security issues (path traversal in `fw_version`/`target_slot`, untrusted input validation, TLS config) and spec compliance against SPEC §5.3.

#### Checkpoint 2: `server/` Complete — Gate Promotion
- [ ] `ruff`, `mypy`, and `pytest` all green; manual HTTPS download of a signed `.fota` package succeeds.
- [ ] **Constraint Update (Ask User/Agent):** Promote `Server & Host Tooling Safety` in [CONSTRAINTS.md](../../CONSTRAINTS.md) from `Warn` to `Block`.

---

## Phase 3: `gateway/` — Central Gateway App & Buildroot Custom Linux (`gateway-core` & `gateway-linux`)

### [ ] Task 3.1: Default-Deny CAN-FD Routing Table & Firewall (`gateway/app/core/`)
**Dependencies:** Task 1.4, Task 1.5 | **Estimated scope:** Small (3 files)

1. **Goal & Deliverables:**
   - Implement the pure-C11 Central Gateway default-deny routing and firewall engine: static rule table (`gw_route_rule_t`), FOTA-mode traffic throttling (`allow_during_fota`), SecOC sign/verify policy execution, and bounded diagnostics counters (`forwarded`, `dropped`, `secoc_fail`).
   - Files to write:
     - `gateway/app/core/router.h`
     - `gateway/app/core/router.c`
     - `tests/test_gateway_router.c` (tests all 4 V1 default rules, unknown CAN ID drop, `0x110` throttled during FOTA mode, spoofed/replayed `0x101` SecOC frame dropped & counted)
2. **Prerequisite Concepts & Official References:**
   - Automotive Central Gateway default-deny CAN firewalling and SecOC gateway verification.
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§4.2 CAN-FD ID Allocation Table, §4.3 Gateway CAN-FD Routing Table).
   - Contract: [CONSTRAINTS.md](../../CONSTRAINTS.md) (zero Linux/SocketCAN headers in `gateway/app/core/`).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] `ctest --test-dir build/host -R router --output-on-failure` passes under ASan/UBSan (`>= 80%` line coverage).
   - [ ] `git grep -nE "#\s*include\s*[<\"].*(stm32|cmsis|FreeRTOS|linux/can|sys/socket)" -- gateway/app/core/` returns empty.
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to audit `router.c` against SPEC §4.2 & §4.3 rules: confirm no frame can bypass `ROUTE_SECOC_VERIFY`, and verify counter overflow behavior is bounded/safe.

### [ ] Task 3.2: Gateway UDS Master & FOTA Orchestrator State Machine (`gateway/app/core/`)
**Dependencies:** Task 1.4, Task 1.5, Task 1.6, Task 3.1 | **Estimated scope:** Medium (5 files)

1. **Goal & Deliverables:**
   - Implement the pure-C11 UDS Master FOTA client state machine driving all 12 steps of SPEC §4.5 plus Freshness Value session management (`0x200`/`0x201` sync) and `0x3E` TesterPresent keep-alive.
   - Files to write:
     - `gateway/app/core/secoc_session.h`
     - `gateway/app/core/secoc_session.c` (manages Gateway-side `boot_epoch`, monotonic `FV`, and `0x200`/`0x201` sync)
     - `gateway/app/core/fota_master.h`
     - `gateway/app/core/fota_master.c` (12-step UDS FOTA FSM: query `0x22 0xF195`, validate target slot != active slot, `0x10 0x03`, `0x31 0x0203`, `0x10 0x02`, `0x27` Seed→Key with `PSK_UDS_AUTH`, `0x31 0xFF00` handling NRC `0x78`, `0x34`, chunked `0x36` with TinyJAMBU-128 AEAD + `PSK_FOTA_DATA`, `0x37`, `0x31 0x0202` with full-image TinyJAMBU tag, `0x11 0x01`, post-reset `0x22 0xF195` confirm)
     - `tests/test_fota_master.c` (mock ECU responder driving happy path, wrong target slot rejection, NRC `0x78` ResponsePending wait, NRC `0x35` InvalidKey abort, NRC `0x72` ProgrammingFailure abort, timeout abort)
2. **Prerequisite Concepts & Official References:**
   - ISO 14229-1 client timing (`P2_client`, `P2*_client` after NRC `0x78`, `S3_client` TesterPresent interval).
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§4.5, §4.6, §5.3 Trust Boundary Handoff).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] Happy-path unit test transfers a multi-block (>4 KB) firmware image through all 12 steps and reaches `FOTA_MASTER_DONE_CONFIRMED`.
   - [ ] Attempting to flash Slot A while ECU reports active slot A is rejected **before** entering Programming Session (`0x10 0x02`).
   - [ ] `ctest --test-dir build/host -R fota_master --output-on-failure` passes under ASan/UBSan (`>= 80%` line coverage, `0` Linux headers in `gateway/app/core/`).
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to audit `fota_master.c` state transitions for deadlock/stuck states, verify `0x78 ResponsePending` resets the response timer properly, and confirm full-image tag computation matches what Bootloader will compute in Phase 4.

### [ ] Task 3.3: Linux Port — SocketCAN CAN-FD Adapter & HTTPS/Ed25519 FOTA Client (`gateway/app/port/`)
**Dependencies:** Task 2.2, Task 3.1, Task 3.2 | **Estimated scope:** Medium (5 files)

1. **Goal & Deliverables:**
   - Implement the Linux OS bindings that connect `gateway/app/core/` to real SocketCAN (`can0`/`vcan0`) and the FastAPI HTTPS FOTA Server, verifying TLS certificate (pinned dev CA), Ed25519 header signature (pinned public key), and SHA-256 payload digest before handing plaintext to `fota_master`.
   - Files to write:
     - `gateway/app/port/socketcan_port.h`
     - `gateway/app/port/socketcan_port.c` (`AF_CAN`, `SOCK_RAW`, `CAN_RAW`, `setsockopt(CAN_RAW_FD_FRAMES=1)`, non-blocking `poll()`, implementing `can_hal_t`)
     - `gateway/app/port/https_client.h`
     - `gateway/app/port/https_client.c` (`libcurl` HTTPS GET/POST with `CURLOPT_CAINFO` + OpenSSL 3 `EVP_DigestVerify` Ed25519 & `EVP_sha256` verification)
     - `gateway/app/port/main.c` (CLI args: `--iface can0|vcan0`, `--server https://...`, `--ca ...`, `--pubkey ...`, `--psk-file ...`; runs event loop + FOTA trigger)
2. **Prerequisite Concepts & Official References:**
   - Linux Kernel Documentation: *SocketCAN — Controller Area Network* (`Documentation/networking/can.rst`: `struct canfd_frame`, `CANFD_BRS`, `CAN_RAW_FD_FRAMES`).
   - OpenSSL 3 `EVP_DigestVerifyInit` / `EVP_DigestVerify` for Ed25519 (`EVP_PKEY_ED25519`) and `libcurl` easy API (`CURLOPT_CAINFO`, `CURLOPT_SSL_VERIFYPEER`).
   - Local thesis notes: [docs/pdf/05_MCUvsMPUAndRTOSvsOS.pdf](../pdf/05_MCUvsMPUAndRTOSvsOS.pdf), [docs/pdf/06_RTOSvsLinuxAppDevelopment.pdf](../pdf/06_RTOSvsLinuxAppDevelopment.pdf).
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§5.3, §6 command #3).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] Compiles cleanly in WSL2 Ubuntu with `gcc` (`-std=c11 -Wall -Wextra -Werror -Wpedantic`) linked against `libcurl` and `libcrypto`.
   - [ ] Live test against local FastAPI server (Task 2.2): valid signed `.fota` passes TLS + Ed25519 + SHA-256 verification; a 1-byte tampered `.fota` is rejected with `VERIFY_FAILED` telemetry POSTed to Server and zero CAN frames emitted.
   - [ ] PSKs and Ed25519 public key are loaded from runtime config paths (zero hardcoded secrets in source).
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to audit `https_client.c` (confirm `CURLOPT_SSL_VERIFYPEER=1` & `CURLOPT_SSL_VERIFYHOST=2`, confirm Ed25519 signs exact bytes `[0x00..0x3F]`, confirm SHA-256 is verified over full `image_size` before trusting payload) and `socketcan_port.c` (`CANFD_MTU` read/write handling).

### [ ] Task 3.4: Buildroot Custom Linux Image & `STM32MP157D-DK1` Device Tree in WSL2 (`gateway/linux/`)
**Dependencies:** Task 3.3 | **Estimated scope:** Medium (5 files)

1. **Goal & Deliverables:**
   - Configure a reproducible Buildroot external tree (`BR2_EXTERNAL`) inside WSL2 Ubuntu for `STM32MP157D-DK1`, enabling `m_can1` (`PA12`/`PA11`, `AF9`, disabling conflicting `i2c5`), packaging `gateway/app` (`gw-fota`) with `libcurl`/`openssl`/`can-utils`/`iproute2`, and producing a bootable microSD image (`sdcard.img`).
   - Files to write:
     - `gateway/linux/stm32mp157d_gw_defconfig` (minimal Buildroot defconfig derived from `stm32mp157c_dk2_defconfig` / DK1 base with `can-utils`, `iproute2`, `libcurl`, `openssl`, `openssh`/serial console)
     - `gateway/linux/dts/stm32mp157d-dk1-fota.dts` (enables `&m_can1` on `PA12`/`PA11` `AF9` and sets `&i2c5 { status = "disabled"; }`, matching `DS12504`/`UM2534` & SPEC §2)
     - `gateway/linux/rootfs_overlay/etc/network/interfaces` (Ethernet `eth0` static/DHCP + `can0` CAN-FD 500k/2M helper script)
     - `gateway/linux/package/gw-fota/gw-fota.mk` (Buildroot CMake package recipe cross-compiling `gateway/app` + `shared/`)
     - `gateway/linux/README_wsl2.md` (exact reproducible WSL2 commands: native ext4 `~/kltn-buildroot`, `make BR2_EXTERNAL=...`, `sdcard.img` flashing steps)
2. **Prerequisite Concepts & Official References:**
   - *The Buildroot User Manual* (`BR2_EXTERNAL` trees, custom packages, rootfs overlays, Linux kernel `CONFIG_CAN_M_CAN=y`, `CONFIG_CAN_M_CAN_PLATFORM=y`).
   - Linux Device Tree bindings for Bosch M_CAN (`Documentation/devicetree/bindings/net/can/bosch,m_can.yaml`).
   - Local hardware docs: `docs/references/DS_stm32mp157d.pdf` (`DS12504`), `docs/references/um2534-discovery-kits-with-stm32mp157-mpus-stmicroelectronics.pdf` (`UM2534` — CN2/CN13 Arduino/RPi header pinout & `PA11`/`PA12` conflict with `i2c5`).
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§2 Hardware Specifications, §13 #3).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] Buildroot build completes cleanly in WSL2 (`~/kltn-buildroot`) producing `output/images/sdcard.img` and cross-compiled `gw-fota` ARM binary (`file` confirms `ARM, EABI5`).
   - [ ] `dtc` (Device Tree Compiler) compiles `stm32mp157d-dk1-fota.dts` with zero errors/pinmux conflicts.
   - [ ] *(When board is connected)* Booting `sdcard.img` on `STM32MP157D-DK1` shows `can0` in `ip -details link show can0` and accepts `ip link set can0 up type can bitrate 500000 dbitrate 2000000 fd on`.
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to audit `stm32mp157d-dk1-fota.dts` pinmux (`PA12`/`PA11` `AF9` vs `i2c5` disable) against `DS12504`/`UM2534` and verify `gateway/linux/` contains zero duplicated C source files (per `CONSTRAINTS.md`).

#### Checkpoint 3: `gateway/` Complete
- [ ] All `gateway/app/core/` host unit tests pass under ASan/UBSan (`>= 80%` line coverage, `0` OS headers in `core/`).
- [ ] `gateway/app/` builds cleanly for both host Linux (WSL2) and ARM Cortex-A7 (Buildroot).
- [ ] `sdcard.img` generated cleanly in WSL2.

---

## Phase 4: `ecu/bootloader/` — STM32G474VET6 Custom Bootloader (`ecu-bootloader`)

### [ ] Task 4.1: Dual-Copy Ping-Pong Metadata & Freshness NVM Manager (`ecu/bootloader/core/`)
**Dependencies:** Task 1.1, Task 1.2, Task 1.6 | **Estimated scope:** Medium (5 files)

1. **Goal & Deliverables:**
   - Implement power-loss-safe dual-copy Bootloader Metadata management (Copy 0 @ `0x0800_C000`, Copy 1 @ `0x0800_D000`) and wear-leveled append-only Freshness/Epoch NVM logging (`0x0800_E000`–`0x0800_FFFF`) using only `flash_hal_t`.
   - Files to write:
     - `ecu/bootloader/core/boot_meta.h`
     - `ecu/bootloader/core/boot_meta.c` (load both copies, validate `magic` + `struct_version` + `CRC32`, pick higher `seq_num`; commit always erases and writes the **older** copy with `CRC32` in the final 8-byte double-word)
     - `ecu/bootloader/core/freshness_nvm.h`
     - `ecu/bootloader/core/freshness_nvm.c` (two 4 KB ping-pong sectors storing 16-byte double-word-aligned `{magic, boot_epoch, fv_checkpoint, crc32}` records; rolls over cleanly when a sector fills)
     - `tests/test_boot_meta.c` (unit tests including simulated power loss after erase, mid-write before final double-word, corrupted Copy 0 fallback to Copy 1, and freshness sector rollover)
2. **Prerequisite Concepts & Official References:**
   - STM32G4 Flash 64-bit double-word programming and ECC behavior (why writing to non-erased Flash triggers `PROGERR`/`ECCD`, and why writing `CRC32` in the final double-word makes metadata commits atomic).
   - Local hardware docs: `docs/references/rm0440-stm32g4-series-advanced-armbased-32bit-mcus-stmicroelectronics.pdf` (`RM0440`, Embedded Flash memory chapter — Dual-Bank `DBANK=1`, 2 KB page erase, 64-bit programming).
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§3.1, §3.2, §3.3 Power-Loss Safety Guarantee, §8 Example).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] Simulating power cut at *every* 8-byte double-word boundary during `boot_meta_commit()` always recovers the previous valid metadata copy on reboot (`100%` recovery across all 16 write steps).
   - [ ] Freshness NVM survives `> 600` sequential checkpoint writes (spanning multiple 4 KB sector rollovers) with monotonic `boot_epoch`/`fv` preserved across simulated reboots.
   - [ ] `ctest --test-dir build/host -R boot_meta --output-on-failure` passes under ASan/UBSan (`0` `malloc`, `0` STM32 headers).
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to audit `boot_meta.c` and `freshness_nvm.c` for: (a) 8-byte alignment compliance, (b) `seq_num` wrap-around safety, (c) erased-Flash `0xFF` detection, and (d) exact Flash addresses matching SPEC §3.1.

### [ ] Task 4.2: Fake 512 KB STM32G474 Flash Model & A/B Boot / Rollback FSM (`ecu/bootloader/core/`)
**Dependencies:** Task 1.3, Task 4.1 | **Estimated scope:** Medium (5 files)

1. **Goal & Deliverables:**
   - Build a realistic host in-RAM 512 KB STM32G474 Flash simulator (enforcing 2 KB page erase, `0xFF` erased state, double-word write-once rule, and injectable power-loss countdown) and implement the complete A/B Boot Decision, Trial-Boot & Auto-Rollback state machine (SPEC §3.3).
   - Files to write:
     - `tests/mocks/fake_stm32g4_flash.h`
     - `tests/mocks/fake_stm32g4_flash.c` (512 KB byte array mapped to `0x0800_0000..0x0807_FFFF`, rejects unaligned writes or writing to non-`0xFF` double-words, supports fault injection)
     - `ecu/bootloader/core/boot_fsm.h`
     - `ecu/bootloader/core/boot_fsm.c` (implements §3.3: `BKP0R == 0x464F5441` → ProgSession; pending trial boot check `boot_attempts < 3`, TinyJAMBU full-image verify, MSP & Thumb Reset-Vector validation, increment `boot_attempts` + save shadow meta + arm IWDG before jump; auto-rollback to `active_slot` on 3 failures or corrupt image with `last_rollback_reason`; Recovery Mode if both slots invalid)
     - `tests/test_boot_fsm.c` (tests all state transitions: normal boot, `BKP0R` entry, trial boot #1/#2/#3 → rollback on attempt 4, bad MSP/Reset Vector rejection, corrupted image tag rollback, both-slots-invalid Recovery Mode)
2. **Prerequisite Concepts & Official References:**
   - ARM Cortex-M4 Vector Table (`[0]` = Initial MSP in SRAM `0x2000_0000..0x2002_0000`, `[1]` = Reset Handler with Thumb LSB `== 1`).
   - Local hardware docs: `docs/references/pm0214-stm32-cortexm4-mcus-and-mpus-programming-manual-stmicroelectronics.pdf` (`PM0214`, Vector Table & `VTOR`), [docs/pdf/01_MCUStartupAndRAM.pdf](../pdf/01_MCUStartupAndRAM.pdf).
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§3.3).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] Fake Flash rejects writing to an unerased double-word and rejects non-8-byte-aligned addresses (catching real STM32G4 hardware faults on host PC!).
   - [ ] Full 3-strike rollback test: pending Slot B fails to confirm 3 times → on 4th boot `boot_fsm` marks Slot B `SLOT_STATE_INVALID`, sets `last_rollback_reason = 1`, saves shadow metadata, and selects Slot A.
   - [ ] `ctest --test-dir build/host -R boot_fsm --output-on-failure` passes under ASan/UBSan (`>= 80%` coverage).
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to audit `boot_fsm.c` against the state diagram in SPEC §3.3 line-by-line, especially verifying that `boot_attempts` is persisted to Flash **before** jumping to the trial image.

### [ ] Task 4.3: Bootloader UDS Programming Server (`ecu/bootloader/core/`)
**Dependencies:** Task 1.4, Task 1.5, Task 1.6, Task 4.1, Task 4.2 | **Estimated scope:** Small (3 files)

1. **Goal & Deliverables:**
   - Implement the pure-C11 Bootloader UDS Programming Server handling the complete flashing session over ISO-TP: `0x10`, `0x11`, `0x22 0xF195`, `0x27` SecurityAccess (with 3-failure / 10 s lockout), `0x31 0xFF00` Erase Inactive Slot (with NRC `0x78`), `0x34` RequestDownload, `0x36` TransferData (TinyJAMBU-128 AEAD decrypt + double-word Flash write), `0x37` RequestTransferExit, `0x31 0x0202` Verify & Commit (full-image tag + anti-downgrade), and `0x3E` TesterPresent (`S3_server = 5000 ms`).
   - Files to write:
     - `ecu/bootloader/core/uds_prog_server.h`
     - `ecu/bootloader/core/uds_prog_server.c`
     - `tests/test_uds_prog_server.c` (uses `fake_stm32g4_flash` + real `fota_master` from Task 3.2 back-to-back on host!)
2. **Prerequisite Concepts & Official References:**
   - ISO 14229-1 Programming Session security & sequence enforcement (requiring `0x27` unlocked before `0x31`/`0x34`/`0x36`, rejecting active-slot erase with NRC `0x31`, rejecting downgrade version with NRC `0x22`/`0x72`, rejecting bad MAC with NRC `0x72` without writing Flash).
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§4.5, §4.6).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] Host integration test pairing `fota_master` (Task 3.2) ↔ `uds_prog_server` (Task 4.3) over a mock CAN-FD queue flashes a valid dummy binary into inactive Slot B in `fake_stm32g4_flash`, verifies `SLOT_STATE_PENDING_TEST`, and passes `boot_fsm` validation.
   - [ ] Negative tests confirm: (a) 3 bad `0x27` keys trigger NRC `0x36` and 10 s lockout (`0x37`), (b) erasing active slot returns NRC `0x31`, (c) 1-bit tampered `0x36` block returns NRC `0x72` and writes `0` bytes to Flash, (d) downgrade `fw_version <= active_version` is rejected at `0x31 0x0202`, (e) `S3_server` timeout exits Programming Session cleanly.
   - [ ] `ctest --test-dir build/host -R uds_prog --output-on-failure` passes under ASan/UBSan (`>= 80%` coverage, `0` `malloc`, `0` STM32 headers).
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to perform a security audit on `uds_prog_server.c` checking: state-skip attacks (calling `0x36` without `0x34` or `0x27`), integer overflows in `addr + offset + len`, partial last-block `< 8 B` padding with `0xFF` for 64-bit Flash writes, and constant-time key verification.

### [ ] Task 4.4: Hardware Bring-Up Confirmation & STM32G474 Bootloader Toolchain / Linker Script
**Dependencies:** Task 4.3 | **Estimated scope:** Medium (5 files)

1. **Goal & Deliverables:**
   - **Close the two hardware `TBD`s from SPEC §13:** inspect your physical `STM32G474VET6` core board to confirm (1) external `HSE` crystal frequency (`8 MHz` or `24 MHz`) and (2) `FDCAN1` GPIO pair (`PA11/PA12` `AF9` or `PD0/PD1` `AF9`, never `PB8/BOOT0`), document wiring in `hardware/pinout_wiring.md`, and update `SPEC.md` §13.
   - Create the `arm-none-eabi-gcc` CMake toolchain, Bootloader linker script (`0x0800_0000`, 48 KB partition), and STM32G474 `170 MHz` HSE+PLL + `DBANK=1` clock/startup setup.
   - Files to write:
     - `hardware/pinout_wiring.md` (locked STM32G474VET6 + STM32MP157D-DK1 + 2× MCP2562FD pinout table, power rails `VDD=5 V` / `VIO=3.3 V`, `2 × 120 Ω` termination)
     - `ecu/cmake/arm-none-eabi.cmake` (Cortex-M4 + FPU `-mcpu=cortex-m4 -mfpu=fpv4-sp-d16 -mfloat-abi=hard`, `-Os`, `-ffunction-sections -fdata-sections`, `--gc-sections`, `--specs=nosys.specs`)
     - `ecu/cmake/stm32g474_bootloader.ld` (`FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 48K`; `RAM (xrw) : ORIGIN = 0x20000000, LENGTH = 16K` budget guard + CCM-SRAM section if used for Flash routines)
     - `ecu/bootloader/port/board_config.h` (single source of truth for locked `HSE_VALUE`, `FDCAN1` pins/AF, status LED pin, PLL dividers, and FDCAN bit timings)
     - `ecu/bootloader/port/startup_clock.c` (reset vector/startup, HSE → PLL `170 MHz`, Flash latency 4 WS, `FDCAN` kernel clock selection, `DBANK=1` option-byte assertion)
2. **Prerequisite Concepts & Official References:**
   - ARM ELF linking, `.isr_vector`, `.text`, `.rodata`, `.data` (LMA in Flash → VMA in SRAM), `.bss` zeroing.
   - Local hardware docs: `docs/references/DS_stm32g474ve.pdf` (`DS12288` pinout/AF table), `docs/references/rm0440-stm32g4-series-advanced-armbased-32bit-mcus-stmicroelectronics.pdf` (`RM0440` RCC/PLL 170 MHz Range 1 Boost mode, Flash wait states, Option Bytes `DBANK`), `docs/references/MCP2562FD_Datasheet.pdf` (`DS20005284A`), `docs/references/TIS_ELF_Specification_v1.2.pdf`, [docs/pdf/00_HardwareCompatibility.pdf](../pdf/00_HardwareCompatibility.pdf), [docs/pdf/01_MCUStartupAndRAM.pdf](../pdf/01_MCUStartupAndRAM.pdf).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] Both `TBD` items (`FDCAN1` pins & `HSE` frequency) are verified on the physical board and updated in `hardware/pinout_wiring.md` and `docs/architecture/SPEC.md` §13.
   - [ ] Minimal blinky/clock bring-up links cleanly with `stm32g474_bootloader.ld` under `arm-none-eabi-gcc` (`0` warnings) and boots at `170 MHz` on `STM32G474VET6` via ST-LINK SWD.
   - [ ] `FLASH->OPTR & FLASH_OPTR_DBANK` is verified `== 1` (2 KB pages, Dual-Bank active).
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to verify your `HSE → PLL 170 MHz` math (VCO range, `R`/`Q` dividers, Range 1 Boost mode `PWR_CR5_R1MODE`), `FDCAN` bit-timing sample points (`80%` @ 500 kbps, `75%` @ 2 Mbps), and linker script `MEMORY` bounds against `RM0440` and `SPEC.md` §3.1.

### [ ] Task 4.5: STM32G474 Bootloader Peripheral Drivers & Safe Jump-to-App (`ecu/bootloader/port/`)
**Dependencies:** Task 4.3, Task 4.4 | **Estimated scope:** Medium (5 files)

1. **Goal & Deliverables:**
   - Implement the thin STM32G474 hardware port wiring `ecu/bootloader/core/` to real hardware peripherals: 64-bit ECC Flash driver (handling Bank 1 & Bank 2 page mapping), non-blocking `FDCAN1` driver with ISR→ring-buffer RX, hardware `RNG` + `IWDG` (`4 s`) + `TAMP->BKP0R`, and the 5-step `SCB->VTOR` / `__set_MSP` Jump-to-App routine.
   - Files to write:
     - `ecu/bootloader/port/flash_stm32g4.c` (implements `flash_hal_t`; handles Bank 1 `0x0800_0000..0x0803_FFFF` vs Bank 2 `0x0804_0000..0x0807_FFFF` `BKER`/`PNB` bits; places critical Bank-1 erase/write helper in `.ccmram` / RAM function so Slot A Bank-1 pages 32–127 erase safely)
     - `ecu/bootloader/port/fdcan_stm32g4.c` (implements `can_hal_t`; configures `FDCAN1` 500 kbps / 2 Mbps, hardware filter for `0x7DF`/`0x7E0`/`0x200`; ISR strictly copies received frame into lock-free static ring buffer — zero parsing/crypto/Flash in ISR!)
     - `ecu/bootloader/port/rng_iwdg_bkp.c` (implements `rng_hal_t` & `timer_hal_t`; `RNG` peripheral, `IWDG` 4 s timeout, `TAMP->BKP0R` unlock/read/write, `SysTick` 1 ms counter)
     - `ecu/bootloader/port/boot_jump.c` (implements SPEC §3.3 5-step Jump-to-App checklist: validate MSP/Reset Vector, `__disable_irq()`, stop SysTick, de-init `FDCAN1`/clocks, clear NVIC pending bits, `__DSB(); __ISB();`, `SCB->VTOR = slot_base`, `__set_MSP()`, jump)
     - `ecu/bootloader/port/main.c` (thin Bootloader entry wiring HAL tables to `boot_fsm` & `uds_prog_server`)
2. **Prerequisite Concepts & Official References:**
   - STM32G4 `FDCAN` Message RAM layout & bit-timing registers (`NBTP`, `DBTP`, `TDCR` transceiver delay compensation at 2 Mbps).
   - STM32G4 `FLASH_CR` (`BKER`, `PNB`, `PER`, `PG`), `PWR_CR1_DBP` for `TAMP->BKP0R`, `IWDG` prescaler/reload math (`LSI ~32 kHz`), and Cortex-M4 interrupt cleanup before jumping from Bootloader to App.
   - Local hardware docs: `RM0440` (Flash, FDCAN, RNG, IWDG, TAMP/RTC chapters), `PM0214` (`VTOR`, `DSB`/`ISB`, `MSP`), `an2606-stm32-microcontroller-system-memory-boot-mode-stmicroelectronics.pdf` (`AN2606`), [docs/pdf/03_InterruptInEmbedded.pdf](../pdf/03_InterruptInEmbedded.pdf).
   - Contract: [CONSTRAINTS.md](../../CONSTRAINTS.md) (§ISR safety, §Embedded Heap Ban, §ECU Bootloader Footprint).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] `cmake -B build/ecu -DCMAKE_TOOLCHAIN_FILE=ecu/cmake/arm-none-eabi.cmake -DCMAKE_BUILD_TYPE=MinSizeRel && cmake --build build/ecu` builds `ecu_bootloader.elf` with `0` warnings.
   - [ ] `arm-none-eabi-size --format=berkeley build/ecu/ecu_bootloader.elf` confirms `.text + .rodata + .data <= 32768` bytes (`<= 32 KB`) and `.data + .bss <= 16384` bytes (`<= 16 KB`).
   - [ ] `arm-none-eabi-nm build/ecu/ecu_bootloader.elf | grep -E "(_sbrk|malloc|free|calloc|realloc)"` returns empty (`0` heap symbols).
   - [ ] Hardware bench smoke test: flashing `ecu_bootloader.elf` to `STM32G474VET6` enters Recovery Mode (empty slots) and responds to `0x22 0xF195` over CAN-FD.
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to audit: (a) `flash_stm32g4.c` Bank 1/2 `BKER`/`PNB` address math (especially Slot A crossing `0x0804_0000`!), (b) `fdcan_stm32g4.c` ISR compliance with `CONSTRAINTS.md` ISR rules, and (c) `boot_jump.c` peripheral/NVIC teardown sequence.

#### Checkpoint 4: `ecu/bootloader/` Complete — Gate Promotion
- [ ] All Bootloader host unit tests pass (`100%`, ASan/UBSan, `>= 80%` coverage).
- [ ] `ecu_bootloader.elf` passes footprint (`<= 32 KB` Flash, `<= 16 KB` RAM) and `0` `_sbrk` checks.
- [ ] **Constraint Update (Ask User/Agent):** Promote `ECU Bootloader Footprint` in [CONSTRAINTS.md](../../CONSTRAINTS.md) from `Warn` to `Block` and record exact Flash/RAM baseline in the Ratchets table.

---

## Phase 5: `ecu/app/` — STM32G474VET6 Target Application (`ecu-app`)

### [ ] Task 5.1: ECU App UDS Server, SecOC Heartbeat & Post-Boot Confirmation Core (`ecu/app/core/`)
**Dependencies:** Task 1.4, Task 1.5, Task 1.6, Task 4.1 | **Estimated scope:** Medium (5 files)

1. **Goal & Deliverables:**
   - Implement the pure-C11 Target ECU Application core logic: App-mode UDS server (`0x22 0xF195`, `0x10 0x03`, `0x31 0x0203` preconditions, `0x10 0x02` Programming Session handoff via `BKP0R` + reset, `0x3E`), SecOC-authenticated `0x101` heartbeat & `0x110` simulated sensor frames, `0x200`/`0x201` FV sync, and post-boot self-test + `fota_confirm_boot()` (transitioning `SLOT_STATE_PENDING_TEST` → `SLOT_STATE_VALID_CONFIRMED` in shadow Metadata while kicking `IWDG`).
   - Files to write:
     - `ecu/app/core/app_uds.h`
     - `ecu/app/core/app_uds.c`
     - `ecu/app/core/app_secoc.h`
     - `ecu/app/core/app_secoc.c`
     - `tests/test_ecu_app_core.c` (tests `fota_confirm_boot()` updates shadow metadata once and stays idempotent on subsequent boots, `0x10 0x02` sets `BOOT_REQ_MAGIC` only after `0x10 0x03` + `0x31 0x0203`, `0x101`/`0x110` SecOC frames verify with `secoc.c`, fault-injection mode skips confirm to trigger rollback)
2. **Prerequisite Concepts & Official References:**
   - UDS Session transitions (`Default 0x01` → `Extended 0x03` → `Programming 0x02` handoff to Bootloader via backup register).
   - Trial-boot confirmation invariant (confirm only after peripheral/CAN self-test succeeds; never confirm immediately at reset before verifying system health).
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§3.3, §4.2, §4.5 steps 1–4 & 12).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] `fota_confirm_boot()` transitions `PENDING_TEST` → `VALID_CONFIRMED`, sets `active_slot = pending_slot`, clears `pending_slot = NONE`, and commits shadow metadata using `boot_meta_commit()` (zero extra Flash wear on subsequent confirmed boots).
   - [ ] `0x10 0x02` without prior `0x31 0x0203` precondition check returns NRC `0x22` (`ConditionsNotCorrect`).
   - [ ] `ctest --test-dir build/host -R ecu_app --output-on-failure` passes under ASan/UBSan (`>= 80%` coverage, `0` `malloc`, `0` STM32 headers).
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to audit `app_uds.c` and `app_secoc.c` for: (a) ensuring `0x50 0x02` positive response finishes transmitting on CAN-FD **before** `NVIC_SystemReset()`, and (b) ensuring `fota_confirm_boot()` does not erase/rewrite Metadata on normal already-confirmed boots (Flash endurance).

### [ ] Task 5.2: Slot A / Slot B Linker Scripts & STM32G474 App Port (`ecu/app/port/`)
**Dependencies:** Task 4.4, Task 4.5, Task 5.1 | **Estimated scope:** Medium (5 files)

1. **Goal & Deliverables:**
   - Create separate Slot A (`0x0801_0000`, 224 KB) and Slot B (`0x0804_8000`, 224 KB) linker scripts, the thin STM32G474 App hardware port (reusing `board_config.h` and HAL drivers), and CMake targets producing `ecu_app_slotA.bin` & `ecu_app_slotB.bin` (plus an optional `-DFAULT_INJECT_ROLLBACK=ON` build variant for testing rollback with a deliberately faulty `v1.2.0` binary).
   - Files to write:
     - `ecu/cmake/stm32g474_slotA.ld` (`FLASH (rx) : ORIGIN = 0x08010000, LENGTH = 224K`; `RAM (xrw) : ORIGIN = 0x20000000, LENGTH = 64K` static budget)
     - `ecu/cmake/stm32g474_slotB.ld` (`FLASH (rx) : ORIGIN = 0x08048000, LENGTH = 224K`; `RAM (xrw) : ORIGIN = 0x20000000, LENGTH = 64K` static budget)
     - `ecu/app/port/app_fdcan_gpio.c` (thin App FDCAN1 + status LED + `IWDG` refresh bindings)
     - `ecu/app/port/main.c` (re-verifies `SCB->VTOR`, initializes clocks/FDCAN1, runs self-test → `fota_confirm_boot()` unless `FAULT_INJECT_ROLLBACK`, runs main non-blocking poll loop)
     - `ecu/app/CMakeLists.txt` (builds `ecu_app_slotA.elf/.bin`, `ecu_app_slotB.elf/.bin`, and optional `ecu_app_faulty_slotA/B.bin`)
2. **Prerequisite Concepts & Official References:**
   - Cortex-M4 `SCB->VTOR` alignment requirements (`0x0801_0000` is 64 KB aligned; `0x0804_8000` is 32 KB aligned — both satisfy `VTOR` alignment per `PM0214`).
   - `arm-none-eabi-objcopy -O binary` and `arm-none-eabi-objdump -h` to inspect `.isr_vector` VMA/LMA.
   - Local hardware docs: `PM0214`, `RM0440`, `ARM_ELF_for_Arm_Architecture_aaelf32.pdf`.
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§3.1 Slot-Specific Binaries, §6 command #2).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] `cmake --build build/ecu` builds `ecu_app_slotA.elf/.bin` and `ecu_app_slotB.elf/.bin` with `0` warnings.
   - [ ] `arm-none-eabi-size --format=berkeley build/ecu/ecu_app_slotA.elf build/ecu/ecu_app_slotB.elf` confirms Flash `<= 229376` bytes (`<= 224 KB`) and static SRAM `<= 65536` bytes (`<= 64 KB`), with `0` `_sbrk`/`malloc` symbols.
   - [ ] `arm-none-eabi-objdump -h` confirms `.isr_vector` VMA is `0x08010000` for Slot A and `0x08048000` for Slot B.
   - [ ] Hardware smoke test: flash Bootloader + `ecu_app_slotA.bin` (`v1.0.0`) + initial valid Metadata via ST-LINK; confirm Bootloader jumps cleanly to Slot A and App broadcasts authenticated `0x101` heartbeats.
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to inspect `arm-none-eabi-objdump -h` output and `stm32g474_slotA.ld` / `stm32g474_slotB.ld` to verify zero overlap with Bootloader (`0x0800_0000..0x0800_BFFF`) or Metadata/NVM (`0x0800_C000..0x0800_FFFF`).

#### Checkpoint 5: `ecu/app/` Complete — Gate Promotion
- [ ] All `ecu/app/core/` host unit tests pass (`100%`, ASan/UBSan, `>= 80%` coverage).
- [ ] Both Slot A & Slot B binaries pass footprint and `0` `_sbrk` checks; Bootloader → Slot A hardware jump verified.
- [ ] **Constraint Update (Ask User/Agent):** Promote `ECU App Footprint` in [CONSTRAINTS.md](../../CONSTRAINTS.md) from `Warn` to `Block` and record exact Flash/RAM baselines in the Ratchets table.

---

## Phase 6: System Integration — SIL (`vcan0`) → HIL Bench Verification & Thesis Benchmarks

### [ ] Task 6.1: Software-in-the-Loop (SIL) End-to-End Verification (`tests/sil/`)
**Dependencies:** Tasks 2.2, 3.3, 4.3, 5.1 | **Estimated scope:** Small (3 files)

1. **Goal & Deliverables:**
   - Assemble a host-compiled Simulated ECU running the **exact** `ecu/bootloader/core/` + `ecu/app/core/` + `fake_stm32g4_flash` over Linux SocketCAN `vcan0` (or loopback socket if WSL2 kernel lacks `vcan.ko`, documented in script), and run end-to-end automated scenarios with the real FastAPI HTTPS Server and real Gateway C11 app.
   - Files to write:
     - `tests/sil/ecu_sim_main.c` (wraps Bootloader Core + App Core + `fake_stm32g4_flash` with `socketcan_port`, simulating `NVIC_SystemReset()`, `IWDG` timeout, and mid-transfer power loss)
     - `tests/sil/CMakeLists.txt`
     - `tests/sil/run_sil_e2e.sh` (automated test runner executing 5 E2E scenarios and checking exit codes + server telemetry logs)
2. **Prerequisite Concepts & Official References:**
   - Linux `vcan` virtual CAN-FD setup (`sudo modprobe vcan && sudo ip link add dev vcan0 type vcan && sudo ip link set vcan0 mtu 72 up`) and WSL2 kernel module caveat (if default WSL2 kernel omits `vcan`, either enable in custom WSL2 kernel or use a small UDP/Unix-socket `can_hal_t` loopback bridge).
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§9 Level 3 SIL, §11 Success Criteria #4, #5, #6).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] Scenario 1 (Happy Path): `v1.0.0` (Slot A) → signed `v1.1.0` (Slot B) completes and confirms `SLOT_STATE_VALID_CONFIRMED` with telemetry `CONFIRMED` on FastAPI Server.
   - [ ] Scenario 2 (Tampered `.fota`): 1-byte corrupted package rejected by Gateway (`VERIFY_FAILED` telemetry, `0` Flash writes).
   - [ ] Scenario 3 (Spoof / Replay Attack): replayed `0x101` and spoofed CAN ID dropped & logged by Gateway/ECU.
   - [ ] Scenario 4 (Faulty Firmware Auto-Rollback): `v1.2.0` failing to confirm triggers automatic rollback after 3 attempts back to `v1.1.0` (`last_rollback_reason = 1`).
   - [ ] Scenario 5 (Power-Cut Mid-`0x36`): killing `ecu_sim` mid-transfer leaves `v1.1.0` immediately bootable on restart.
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to review the SIL logs and `run_sil_e2e.sh` assertions to confirm every V1 Success Criterion in SPEC §11 is tested without mocked shortcuts.

### [ ] Task 6.2: Hardware-in-the-Loop (HIL) Bench Bring-Up & Live FOTA Validation
**Dependencies:** Tasks 3.4, 4.5, 5.2, 6.1 | **Estimated scope:** Small (3 files)

1. **Goal & Deliverables:**
   - Wire the complete physical bench (`Host PC FastAPI Server` ↔ Ethernet ↔ `STM32MP157D-DK1` Buildroot Gateway ↔ `MCP2562FD` ↔ `CANH/CANL + 2 × 120 Ω` ↔ `MCP2562FD` ↔ `STM32G474VET6` Target ECU), execute every SPEC §11 acceptance test on live hardware, and record reproducible evidence (terminal logs, `candump` captures, photos/scope traces) for your thesis.
   - Files to write:
     - `hardware/bom.md` (final Bill of Materials with exact part numbers used)
     - `hardware/bench_bringup_checklist.md` (step-by-step power-rail check `5 V`/`3.3 V`, `60 Ω` bus resistance check with power off, `candump` smoke test, ST-LINK provisioning commands)
     - `docs/architecture/hil_test_report.md` (pass/fail table and captured logs for all 6 SPEC §11 scenarios on real hardware)
2. **Prerequisite Concepts & Official References:**
   - CAN-FD physical layer troubleshooting: `60 Ω` DC resistance across `CANH`–`CANL` (`2 × 120 Ω` parallel), common ground between transceivers, `MCP2562FD` `STBY` pin pulled LOW (normal mode), `VDD = 5 V` & `VIO = 3.3 V` decoupling capacitors.
   - Local hardware docs: `MCP2562FD_Datasheet.pdf` (`DS20005284A`), `UM2534`, `DS12288`.
   - Spec: [docs/architecture/SPEC.md](SPEC.md) (§9 Level 4 HIL, §11 Success Criteria).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] Physical bus check: multimeter reads `~60 Ω` between `CANH` and `CANL` (unpowered); `candump can0` on `STM32MP157D-DK1` receives valid `0x101` CAN-FD frames (`500 kbps / 2 Mbps`, `BRS=1`) from `STM32G474VET6` with `0` bus-off/error-passive events.
   - [ ] Live HIL FOTA Happy Path: `v1.0.0` (Slot A) → `v1.1.0` (Slot B) completes and confirms via `0x22 0xF195` & FastAPI telemetry.
   - [ ] Live HIL Auto-Rollback: flashing faulty `v1.2.0` (`FAULT_INJECT_ROLLBACK=ON`) triggers 3 IWDG resets and automatically restores `v1.1.0` (`last_rollback_reason = 1`).
   - [ ] Live HIL Power-Cut Recovery: physically unplugging `STM32G474VET6` power mid-`0x36` and reconnecting immediately boots `v1.1.0`.
   - [ ] Live HIL Attack Demo: injecting a replayed/spoofed frame via `cansend` and serving a tampered `.fota` are both rejected and logged.
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to audit `docs/architecture/hil_test_report.md` against all 6 items in `SPEC.md` §11 and identify any missing evidence or figures needed for your thesis report (`docs/latex/`).

### [ ] Task 6.3: Performance Benchmarking & Thesis Data Collection
**Dependencies:** Task 6.2 | **Estimated scope:** Small (2 files)

1. **Goal & Deliverables:**
   - Measure and document quantitative performance metrics on the real `STM32G474VET6` (`170 MHz`) and `STM32MP157D-DK1` hardware to back up the thesis evaluation chapter and populate the final ratchet in `CONSTRAINTS.md`.
   - Files to write:
     - `tests/bench/bench_tinyjambu.c` (cycle-accurate benchmark using ARM Cortex-M4 `DWT->CYCCNT` on STM32G474 and `clock_gettime` on host/MP157 for 64 B SecOC frame & 2048 B UDS `0x36` block)
     - `docs/architecture/benchmark_results.md` (tables of: TinyJAMBU-128 latency `µs` & cycles/byte, total FOTA time for 50 KB / 100 KB / 224 KB images at 500k/2M CAN-FD, Bootloader & App Flash/RAM breakdown, rollback detection/recovery time)
2. **Prerequisite Concepts & Official References:**
   - ARM Cortex-M4 Data Watchpoint and Trace cycle counter (`CoreDebug->DEMCR |= TRCENA`, `DWT->CTRL |= CYCCNTENA`, `DWT->CYCCNT` per `PM0214` / Arm v7-M Architecture Reference Manual).
   - Spec & Contract: [CONSTRAINTS.md](../../CONSTRAINTS.md) (§Measured, Not Yet Enforced — TinyJAMBU-128 latency ratchet).
3. **Acceptance & Self-Verification Criteria:**
   - [ ] `DWT->CYCCNT` measurements recorded on `STM32G474VET6 @ 170 MHz` (average + worst-case over 1000 iterations) for 64-byte frame and 2048-byte FOTA block.
   - [ ] End-to-end FOTA time breakdown recorded (HTTPS download + Ed25519 verify + Flash erase + CAN-FD transfer + full-image verify + reboot confirm).
   - [ ] `CONSTRAINTS.md` Ratchets table updated with final measured baselines.
4. **Review Checkpoint (Ask the Agent):**
   - Ask the agent to sanity-check benchmark math (cycles → `µs` at `170 MHz`, effective CAN-FD payload throughput vs theoretical 2 Mbps limit) for consistency before inserting tables/charts into `docs/latex/`.

#### Checkpoint 6: V1 Complete — Final Thesis Sign-Off
- [ ] All 6 Success Criteria in [docs/architecture/SPEC.md](SPEC.md) §11 passed on live HIL hardware.
- [ ] Every row in [CONSTRAINTS.md](../../CONSTRAINTS.md) enforced at `Block` with `0` violations and all Ratchets populated.
- [ ] `docs/architecture/SPEC.md`, `hardware/pinout_wiring.md`, `docs/architecture/benchmark_results.md`, and `docs/latex/` figures/tables fully synchronized.

---

## 5. Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| **Hardware TBDs (`FDCAN1` pins / `HSE` crystal) mismatch physical board** | High | Isolated inside `ecu/bootloader/port/board_config.h`; verified physically in Task 4.4 before any Flash/FDCAN hardware bring-up. Never use `PB8` (`BOOT0`). |
| **Bank-1 Flash erase stall when flashing Slot A (`0x0801_0000..0x0803_FFFF`) from Bootloader (`0x0800_0000`)** | High | Place critical Bank-1 erase/program loop in `.ccmram` / SRAM (`__attribute__((section(".ramfunc")))`) in Task 4.5, disable interrupts during single-page erase trigger, and emit UDS NRC `0x78` (`ResponsePending`) before erasing. |
| **STM32G474 Option Byte `DBANK=0` (Single-Bank 4 KB pages) out of factory** | High | Explicitly check `FLASH->OPTR & FLASH_OPTR_DBANK` in Task 4.4 startup and document ST-LINK option-byte configuration (`DBANK=1`) in `bench_bringup_checklist.md`. |
| **TinyJAMBU-128 bit/byte-order mismatch against NIST LWC spec** | High | Validate against official NIST LWC `LWC_AEAD_KAT_128_96.txt` vectors in Task 1.3 *before* writing SecOC or UDS code. |
| **Buildroot slow or broken on Windows `/mnt/c` under WSL2** | Medium | Always build Buildroot inside WSL2's native Linux ext4 filesystem (`~/kltn-buildroot`) with `BR2_EXTERNAL` pointing to the repo's `gateway/linux/`. |
| **Default WSL2 kernel missing `vcan` module for SIL** | Medium | Document both enabling `CONFIG_CAN_VCAN` in WSL2 and providing a fallback loopback `can_hal_t` (Unix domain socket) in Task 6.1 so SIL tests never block on WSL2 kernel rebuilds. |
| **Host Windows MinGW missing AddressSanitizer (`-fsanitize=address`)** | Medium | Run full ASan/UBSan host CTest suite inside WSL2 Ubuntu `gcc` at every Task Checkpoint (native Windows `gcc` still runs fast warning/unit-test edit loops). |
| **Accidental secret/key commit to Git** | High | Enforce `.gitignore` rules for `server/secrets/`, `server/certs/`, `*.pem`, `*.key` plus automated check in `scripts/floor_guard.py`. |

---

## 6. Deferred Confirmations (Tracked Until Task 4.4)

- [ ] **Task 4.4 Gate:** Confirm `STM32G474VET6` `FDCAN1` GPIO pair (`PA11/PA12` `AF9` vs `PD0/PD1` `AF9`) on physical board → update `board_config.h`, `hardware/pinout_wiring.md`, and `SPEC.md` §13.
- [ ] **Task 4.4 Gate:** Confirm `STM32G474VET6` `HSE` crystal frequency (`8 MHz` vs `24 MHz`) on physical board → update `board_config.h` and `SPEC.md` §13.
