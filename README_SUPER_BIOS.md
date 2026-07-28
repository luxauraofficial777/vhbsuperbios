# Project Frankenstein: Clean-Room Super BIOS

## Legal Compliance

This BIOS is built using **clean-room design methodology**. The distributed package contains:
- **Zero proprietary binaries** — no SCPH-1001.BIN or any copyrighted firmware dump
- **100% original assembly source** — no disassembled Sony/Nintendo code
- **Open-source signatures** — `FRANK-PSX` / `OPEN-BIOS` (not `SCPH-*`)

The output binary is built entirely from `frankenstein_bios.s` using standard MIPS R3000A ISA instructions and publicly documented hardware register addresses.

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
python bioshackproject/build_frankenstein_bios.py --region NA

# Force Python assembler (no external toolchain needed)
python bioshackproject/build_frankenstein_bios.py --region NA --force-python

# Verify an existing BIOS
python bioshackproject/build_frankenstein_bios.py --verify-only --output bioshackproject/FRANKENSTEIN.BIOS

# Compile C++ bios_tool separately
g++ -O2 -std=c++17 -o bioshackproject/bios_tool.exe bioshackproject/bios_tool.cpp
```

## Architecture

```
0xBFC00000  +-----------------------------------+
            | Clean-Room Bootstrap (424 bytes)  |
            |   - CP0 init (Status/Cause)       |
            |   - Memory controller config      |
            |   - RAM clear (64KB sanity)        |
            |   - SFX-100 bus probe             |
            |   - Region detection (NA/JP/EU)   |
            |   - Jump to 0x80100000            |
0xBFC001A4  +-----------------------------------+
            | Zero-fill (no proprietary code)   |
            |   523,864 bytes of zeros          |
0xBFC07FE0  +-----------------------------------+
            | FRANK-PSX signature (16 bytes)   |
            | OPEN-BIOS tag (16 bytes)          |
0xBFC08000  +-----------------------------------+
```

## Verification (6/6 checks)

| Check | Description |
|-------|-------------|
| Size | 524,288 bytes (512KB) |
| Entry opcode | Valid MIPS instruction (LUI or COP0) |
| FRANK-PSX sig | Open-source signature at 0x7FE0 |
| OPEN-BIOS tag | Open-source tag at 0x7FF0 |
| No proprietary strings | No SCPH/Sony/SONY anywhere in image |
| Clean kernel area | Zero-fill between code and signature (no proprietary code) |

## Files

| File | Language | Description |
|------|----------|-------------|
| `frankenstein_bios.s` | MIPS assembly | Clean-room source (106 instructions, 424 bytes) |
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

## Extended Assembler

The `cybergrime/mips/assembler.py` was extended with:
- **CP0 instructions**: `mtc0`, `mfc0`, `ctc0`, `cfc0`
- **CP0 register names**: Status(12), Cause(13), EPC(14), etc.
- **GAS directives**: `.org`, `.set`, `.global`, `.byte`, `.half`, etc.

## DuckStation Deployment

1. Copy `FRANKENSTEIN.BIOS` to your DuckStation BIOS directory
2. Set BIOS path to `FRANKENSTEIN.BIOS` in DuckStation settings
3. Boot the Frankenstein disc
4. Bootstrap sequence: CP0 init → memory controller → RAM clear → SFX-100 probe → region detection → jump to `0x80100000`
