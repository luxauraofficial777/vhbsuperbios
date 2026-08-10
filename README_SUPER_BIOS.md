# Project Frankenstein: VHB Super BIOS V1.01A

## Legal Compliance

This BIOS is built using **clean-room design methodology**. The distributed package contains:
- **Zero proprietary binaries** — no SCPH-1001.BIN or any copyrighted firmware dump
- **100% original assembly source** — no disassembled Sony/Nintendo code
- **Open-source signatures** — `VHB-SUPER-BIOS-v1.01A` + `OpenBIOS` tag (not `SCPH-*`)
- **GammaLanguage v1.1C** — `GAMMA-v1.1C` tag at offset 0x7FF0 (V1.01A)

The output binary is built entirely from `frankenstein_bios.s` (2,601 lines) using standard MIPS R3000A ISA instructions and publicly documented hardware register addresses.

## Hybrid Build Pipeline

```
[ Python Orchestrator ] ---> Invokes ---> [ C++ bios_tool ]
(Assembly, orchestration,               (Pad, signature inject,
 dependency checking, fallback)           checksum, clean-room verify)
```

### Toolchain Priority

1. **External MIPS toolchain** (if installed): `mipsel-linux-gnu-as` + `mipsel-linux-gnu-ld`
2. **Python fallback**: `cybergrime/mips/assembler.py` (pure Python MIPS I assembler)
3. **C++ binary utility**: `bios_tool` (auto-compiled from `bios_tool.cpp`)

### Build

```bash
# Build clean-room BIOS (auto-detects toolchain)
python build_frankenstein_bios.py --region NA

# Force Python assembler (no external toolchain needed)
python build_frankenstein_bios.py --region NA --force-python

# Verify an existing BIOS
python build_frankenstein_bios.py --verify-only --output FRANKENSTEIN.BIOS

# Compile C++ bios_tool separately
g++ -O2 -std=c++17 -o bios_tool.exe bios_tool.cpp
```

## Architecture

```
0xBFC00000  +-----------------------------------+
            | Clean-Room Kernel (8,448 bytes)   |
            |   ~2,112 instructions             |
            |   - CP0 init + hardware init      |
            |   - GPU_Init                      |
            |   - Interrupt controller + VBlank |
            |   - Heap/Events/Threads init      |
            |   - CD-ROM driver                 |
            |   - SYSTEM.CNF boot parser        |
            |   - Find_HBD (HBD boot logic)     |
            |   - A0/B0/C0 syscall tables       |
            |   - 50+ kernel services           |
            |   - Exception handler + VBlank    |
0xBFC02100  +-----------------------------------+
            | Zero-fill (no proprietary code)   |
            |   515,968 bytes of zeros          |
0xBFC07FE0  +-----------------------------------+
            | VHB-SUPER-BIOS-v0.2 (16 bytes)   |
            | Version tag (16 bytes)            |
0xBFC08000  +-----------------------------------+
```

## Verification (6/6 current → 7/7 for V1.01A)

| Check | Description |
|-------|-------------|
| Size | 524,288 bytes (512KB) |
| Entry opcode | Valid MIPS instruction (LUI, opcode 0x0F) |
| OpenBIOS sig | Open-source identifier at offset 0x0078 |
| VHB-SUPER-BIOS sig | `VHB-SUPER-BIOS-v1.01A` at offset 0x7FE0 (was v0.2) |
| No proprietary strings | No SCPH/Sony/SONY anywhere in image |
| Clean kernel area | Bytes 0x2100–0x7FE0 are zero (no proprietary code) |
| Gamma sig (V1.01A) | `GAMMA-v1.1C` at offset 0x7FF0 — GammaLanguage feature tag |

## Files

| File | Language | Description |
|------|----------|-------------|
| `frankenstein_bios.s` | MIPS assembly | Clean-room kernel source (2,601 lines, ~2,112 instructions, ~8,448 bytes) |
| `build_frankenstein_bios.py` | Python | Orchestrator (assemble + pack + verify) |
| `bios_tool.cpp` | C++ | Binary utility (pad, inject, checksum, verify) |
| `bios_tool.exe` | C++ binary | Compiled bios_tool (auto-built) |
| `FRANKENSTEIN.BIOS` | Output | Clean-room BIOS image (512KB) |

## Key Constants

| Constant | Value | Description |
|----------|-------|-------------|
| BIOS_BASE | `0xBFC00000` | BIOS ROM base address |
| RAM_CLEAR_END | `0x0000FFFF` | First 64KB of RAM cleared |
| PAYLOAD_ENTRY | `0x80100000` | Frankenstein ROM load address |
| SFX100_BASE | `0x1F400000` | SFX-100 expansion bus (hypothetical) |
| MEM_CTRL_BASE | `0x1F801000` | Memory control registers (public datasheet) |
| KANJI_ROM_BASE | `0xB8000000` | Kanji ROM mapping (JP region, public spec) |
| SIG_OFFSET | `0x7FE0` | Open-source signature offset |
| GAMMA_SIG_OFFSET | `0x7FF0` | GammaLanguage v1.1C signature offset (V1.01A) |
| CP0_SNAPSHOT_ADDR | `0x80000200` | CP0 snapshot in kernel reserved RAM (NOT 0x80010000) |
| GAMMA_FLAG_ADDR | `0x80000220` | Gamma enabled flag byte |
| PSX_HWREG_SHADOW | `0x80000221` | PSX hardware register shadow (40 bytes) |

## Extended Assembler

The `cybergrime/mips/assembler.py` was extended with:
- **CP0 instructions**: `mtc0`, `mfc0`, `ctc0`, `cfc0`
- **CP0 register names**: Status(12), Cause(13), EPC(14), etc.
- **GAS directives**: `.org`, `.set`, `.global`, `.byte`, `.half`, etc.

## DuckStation Deployment

1. Copy `FRANKENSTEIN.BIOS` to your DuckStation BIOS directory
2. Set BIOS path to `FRANKENSTEIN.BIOS` in DuckStation settings
3. Boot the Frankenstein disc
4. Boot sequence: CP0 init → memory controller → RAM clear → SFX-100 probe → region detect → GPU init → interrupt controller → heap/events/threads init → syscall table install → CD-ROM init → SYSTEM.CNF parse → EXE load → HBD locate → jump to EXE entry

## GammaLanguage v1.1C Integration (V1.01A Blueprint)

The V1.01A upgrade adds GammaLanguage v1.1C support across all VHB layers:

- **5 new opcodes:** `RUST_OFFSET_LOAD`, `RUST_FFI_CALL`, `ALIGN_BOUND`, `CP0_REG_MAP`, `PSX_DMA_DISPATCH`
- **Rust FFI Bridge:** `gamma_ffi` crate with `repr(C)` parity types, PEAK-1..5 dispatch
- **CP0 Mapping:** On-demand getters from live hypervisor state (not VBlank polling)
- **DMA Dispatch:** Controller-driven IRQ (no manual `I_STAT` write)
- **Thread Safety:** `RegisterBank` protected with `std::atomic_flag` spinlock
- **Gate B4 Prerequisite:** `--gamma` production builds require DuckStation boot pass

See `BLUEPRINT_V1.01A_GammaLanguage_v1.1C.md` for the full implementation plan.
