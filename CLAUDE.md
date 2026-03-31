# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is the FreeRTOS main repository, containing the FreeRTOS real-time kernel (as a git submodule), supplementary libraries (FreeRTOS-Plus), demos, and test suites. The kernel is MIT-licensed. Many components are managed as git submodules (29 total, defined in `.gitmodules`).

**Clone with submodules:** `git clone --recurse-submodules` (or `git submodule update --init --recursive` after cloning).

## Architecture

```
FreeRTOS/Source/           → Kernel (submodule: FreeRTOS-Kernel) — tasks, queue, list, timers, etc.
FreeRTOS/Demo/             → 200+ kernel demos organized by <ARCH>_<BOARD>_<TOOLCHAIN>
FreeRTOS/Test/             → Kernel test suites (CMock, CBMC, VeriFast, Target)

FreeRTOS-Plus/Source/      → Supplementary libraries (TCP, MQTT, HTTP, AWS IoT, cellular, CLI, etc.)
FreeRTOS-Plus/Demo/        → Plus library demos
FreeRTOS-Plus/Test/        → Plus library tests
FreeRTOS-Plus/ThirdParty/  → mbedtls, wolfSSL, glib, libslirp, tinycbor

tools/                     → Uncrustify config, AWS config helpers
```

**Dependency chain:** Kernel → FreeRTOS-Plus-TCP → Application Protocols (coreMQTT, coreHTTP, coreSNTP) → AWS IoT libraries. TLS via mbedtls/wolfSSL through corePKCS11.

## Build Commands

There is no top-level build system. Each demo and test has its own build.

### POSIX demo (Linux)
```bash
cd FreeRTOS/Demo/Posix_GCC && mkdir build && cd build && cmake .. && make
```

### FreeRTOS-Plus TCP echo demo (POSIX)
```bash
cd FreeRTOS-Plus/Demo/FreeRTOS_Plus_TCP_Echo_Posix && make
```

### CMock Kernel Unit Tests
```bash
# Run all kernel unit tests (requires CMock submodule initialized)
make -C FreeRTOS/Test/CMock run_col_formatted

# Run with address sanitizer
make -C FreeRTOS/Test/CMock ENABLE_SANITIZER=1 run_col_formatted

# Run a single test unit (e.g., queue)
make -C FreeRTOS/Test/CMock queue

# Generate coverage report
make -C FreeRTOS/Test/CMock lcovhtml
```

Test units: `timers`, `tasks`, `list`, `queue`, `stream_buffer`, `message_buffer`, `event_groups`, `smp`.

### CBMC Proofs (memory safety verification)
```bash
cd FreeRTOS/Test/CBMC && python3 prepare.py   # generate per-proof Makefiles
cd proofs/<area>/<function> && make            # run individual proof
```

## Code Style

- Follow the [FreeRTOS Coding Standard and Style Guide](https://www.freertos.org/FreeRTOS-Coding-Standard-and-Style-Guide.html)
- 4-space indentation for C code; tabs for Makefiles (see `.editorconfig`)
- Code formatter config: `tools/uncrustify.cfg`
- Spell-check config: `cspell.config.yaml` with custom dictionary at `.github/.cSpellWords.txt`

## CI Workflows

All defined in `.github/workflows/`:
- **ci.yml** — git-secrets scan, formatting check, spell check, Doxygen docs, manifest verification, CBMC proofs
- **kernel-unit-tests.yml** — CMock kernel unit tests with address sanitizer and coverage
- **freertos_demos.yml** — Build kernel demos (WIN32, POSIX, QEMU, Cortex-MPU)
- **freertos_plus_demos.yml** — Build Plus demos (Windows, POSIX, QEMU)
- **freertos_mpu_demo.yml** — Build ARM MPU demos with cross-compilation
- **core-checks.yml** — Validate file headers (PR only)

## Key Configuration Files

- `manifest.yml` — Pins exact versions for all submodules
- `.gitmodules` — Defines all 29 git submodules
- `.editorconfig` — Editor formatting rules
- `tools/uncrustify.cfg` — Detailed C code formatting rules

## Submodules

Kernel (`FreeRTOS/Source`), FreeRTOS-Plus-TCP, coreMQTT, coreHTTP, coreMQTT-Agent, coreSNTP, coreJSON, corePKCS11, mbedtls, wolfSSL, AWS IoT libraries (device-shadow, jobs, device-defender, ota, sigv4, fleet-provisioning), FreeRTOS-Cellular-Interface + modem drivers, CMock (two instances), and more.

Updating: Changes to submodule pins go in `manifest.yml` and `.gitmodules`.
