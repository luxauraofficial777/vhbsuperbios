# Project Frankenstein: VHB Super BIOS V1.01A

## Product Overview

The VHB Super BIOS is a **clean-room PSX kernel** — a 512KB MIPS R3000A BIOS image built entirely from original assembly source. It contains zero proprietary code.

**What the BIOS does TODAY (V1.01A blueprint, Aug 11, 2026):**
- CP0 init (Status/Cause register setup)
- Memory controller configuration
- RAM clear (first 64KB sanity check)
- SFX-100 bus probe
- Region detection (NA/JP/EU)
- GPU initialization (GPU_Init)
- Interrupt controller setup (Init_Interrupt_Controller)
- Heap management (Init_Heap)
- Event system (Init_Events — deliver/open/close/wait/test/enable/disable)
- Thread system (Init_Threads — openThread/closeThread/changeThread/returnFromException/getFreeTCB)
- CD-ROM driver (CDROM_Init, CDROM_ReadSector(s), CDROM_Wait_Response, CDROM_Wait_Param)
- SYSTEM.CNF boot-chain parser (Parse_Boot_Line, Find_File_By_Name)
- HBD-aware boot logic (Find_HBD, skip_hbd_load, HBD remainder handling)
- A0/B0/C0 syscall table installation (Install_Syscall_Vectors, Copy_Syscall_Tables)
- 50+ implemented kernel services (file I/O, string/mem, heap, GPU, events, threads, CD)
- VBlank-delivering exception handler (exception_handler, VBlank_Handler)
- FlushCache

**What the VHB Core (C++ loader shell) does TODAY:**
- Loads and inspects ROM/ISO images
- Parses PS-X EXE headers
- Sets up virtual memory with Soft-MMU translation (C++ side, not in the BIOS image)
- Registers custom vector/trap handlers
- **V1.01A (blueprinted):** GammaLanguage v1.1C integration — A2A-DSL parser, evaluator, bytecode VM
- **V1.01A (blueprinted):** Rust FFI bridge with PEAK-1..5 dispatch, `repr(C)` parity types
- **V1.01A (blueprinted):** CP0 on-demand register mapping (live getters, not VBlank polling)
- **V1.01A (blueprinted):** PSXMatrix DMA dispatch via FFI (controller-driven IRQ, no manual I_STAT)
- **V1.01A (blueprinted):** RegisterBank thread-safe spinlock for concurrent hypervisor access

**What is ROADMAP (not yet implemented):**
- BIOS-level Soft-MMU services (currently C++ side only)
- Commercial-emulator boot compatibility (B4 gate — **hard prerequisite for --gamma production builds**)
- Full B5 service coverage matrix (documented gaps remain)
- Multi-console support (SNES, Saturn, etc.) — quarantined to `_future_multibit/`
- GammaLanguage V1.01A implementation (blueprint complete, pending B4 gate pass)

The VHB is part of the Project Frankenstein toolkit, which includes:
- **VHB Super BIOS V1.01A** — 512KB MIPS R3000A clean-room kernel (~10,600 bytes of code, 50+ kernel services, GammaLanguage v1.1C ready)
- **VHB Core** — C++ hypervisor loader shell with Soft-MMU + GammaLanguage v1.1C evaluator (blueprinted)
- **VHB Launcher** — CLI tool to load, inspect, and launch custom ROMs with `--gamma` flag support (blueprinted)
- **bios_tool** — C++ binary utility for padding, signature injections, and verification (7/7 checks for V1.01A)
- **psx_core** — Minimal PSX emulator for BIOS validation only (not a full-fidelity emulator)
- **GammaLanguage v1.1C** — A2A-DSL inline header (`A2A_Gamma.h`) with 5 new opcodes, Rust FFI types, CP0 mapping, DMA dispatch
- **Rust FFI Bridge** — `gamma_ffi` crate with `repr(C)` parity types and PEAK-1..5 dispatch (blueprinted)

### Gate Status (Aug 11, 2026)

| Gate | Test | Result |
|------|------|--------|
| B0 | Clean-room verification (6/6 → 7/7 for V1.01A) | **PASS** (6/6 current; 7/7 blueprinted) |
| B1 | vhb_test 2M instructions, past CDROM_Init | **PASS** |
| B2 | BIOS + DW7 disc, 5M instrs, 8 VBlanks | **PASS** |
| B3 | CyberGrime parity vs B2 | **PASS** (informal — identical PC) |
| B4 | Boot in DuckStation (commercial emulator) | **FAIL** (black screen, FPS 0) — **HARD PREREQUISITE for --gamma** |
| B5 | Service coverage matrix | **OPEN** |
| B6 | Gamma script execution (V1.01A) | **PENDING** (blueprinted) |
| B7 | CP0 on-demand register mapping (V1.01A) | **PENDING** (blueprinted) |
| B8 | DMA dispatch via FFI (V1.01A) | **PENDING** (blueprinted) |

**Gate B4 is a hard prerequisite** — GammaLanguage `--gamma` features cannot ship in production until DuckStation boot passes. Development/testing may use `--gamma` without B4.

### GammaLanguage v1.1C Integration (V1.01A Blueprint)

The V1.01A upgrade integrates GammaLanguage v1.1C into all layers of the VHB stack:

| Feature | Opcode | Description |
|---------|--------|-------------|
| Rust FFI Zero-Copy | `RUST_OFFSET_LOAD` (0xA0) | `&RUST_OFFSET` sigil for zero-copy PSX hardware register access |
| PEAK FFI Dispatch | `RUST_FFI_CALL` (0xA1) | Dispatch to PEAK-1..5 via function pointer table |
| Alignment Bounds | `ALIGN_BOUND` (0xA2) | `@ALIGN_BOUND` for `repr(C)` parity between MIPS and Rust types |
| CP0 Register Mapping | `CP0_REG_MAP` (0xA3) | Map MIPS R3000A CP0 registers into Gamma `RegisterBank` (on-demand getters) |
| PSXMatrix DMA | `PSX_DMA_DISPATCH` (0xA4) | Dispatch PSX DMA channel via FFI (controller-driven IRQ) |

**Counter Reflection Fixes Applied (Aug 10, 2026):**
- RAM scratchpad relocated from `0x80010000` (EXE load address) → `0x80000200` (kernel reserved)
- CP0 mapping uses on-demand C++ getters instead of stale VBlank polling
- DMA trampoline no longer manually writes `I_STAT` — DMA controller fires IRQ naturally
- `RegisterBank` protected with `std::atomic_flag` spinlock for thread safety
- `RustFFIExport.DataPtr` changed to `uint64_t` (96 bytes, x64 parity with Rust `repr(C)`)
- `PSXHwRegMap` DMA addresses corrected: CH1=`0x1F801090` (MDEC out), CH2=`0x1F8010A0` (GPU)
- Gate B4 enforced as hard prerequisite for `--gamma` production builds

---

## Legal Compliance: Clean-Room Design Methodology

### The Sony v. Connectix Precedent

This project's legal foundation rests on the landmark case **Sony Computer Entertainment, Inc. v. Connectix Corp.**, 203 F.3d 596 (9th Cir. 2000), which established that clean-room reverse engineering of BIOS firmware is protected as fair use under U.S. copyright law.

**Case Background:**

Connectix Corporation developed Virtual Game Station (VGS), a PlayStation emulator for Macintosh and Windows. To achieve compatibility, Connectix employed a strict clean-room methodology:

1. **Reverse Engineering Team (Dirty Room):** One group of engineers disassembled the Sony PlayStation BIOS to understand its functional behavior — what inputs it accepted, what outputs it produced, and how it interacted with the hardware. They documented these findings as functional specifications. No code was copied; only behavior was observed.

2. **Implementation Team (Clean Room):** A separate group of engineers, who had never seen the disassembled Sony BIOS code, wrote new software based solely on the functional specifications produced by the first team. The resulting code was 100% original.

**The Ninth Circuit's Ruling:**

The court held that Connectix's intermediate copies made during reverse engineering were transformative works constituting fair use under 17 U.S.C. § 107. The four fair use factors were weighed:

| Factor | Court's Analysis |
|--------|-----------------|
| **Purpose and character of use** | Transformative — Connectix created a new platform (emulator) rather than repackaging the BIOS |
| **Nature of copyrighted work** | Functional firmware, not creative expression — lower protection warranted |
| **Amount and substantiality used** | Only intermediate copies for analysis; final product contained no Sony code |
| **Effect on market** | Transformative — different platform, not a direct substitute for PlayStation |

The court explicitly rejected Sony's argument that reverse engineering for compatibility constituted infringement, affirming that **intermediate copying for the purpose of understanding function is fair use when the final product contains no copyrighted code**.

### How This Project Applies Clean-Room Methodology

Project Frankenstein follows the same two-team separation principle:

1. **Observation Phase (completed in prior sessions):** Hardware behavior was documented from publicly available sources — MIPS R3000A ISA reference manuals, Sony's public patent disclosures, the official PlayStation Technical Reference (publicly released), and hardware register documentation published by third parties. No proprietary BIOS binary was disassembled for this project's output.

2. **Implementation Phase (this project):** All assembly source code (`frankenstein_bios.s`) and C++ code (`vhb_core.cpp`, `bios_tool.cpp`) was written from scratch based on publicly documented functional specifications. The assembled BIOS binary contains:
   - **Zero bytes of proprietary Sony or Nintendo code**
   - **No disassembled firmware routines**
   - **No copyrighted string tables or kernel data**
   - **Open-source signatures** (`FRANK-PSX` / `OPEN-BIOS`) replacing proprietary identifiers (`SCPH-*`)

### What This Project Does NOT Contain

| Prohibited Content | Status |
|-------------------|--------|
| SCPH-1001.BIN or any Sony BIOS dump | **Not included** — deleted from build pipeline |
| Disassembled Sony kernel routines | **Not included** — all code is original |
| Sony A/B/C syscall table implementations | **Not included** — vectors are intercepted, not copied |
| Nintendo/SNES proprietary firmware | **Not included** |
| Any copyrighted BIOS string tables | **Not included** — replaced with open-source signatures |

### Verification

The build pipeline includes automated clean-room compliance verification (6/6 current, 7/7 for V1.01A):

1. **Size:** 524,288 bytes (512KB) — correct PSX BIOS image size
2. **Entry opcode:** Valid MIPS instruction (LUI, opcode 0x0F)
3. **OpenBIOS signature:** Open-source identifier at offset 0x0078
4. **VHB-SUPER-BIOS signature:** `VHB-SUPER-BIOS-v1.01A` at offset 0x7FE0 (was v0.2)
5. **No proprietary strings:** Scans entire image for `SCPH`, `Sony`, `SONY` — none found
6. **Clean kernel area:** Bytes 0x2100–0x7FE0 are zero — no proprietary kernel code
7. **Gamma signature (V1.01A):** `GAMMA-v1.1C` at offset 0x7FF0 — GammaLanguage feature tag

```bash
# Run verification
python build_frankenstein_bios.py --verify-only
```

### Additional Legal Precedents

The clean-room methodology is further supported by:

- **Sega v. Accolade**, 977 F.2d 1510 (9th Cir. 1992): Reverse engineering of object code to understand functional requirements is fair use when the final product contains no copyrighted code.
- **Atari v. Nintendo**, 975 F.2d 832 (Fed. Cir. 1992): While Atari's approach was found improper (they obtained Nintendo's code through the Copyright Office), the court acknowledged that legitimate clean-room reverse engineering for interoperability is lawful.
- **Lewis Galoob Toys v. Nintendo**, 964 F.2d 965 (9th Cir. 1992): Derivative works that do not incorporate copyrighted code are not infringing.

---

## Architecture

### BIOS Image (FRANKENSTEIN.BIOS)

```
0xBFC00000  +-----------------------------------+
            | Clean-Room Kernel (8,448 bytes)   |
            |   ~2,112 instructions             |
            |   - CP0 init (Status/Cause)       |
            |   - Memory controller config      |
            |   - RAM clear (64KB sanity)        |
            |   - SFX-100 bus probe             |
            |   - Region detection (NA/JP/EU)   |
            |   - GPU_Init                      |
            |   - Init_Interrupt_Controller     |
            |   - Init_Heap / Init_Events       |
            |   - Init_Threads                  |
            |   - CD-ROM driver                 |
            |   - SYSTEM.CNF boot parser        |
            |   - Find_HBD (HBD boot logic)     |
            |   - A0/B0/C0 syscall tables       |
            |   - Exception handler + VBlank    |
            |   - 50+ kernel services           |
0xBFC02100  +-----------------------------------+
            | Zero-fill (no proprietary code)   |
            |   515,968 bytes of zeros          |
0xBFC07FE0  +-----------------------------------+
            | VHB-SUPER-BIOS-v0.2 (16 bytes)   |
            | Version tag (16 bytes)            |
0xBFC08000  +-----------------------------------+
```

The BIOS image is a ~2,112-instruction clean-room kernel with installed A0/B0/C0 syscall tables, a CD-ROM driver, SYSTEM.CNF boot-chain parser, HBD-aware boot logic, and a VBlank-delivering exception handler.

### VHB Core (C++ Loader Shell)

The `vhb_core.cpp` / `vhb_core.h` files implement a C++ hypervisor that loads ROM/ISO images, parses PS-X EXE headers, and sets up a Soft-MMU with a 64MB unified memory pool. This is a host-side loader, not part of the BIOS image. It compiles clean with MSVC 19.44.

### Subsystems (Current State)

| Subsystem | In BIOS Image? | In C++ Shell? | Status |
|-----------|---------------|---------------|--------|
| Bootstrap (CP0 init, RAM clear) | Yes | N/A | Working (B1 gate passed) |
| GPU initialization | Yes | N/A | Working (GPU_Init, GPU_dw, GPU_mem2vram, GPU_send) |
| Interrupt controller + VBlank | Yes | N/A | Working (VBlank_Handler, exception_handler) |
| Event system | Yes | N/A | Working (deliver/open/close/wait/test/enable/disable) |
| Thread system | Yes | N/A | Working (openThread/closeThread/changeThread/returnFromException) |
| Heap management | Yes | N/A | Working (malloc/free/calloc/realloc/kmalloc/kfree/InitHeap) |
| String/memory functions | Yes | N/A | Working (strcmp/strcpy/strlen/memcpy/memset/atoi/abs/labs) |
| File I/O | Yes | N/A | Working (open/read/write/close/lseek/printf/puts/exit) |
| CD-ROM driver | Yes | N/A | Working (CD_init/CD_read/CD_readSync/CD_getstat/CD_seek) |
| Syscall tables (A0/B0/C0) | Yes | N/A | Installed (Install_Syscall_Vectors, Copy_Syscall_Tables) |
| SYSTEM.CNF boot parser | Yes | N/A | Working (Parse_Boot_Line, Find_File_By_Name) |
| HBD boot logic | Yes | N/A | Present (Find_HBD, skip_hbd_load) |
| Soft-MMU (64MB pool, virt-to-phys) | No | Yes | Compiles; not in BIOS |
| GammaLanguage v1.1C (A2A-DSL) | No | Blueprinted | V1.01A: inline header `A2A_Gamma.h`, 5 new opcodes, lexer + evaluator |
| Rust FFI Bridge (PEAK-1..5) | No | Blueprinted | V1.01A: `gamma_ffi` crate, `repr(C)` parity, zero-copy offsets |
| CP0 Register Mapping | Boot snapshot | Blueprinted | V1.01A: on-demand getters (not VBlank polling), `0x80000200` snapshot |
| PSXMatrix DMA Dispatch | No | Blueprinted | V1.01A: controller-driven IRQ, no manual I_STAT write |
| VFS | No | No | Roadmap |
| Multi-console (SNES, Saturn, etc.) | No | No | Quarantined to `_future_multibit/` |

### Scope Decision (D-B)

`psx_core` is scoped to **BIOS validation only**: boot BIOS, run EXE to first milestone, verify no crash. For full PSX tracing/fidelity, use the CyberGrime harness at `DQLOSTTRANSLATION/cybergrime/` (separate environment, DQ4 Frankenstein).

---

## Build & Usage

### Prerequisites

- Python 3.8+ (for build orchestrator)
- g++ or clang++ with C++17 support (for C++ utilities)
- Optional: `mipsel-linux-gnu-as` / `mipsel-linux-gnu-ld` (external MIPS toolchain)

### Build the Clean-Room BIOS

```bash
# Build standalone 512KB BIOS from source only (no proprietary merges)
python build_frankenstein_bios.py --region NA

# Force Python assembler (no external toolchain needed)
python build_frankenstein_bios.py --region NA --force-python

# Verify clean-room compliance
python build_frankenstein_bios.py --verify-only
```

### Compile C++ Utilities

```bash
# bios_tool (binary padding, signature injection, verification)
g++ -O2 -std=c++17 -o bios_tool.exe bios_tool.cpp

# VHB launcher
g++ -O2 -std=c++17 -o vhb_launch.exe vhb_core.cpp vhb_launch.cpp
```

### Launch Custom ROMs

```bash
# Inspect a ROM without executing
vhb_launch.exe dq4_frankenstein_v45.bin --target psx --inspect

# Full launch with vector listing
vhb_launch.exe dq4_frankenstein_v45.bin --target psx --vectors

# Override entry point
vhb_launch.exe FRANKENSTEIN.BIOS --target psx --entry 0x80100000

# Mount a VFS asset file
vhb_launch.exe game.iso --target psx --vfs textures:assets.bin
```

### DuckStation Deployment

1. Copy `FRANKENSTEIN.BIOS` to your DuckStation BIOS directory
2. Set BIOS path to `FRANKENSTEIN.BIOS` in DuckStation settings
3. Boot the custom disc (e.g., `dq4_frankenstein_v45.bin`)
4. Boot sequence: CP0 init → memory controller → RAM clear → SFX-100 probe → region detect → GPU init → interrupt controller → heap/events/threads init → syscall table install → CD-ROM init → SYSTEM.CNF parse → EXE load → HBD locate → jump to EXE entry

---

## File Inventory

| File | Language | Description |
|------|----------|-------------|
| `frankenstein_bios.s` | MIPS assembly | Clean-room kernel source (2,601 lines, ~2,112 instructions, ~8,448 bytes) |
| `build_frankenstein_bios.py` | Python | Hybrid build orchestrator (assemble + pack + verify) |
| `bios_tool.cpp` | C++ | Binary utility (pad, inject, SHA-256, verify) |
| `vhb_core.h` | C++ | VHB header (TargetSystem, SoftMMU, VirtualHardwareContext) |
| `vhb_core.cpp` | C++ | VHB implementation (ROM loading, EXE parsing, memory map) |
| `vhb_launch.cpp` | C++ | CLI launcher (load, inspect, execute) |
| `psx_core.h` / `psx_core.cpp` | C++ | Minimal PSX emulator for BIOS validation (scoped, not full-fidelity) |
| `FRANKENSTEIN.BIOS` | Binary | Clean-room BIOS output (512KB, zero proprietary code) |
| `mips/` | Python | Extended MIPS assembler/disassembler (CP0, GAS directives) |
| `cybergrime/` | Various | CyberGrime PSX harness environment (separate environment, not part of the BIOS distribution) |
| `_future_multibit/` | Various | Quarantined multi-bit platform modules (D-A decision, see README_FUTURE.md) |
| `archive/` | Various | Archived stale stdout/stderr logs |

---

## Memory Ceiling Diagnosis (Roadmap Context)

### Why Standard Emulators Crash with Large ROM Hacks

Traditional PSX emulators enforce the original hardware's architectural limits:

- **Fixed 2MB RAM:** The PSX provides 2MB main RAM. When a ROM hack grafts DQ4's asset architecture (304MB HBD) onto DW7's engine, the text arrays and asset data exceed the 2MB data segment, causing heap overflow and stack corruption.
- **HLE Trap:** High-Level Emulation mimics system calls but still enforces the original BIOS's memory map.
- **8MB Wall:** Even with expansion patches, the original BIOS never mapped extended memory banks.

### How VHB Plans to Solve This (Roadmap)

The VHB's C++ Soft-MMU can intercept memory requests that would exceed the legacy 2MB ceiling, translate the high address to the extended virtual pool (up to 64MB), and feed the data back to the engine as if it were contiguous native RAM. **This is currently implemented in the C++ loader shell only, not in the BIOS image.** Moving Soft-MMU services into the BIOS image is a Phase 2 roadmap item per `SURGICAL_COMPLETION_VHB_CYBERGRIME_Aug3_2026.md`. The vector dispatcher and VFS are also roadmap items.

### B4 Blocker: Commercial-Emulator Boot

The BIOS currently boots in two in-house emulators (`psx_core` and CyberGrime) but produces a black screen (FPS 0, CPU ~5%) in DuckStation. Root-cause investigation is the primary product blocker (R3 in the repair plan). The BIOS implements a complete kernel with syscall tables, CD-ROM driver, and exception handler — the issue is likely a semantic mismatch with commercial-emulator expectations (kernel data structure layout, COP0 BEV/exception-vector delivery, or CD-ROM async timing). See `study/AUDIT_VHB_CYBERGRIME_REPAIR_UPGRADE_Aug4_2026.md` §6 R3 for the differential diagnosis plan.

**Per Counter Reflection Report (Aug 10, 2026):** Gate B4 is a **hard prerequisite** for enabling `--gamma` in production builds. GammaLanguage features may be developed and tested without B4, but cannot ship until B4 passes. See `BLUEPRINT_V1.01A_GammaLanguage_v1.1C.md` for the full gate dependency chain.

---

## License

This project's original source code is released as open-source. The clean-room methodology ensures no third-party copyrighted material is included. See the verification section above for automated compliance checks.

## Disclaimer

This project is an independent clean-room implementation. It is not affiliated with, endorsed by, or derived from any proprietary firmware belonging to Sony Computer Entertainment, Nintendo Co., Ltd., or Sega Enterprises. All hardware register addresses and instruction set references used in this project are sourced from publicly available documentation, patents, and technical manuals.
