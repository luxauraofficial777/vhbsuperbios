# Project Frankenstein: Virtual Hypervisor BIOS (VHB)

## Product Overview

The Virtual Hypervisor BIOS (VHB) is a clean-room, multi-console abstraction layer that enables custom ROM hacks and homebrew software to run on emulator platforms without relying on proprietary firmware. It provides a Soft-MMU for dynamic memory translation, a vector dispatcher for intercepting legacy BIOS calls, and a virtual file system for unified asset streaming.

The VHB is part of the Project Frankenstein toolkit, which includes:
- **Clean-Room Super BIOS** — 512KB MIPS R3000A bootstrap, built entirely from original source
- **VHB Core** — C++ hypervisor with Soft-MMU, vector dispatch, and VFS
- **VHB Launcher** — CLI tool to load, inspect, and launch custom ROMs
- **bios\_tool** — C++ binary utility for padding, signature injection, and verification

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

The build pipeline includes automated clean-room compliance verification (6/6 checks):

1. **Size:** 524,288 bytes (512KB) — correct PSX BIOS image size
2. **Entry opcode:** Valid MIPS instruction (LUI or COP0)
3. **FRANK-PSX signature:** Open-source identifier at offset 0x7FE0
4. **OPEN-BIOS tag:** Open-source identifier at offset 0x7FF0
5. **No proprietary strings:** Scans entire image for `SCPH`, `Sony`, `SONY` — none found
6. **Clean kernel area:** All bytes between bootstrap code and signature block are zero — no proprietary kernel code

```bash
# Run verification
python bioshackproject/build_frankenstein_bios.py --verify-only
```

### Additional Legal Precedents

The clean-room methodology is further supported by:

- **Sega v. Accolade**, 977 F.2d 1510 (9th Cir. 1992): Reverse engineering of object code to understand functional requirements is fair use when the final product contains no copyrighted code.
- **Atari v. Nintendo**, 975 F.2d 832 (Fed. Cir. 1992): While Atari's approach was found improper (they obtained Nintendo's code through the Copyright Office), the court acknowledged that legitimate clean-room reverse engineering for interoperability is lawful.
- **Lewis Galoob Toys v. Nintendo**, 964 F.2d 965 (9th Cir. 1992): Derivative works that do not incorporate copyrighted code are not infringing.

---

## Architecture

```
+---------------------------------------------------------------+
|                 FRANKENSTEIN RUNTIME PAYLOAD                  |
|        (DQ4 Assets + DW7 Engine + Expanded Text Arrays)       |
+---------------------------------------------------------------+
                               |
                               v
+---------------------------------------------------------------+
|                    HYPERVISOR BIOS (VHB)                      |
|  * Intercepts memory requests and translates high addresses   |
|  * Expands heap boundaries beyond the legacy 2MB limit        |
|  * Replaces rigid BIOS calls with dynamic allocation vectors  |
+---------------------------------------------------------------+
                               |
              +----------------+----------------+
              |                |                |
              v                v                v
      [ Virtual SNES ]  [ Virtual PSX ]  [ Virtual Saturn ]
```

### Subsystems

| Subsystem | Description |
|-----------|-------------|
| **Soft-MMU** | 64MB unified memory pool with virtual-to-physical address translation. Maps legacy 2MB RAM to extended 8MB+ pool, bypassing hardware memory ceilings. |
| **Vector Dispatcher** | Intercepts legacy BIOS exception vectors (A/B/C syscall tables, NMI, IRQ). Redirects to custom handlers for unconstrained execution. |
| **VFS** | Virtual File System for mounting and streaming asset bundles, bypassing CD-ROM bandwidth bottlenecks. |
| **Target Profiles** | PSX (MIPS R3000A), SNES (65816 + SFX-100), Saturn (Dual SH-2). Each profile defines memory map, entry point, and sector format. |

---

## Build & Usage

### Prerequisites

- Python 3.8+ (for build orchestrator)
- g++ or clang++ with C++17 support (for C++ utilities)
- Optional: `mipsel-linux-gnu-as` / `mipsel-linux-gnu-ld` (external MIPS toolchain)

### Build the Clean-Room BIOS

```bash
# Build standalone 512KB BIOS from source only (no proprietary merges)
python bioshackproject/build_frankenstein_bios.py --region NA

# Force Python assembler (no external toolchain needed)
python bioshackproject/build_frankenstein_bios.py --region NA --force-python

# Verify clean-room compliance
python bioshackproject/build_frankenstein_bios.py --verify-only
```

### Compile C++ Utilities

```bash
# bios_tool (binary padding, signature injection, verification)
g++ -O2 -std=c++17 -o bioshackproject/bios_tool.exe bioshackproject/bios_tool.cpp

# VHB launcher
g++ -O2 -std=c++17 -o bioshackproject/vhb_launch.exe \
    bioshackproject/vhb_core.cpp bioshackproject/vhb_launch.cpp
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
4. Bootstrap sequence: CP0 init -> memory controller -> RAM clear -> SFX-100 probe -> region detection -> jump to `0x80100000`

---

## File Inventory

| File | Language | Description |
|------|----------|-------------|
| `frankenstein_bios.s` | MIPS assembly | Clean-room bootstrap source (106 instructions, 424 bytes) |
| `build_frankenstein_bios.py` | Python | Hybrid build orchestrator (assemble + pack + verify) |
| `bios_tool.cpp` | C++ | Binary utility (pad, inject, SHA-256, verify) |
| `vhb_core.h` | C++ | VHB header (TargetSystem, SoftMMU, VectorDispatcher, VFS) |
| `vhb_core.cpp` | C++ | VHB implementation (ROM loading, EXE parsing, memory map) |
| `vhb_launch.cpp` | C++ | CLI launcher (load, inspect, execute) |
| `FRANKENSTEIN.BIOS` | Binary | Clean-room BIOS output (512KB, zero proprietary code) |

---

## Memory Ceiling Diagnosis

### Why Standard Emulators Crash with Large ROM Hacks

Traditional PSX emulators enforce the original hardware's architectural limits:

- **Fixed 2MB RAM:** The PSX provides 2MB main RAM. When a ROM hack grafts DQ4's asset architecture (304MB HBD) onto DW7's engine, the text arrays and asset data exceed the 2MB data segment, causing heap overflow and stack corruption.
- **HLE Trap:** High-Level Emulation mimics system calls but still enforces the original BIOS's memory map. A syscall expecting a 2MB buffer throws an unhandled exception when it encounters a multi-megabyte text archive.
- **8MB Wall:** Even with expansion patches, the original BIOS never mapped extended memory banks. Instructions pointing to high addresses fall into unmapped void space.

### How VHB Solves This

The VHB's Soft-MMU intercepts memory requests that would exceed the legacy 2MB ceiling, translates the high address to the extended virtual pool (up to 64MB), and feeds the data back to the engine as if it were contiguous native RAM. The vector dispatcher replaces rigid BIOS allocation calls with dynamic allocation vectors, and the VFS bypasses CD-ROM bandwidth limits for on-the-fly asset injection.

---

## License

This project's original source code is released as open-source. The clean-room methodology ensures no third-party copyrighted material is included. See the verification section above for automated compliance checks.

## Disclaimer

This project is an independent clean-room implementation. It is not affiliated with, endorsed by, or derived from any proprietary firmware belonging to Sony Computer Entertainment, Nintendo Co., Ltd., or Sega Enterprises. All hardware register addresses and instruction set references used in this project are sourced from publicly available documentation, patents, and technical manuals.
