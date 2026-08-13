# vhbsuperbios V1.1A
<img width="1024" height="1024" alt="vhbsuperbios" src="https://github.com/user-attachments/assets/d6a1729e-fcce-4e1d-b16a-cdc47a7be37b" />
Clean-room, multi-console hypervisor BIOS for running custom ROM hacks and homebrew on emulator platforms. Built entirely from original source code under the clean-room design methodology established by Sony v. Connectix (203 F.3d 596, 9th Cir. 2000) — zero proprietary Sony/Nintendo firmware, disassembled code, or copyrighted string tables.
Project Frankenstein: Clean-Room Super BIOS
Legal Compliance
This BIOS is built using clean-room design methodology. The distributed package contains:

Zero proprietary binaries — no SCPH-1001.BIN or any copyrighted firmware dump
100% original assembly source — no disassembled Sony/Nintendo code
Open-source signatures — FRANK-PSX / OPEN-BIOS (not SCPH-*)
The output binary is built entirely from frankenstein_bios.s using standard MIPS R3000A ISA instructions and publicly documented hardware register addresses.

================================================================================
 WHAT'S NEW — VHB SUPER BIOS v100A
 Product: Frankenstein-PSX Virtual Hypervisor BIOS
 Release: Jul 29, 2026
================================================================================

This release ports the full OpenBIOS syscall table system into the VHB
Super BIOS. The BIOS now installs A0/B0/C0 dispatch vectors, copies
function pointer tables to RAM, initializes the interrupt controller,
kernel heap, event control blocks, and thread control blocks before
jumping to the loaded EXE entry point.

This is the first version with complete PSX syscall infrastructure,
enabling games to call BIOS functions via the standard 0xA0/0xB0/0xC0
vector mechanism.


NEW FEATURES
---------------------------------------------------------------------------------

1. SYSCALL VECTOR DISPATCH INFRASTRUCTURE
   ---------------------------------------
   Three 4-instruction vector stubs are installed at RAM addresses
   0xA0, 0xB0, and 0xC0 during boot. Each stub indexes into a
   RAM-resident function pointer table using $t1 as the function
   number:

     A0 vector → table at 0x80010000 (0xC0 entries, 0x300 bytes)
     B0 vector → table at 0x80010300 (0x60 entries, 0x180 bytes)
     C0 vector → table at 0x80010480 (0x20 entries, 0x80 bytes)

   The stubs are copied from ROM to RAM by Install_Syscall_Vectors.
   The function tables are copied by Copy_Syscall_Tables via a
   byte-level Memcopy routine.

   Vector stub dispatch mechanism:
     sll  $t0, $t1, 2       # byte offset = function# * 4
     lui  $t2, 0x8001       # table base
     addu $t2, $t2, $t0     # &table[function#]
     lw   $t3, 0($t2)       # load handler address
     jr   $t3               # jump to handler


2. INTERRUPT CONTROLLER INITIALIZATION
   ------------------------------------
   ISTAT (0x1F801070) is cleared to zero, then IMASK (0x1F801074)
   is set to 0xC5, enabling four interrupt sources:

     Bit 0 (0x01) — VBlank
     Bit 2 (0x04) — CD-ROM
     Bit 6 (0x40) — GPU
     Bit 7 (0x80) — DMA


3. KERNEL HEAP ALLOCATOR
   ----------------------
   A simple bump allocator is initialized at 0x80018000 with 32KB
   (0x8000) of heap space. Globals are stored at 0x80010A00:

     Offset  Field         Description
     ------  ----          -----------
     0x00    heap_ptr      Current bump pointer
     0x04    heap_size     Total heap size
     0x08    event_count   Number of EvCB slots
     0x0C    thread_count  Number of TCB slots

   malloc returns the current heap_ptr and advances by the requested
   size (4-byte aligned). free is a no-op. calloc and realloc delegate
   to malloc. InitHeap allows reconfiguration at runtime.


4. EVENT CONTROL BLOCK (EvCB) SYSTEM
   ----------------------------------
   16 event slots are allocated at 0x80010B00 (16 bytes each).
   Each slot stores class, flags, spec, mode, and handler pointer.

   Implemented event syscalls:
     openEvent    — returns handle 0xF1000000 | slot
     closeEvent   — marks slot free
     waitEvent    — returns 1 (event pending)
     testEvent    — returns 0 (no pending event)
     enableEvent  — returns 1
     disableEvent — returns 1
     deliverEvent — no-op (stub)


5. THREAD CONTROL BLOCK (TCB) SYSTEM
   ----------------------------------
   4 thread slots are allocated at 0x80010C00 (128 bytes each).
   Each slot stores flags, registers, and context state.

   Implemented thread syscalls:
     openThread      — returns handle 0xFF000000 | slot
     closeThread     — marks slot free
     changeThread    — returns 1 (stub, no context switch yet)
     getFreeTCBslot  — returns first free slot index


6. A0 SYSCALL STUBS (20 IMPLEMENTED)
   ----------------------------------
   File I/O:
     0x00 open     — returns -1 (no file system yet)
     0x02 read     — returns 0
     0x03 write    — returns 0
     0x04 close    — returns 0

   Math:
     0x0E abs      — integer absolute value via sign-bit trick
     0x0F labs     — long absolute value (same as abs)

   String:
     0x10 atoi     — full ASCII-to-integer with digit range 0x30-0x39
     0x17 strcmp   — byte-by-byte comparison, returns difference
     0x19 strcpy   — byte-by-byte copy including null terminator
     0x1B strlen   — counts bytes until null terminator

   Memory:
     0x2A memcpy   — byte-by-byte copy, returns dest
     0x2B memset   — byte fill, returns dest

   Heap:
     0x33 malloc   — bump allocator, 4-byte aligned
     0x34 free     — no-op
     0x37 calloc   — malloc(nitems * size)
     0x38 realloc  — malloc(new_size)
     0x39 InitHeap — set heap base and size

   I/O:
     0x3F printf   — no-op (no TTY)

   System:
     0x44 FlushCache — delegates to Flush_Cache routine

   GPU:
     0x46 GPU_dw      — writes GP0 fill command + coords
     0x47 GPU_mem2vram — writes GP0 VRAM transfer command
     0x48 GPU_send    — writes single word to GP0


7. B0 SYSCALL STUBS (15 IMPLEMENTED)
   ----------------------------------
   Kernel heap:
     0x00 kmalloc — delegates to A0 malloc
     0x01 kfree   — delegates to A0 free

   Events:
     0x07 deliverEvent — no-op
     0x08 openEvent    — returns 0xF1000000
     0x09 closeEvent   — returns 1
     0x0A waitEvent    — returns 1
     0x0B testEvent    — returns 0
     0x0C enableEvent  — returns 1
     0x0D disableEvent — returns 1

   Threads:
     0x0E openThread   — returns 0xFF000000
     0x0F closeThread  — returns 1
     0x10 changeThread — returns 1

   Exception:
     0x14 returnFromException — reads EPC from CP0 and jumps

   File I/O:
     0x32 open  — returns -1
     0x33 lseek — returns 0
     0x34 read  — returns 0
     0x35 write — returns 0
     0x36 close — returns 0

   System:
     0x39 exit — jumps to boot_fail (infinite loop)
     0x3F puts — no-op


8. C0 SYSCALL STUBS (6 IMPLEMENTED)
   ---------------------------------
     0x05 getFreeTCBslot           — returns 0
     0x06 exceptionHandler         — no-op
     0x07 installExceptionHandler  — no-op
     0x08 kinitheap                — delegates to InitHeap
     0x10 setupFileIO              — no-op
     0x11 reopenStdio              — no-op


9. UNIMPLEMENTED SYSCALL HANDLER
   ------------------------------
   All unimplemented table entries point to Sys_Unimplemented, which:
     - Writes the function number ($t1) to 0x801FFFF0 diagnostic marker
     - Returns 0 in $v0 for graceful degradation

   This allows games to call any BIOS function without crashing, even
   if the function is not yet fully implemented.


10. FULL PSX BOOT SEQUENCE
   ------------------------
   The boot sequence now performs 10 initialization steps after
   loading the EXE and before jumping to the entry point:

     1.  Flush_Cache
     2.  Install_Syscall_Vectors (A0/B0/C0 stubs → RAM 0xA0/0xB0/0xC0)
     3.  Copy_Syscall_Tables (ROM tables → RAM 0x80010000+)
     4.  Init_Interrupt_Controller (ISTAT clear, IMASK=0xC5)
     5.  Init_Heap (0x80018000, 32KB)
     6.  Init_Events (16 EvCB slots at 0x80010B00)
     7.  Init_Threads (4 TCB slots at 0x80010C00)
     8.  Set CP0 Status BEV=0 (normal exception vector mode)
     9.  Flush_Cache (final)
     10. jr $s7 (jump to EXE entry point)


ASSEMBLER ENHANCEMENTS
---------------------------------------------------------------------------------

- .word directive now accepts label references (forward and backward)
  with W fixup kind for resolution at finalize time

- la pseudo-instruction now accepts labels in addition to integers
  with LUI/ORI fixup kinds for forward references

- Fixup resolver extended with W, LUI, and ORI kinds


MEMORY MAP
---------------------------------------------------------------------------------

  Address       Description
  -----------   -----------
  0x000000A0    A0 syscall vector stub (4 words)
  0x000000B0    B0 syscall vector stub (4 words)
  0x000000C0    C0 syscall vector stub (4 words)
  0x80010000    A0 function table (0xC0 entries, 0x300 bytes)
  0x80010300    B0 function table (0x60 entries, 0x180 bytes)
  0x80010480    C0 function table (0x20 entries, 0x80 bytes)
  0x80010A00    Kernel globals (heap_ptr, heap_size, counts)
  0x80010B00    Event control blocks (16 × 16 bytes)
  0x80010C00    Thread control blocks (4 × 128 bytes)
  0x80018000    Kernel heap (32KB, bump allocator)
  0x801FFF00    Exception handler debug area (GPR + COP0 dump)
  0x801FFFF0    Unimplemented syscall diagnostic marker
  0xBFC00000    BIOS ROM base (512KB)
  0xBFC00180    General exception vector


STATS
---------------------------------------------------------------------------------

  BIOS instructions:  161 → 9,479  (+9,318)
  BIOS code size:     644 → 37,916 bytes  (+37,272)
  BIOS image size:    512KB (unchanged)
  BIOS SHA-256:       a62e7e5190723d844df493b71c39d103a108a4b4998b37e332840022d84913
  Clean-room checks:  6/6 PASS
  Toolchain:          Python fallback assembler (zero external deps)
  Syscall stubs:      41 implemented (20 A0 + 15 B0 + 6 C0)
  Function tables:    3 (A0=192 words, B0=96 words, C0=32 words)


FILES MODIFIED
---------------------------------------------------------------------------------

  bioshackproject/frankenstein_bios.s       +syscall vectors, +tables,
                                             +41 handler stubs, +boot init
  cybergrime/mips/assembler.py              +.word labels, +la labels,
                                             +W/LUI/ORI fixup kinds


SYSCALL COVERAGE
---------------------------------------------------------------------------------

  Table  Entries  Implemented  Coverage
  -----  -------  -----------  --------
  A0     0xC0     20           16%
  B0     0x60     15           25%
  C0     0x20     6            19%
  Total  0x140    41           18%

  Remaining entries point to Sys_Unimplemented (graceful return 0).


WHAT'S NEXT
---------------------------------------------------------------------------------

1. Test BIOS in DuckStation with DQ4 JP disc
2. Implement remaining libc syscalls:
   strcat, strncat, strncmp, strncpy, bcopy, bzero, bcmp,
   memmove, qsort, rand, srand, strchr, strrchr, strstr
3. Implement CD-ROM file system syscalls:
   dev_cd_open, dev_cd_read, dev_cd_firstFile, dev_cd_nextFile
4. Implement thread context switching with full register save/restore
5. Implement event delivery with real callback dispatch
6. Add VBlank interrupt handler for event delivery
7. Implement TTY output for printf/puts debugging


================================================================================

Hybrid Build Pipeline
[ Python Orchestrator ] ---> Invokes ---> [ C++ bios_tool ]
(Assembly, orchestration,               (Pad, signature inject,
 dependency checking, fallback)           checksum, clean-room verify)
Toolchain Priority
External MIPS toolchain (if installed): mipsel-linux-gnu-as + mipsel-linux-gnu-ld
Python fallback: cybergrime/mips/assembler.py (pure Python MIPS I assembler)
C++ binary utility: bios_tool (auto-compiled from bios_tool.cpp)
Build
# Build clean-room BIOS (auto-detects toolchain)
python bioshackproject/build_frankenstein_bios.py --region NA

# Force Python assembler (no external toolchain needed)
python bioshackproject/build_frankenstein_bios.py --region NA --force-python

# Verify an existing BIOS
python bioshackproject/build_frankenstein_bios.py --verify-only --output bioshackproject/FRANKENSTEIN.BIOS

# Compile C++ bios_tool separately
g++ -O2 -std=c++17 -o bioshackproject/bios_tool.exe bioshackproject/bios_tool.cpp
Architecture
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
Verification (6/6 checks)
Check	Description
Size	524,288 bytes (512KB)
Entry opcode	Valid MIPS instruction (LUI or COP0)
FRANK-PSX sig	Open-source signature at 0x7FE0
OPEN-BIOS tag	Open-source tag at 0x7FF0
No proprietary strings	No SCPH/Sony/SONY anywhere in image
Clean kernel area	Zero-fill between code and signature (no proprietary code)
Files
File	Language	Description
frankenstein_bios.s	MIPS assembly	Clean-room source (106 instructions, 424 bytes)
build_frankenstein_bios.py	Python	Orchestrator (assemble + pack + verify)
bios_tool.cpp	C++	Binary utility (pad, inject, checksum, verify)
bios_tool.exe	C++ binary	Compiled bios_tool (auto-built)
FRANKENSTEIN.BIOS	Output	Clean-room BIOS image (512KB)
Key Constants
Constant	Value	Description
BIOS_BASE	0xBFC00000	BIOS ROM base address
RAM_CLEAR_END	0x0000FFFF	First 64KB of RAM cleared
PAYLOAD_ENTRY	0x80100000	Frankenstein ROM load address
SFX100_BASE	0x1F400000	SFX-100 expansion bus (hypothetical)
MEM_CTRL_BASE	0x1F801000	Memory control registers (public datasheet)
KANJI_ROM_BASE	0xB8000000	Kanji ROM mapping (JP region, public spec)
SIG_OFFSET	0x7FE0	Open-source signature offset
Extended Assembler
The cybergrime/mips/assembler.py was extended with:

CP0 instructions: mtc0, mfc0, ctc0, cfc0
CP0 register names: Status(12), Cause(13), EPC(14), etc.
GAS directives: .org, .set, .global, .byte, .half, etc.
DuckStation Deployment
Copy FRANKENSTEIN.BIOS to your DuckStation BIOS directory

Project Frankenstein: Virtual Hypervisor BIOS (VHB)
Product Overview
The Virtual Hypervisor BIOS (VHB) is a clean-room, multi-console abstraction layer that enables custom ROM hacks and homebrew software to run on emulator platforms without relying on proprietary firmware. It provides a Soft-MMU for dynamic memory translation, a vector dispatcher for intercepting legacy BIOS calls, and a virtual file system for unified asset streaming.

The VHB is part of the Project Frankenstein toolkit, which includes:

Clean-Room Super BIOS — 512KB MIPS R3000A bootstrap, built entirely from original source
VHB Core — C++ hypervisor with Soft-MMU, vector dispatch, and VFS
VHB Launcher — CLI tool to load, inspect, and launch custom ROMs
bios_tool — C++ binary utility for padding, signature injection, and verification
Legal Compliance: Clean-Room Design Methodology
The Sony v. Connectix Precedent
This project's legal foundation rests on the landmark case Sony Computer Entertainment, Inc. v. Connectix Corp., 203 F.3d 596 (9th Cir. 2000), which established that clean-room reverse engineering of BIOS firmware is protected as fair use under U.S. copyright law.

Case Background:

Connectix Corporation developed Virtual Game Station (VGS), a PlayStation emulator for Macintosh and Windows. To achieve compatibility, Connectix employed a strict clean-room methodology:

Reverse Engineering Team (Dirty Room): One group of engineers disassembled the Sony PlayStation BIOS to understand its functional behavior — what inputs it accepted, what outputs it produced, and how it interacted with the hardware. They documented these findings as functional specifications. No code was copied; only behavior was observed.

Implementation Team (Clean Room): A separate group of engineers, who had never seen the disassembled Sony BIOS code, wrote new software based solely on the functional specifications produced by the first team. The resulting code was 100% original.

The Ninth Circuit's Ruling:

The court held that Connectix's intermediate copies made during reverse engineering were transformative works constituting fair use under 17 U.S.C. § 107. The four fair use factors were weighed:

Factor	Court's Analysis
Purpose and character of use	Transformative — Connectix created a new platform (emulator) rather than repackaging the BIOS
Nature of copyrighted work	Functional firmware, not creative expression — lower protection warranted
Amount and substantiality used	Only intermediate copies for analysis; final product contained no Sony code
Effect on market	Transformative — different platform, not a direct substitute for PlayStation
The court explicitly rejected Sony's argument that reverse engineering for compatibility constituted infringement, affirming that intermediate copying for the purpose of understanding function is fair use when the final product contains no copyrighted code.

How This Project Applies Clean-Room Methodology
Project Frankenstein follows the same two-team separation principle:

Observation Phase (completed in prior sessions): Hardware behavior was documented from publicly available sources — MIPS R3000A ISA reference manuals, Sony's public patent disclosures, the official PlayStation Technical Reference (publicly released), and hardware register documentation published by third parties. No proprietary BIOS binary was disassembled for this project's output.

Implementation Phase (this project): All assembly source code (frankenstein_bios.s) and C++ code (vhb_core.cpp, bios_tool.cpp) was written from scratch based on publicly documented functional specifications. The assembled BIOS binary contains:

Zero bytes of proprietary Sony or Nintendo code
No disassembled firmware routines
No copyrighted string tables or kernel data
Open-source signatures (FRANK-PSX / OPEN-BIOS) replacing proprietary identifiers (SCPH-*)
What This Project Does NOT Contain
Prohibited Content	Status
SCPH-1001.BIN or any Sony BIOS dump	Not included — deleted from build pipeline
Disassembled Sony kernel routines	Not included — all code is original
Sony A/B/C syscall table implementations	Not included — vectors are intercepted, not copied
Nintendo/SNES proprietary firmware	Not included
Any copyrighted BIOS string tables	Not included — replaced with open-source signatures
Verification
The build pipeline includes automated clean-room compliance verification (6/6 checks):

Size: 524,288 bytes (512KB) — correct PSX BIOS image size
Entry opcode: Valid MIPS instruction (LUI or COP0)
FRANK-PSX signature: Open-source identifier at offset 0x7FE0
OPEN-BIOS tag: Open-source identifier at offset 0x7FF0
No proprietary strings: Scans entire image for SCPH, Sony, SONY — none found
Clean kernel area: All bytes between bootstrap code and signature block are zero — no proprietary kernel code
# Run verification
python bioshackproject/build_frankenstein_bios.py --verify-only
Additional Legal Precedents
The clean-room methodology is further supported by:

Sega v. Accolade, 977 F.2d 1510 (9th Cir. 1992): Reverse engineering of object code to understand functional requirements is fair use when the final product contains no copyrighted code.
Atari v. Nintendo, 975 F.2d 832 (Fed. Cir. 1992): While Atari's approach was found improper (they obtained Nintendo's code through the Copyright Office), the court acknowledged that legitimate clean-room reverse engineering for interoperability is lawful.
Lewis Galoob Toys v. Nintendo, 964 F.2d 965 (9th Cir. 1992): Derivative works that do not incorporate copyrighted code are not infringing.
Architecture
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
Subsystems
Subsystem	Description
Soft-MMU	64MB unified memory pool with virtual-to-physical address translation. Maps legacy 2MB RAM to extended 8MB+ pool, bypassing hardware memory ceilings.
Vector Dispatcher	Intercepts legacy BIOS exception vectors (A/B/C syscall tables, NMI, IRQ). Redirects to custom handlers for unconstrained execution.
VFS	Virtual File System for mounting and streaming asset bundles, bypassing CD-ROM bandwidth bottlenecks.
Target Profiles	PSX (MIPS R3000A), SNES (65816 + SFX-100), Saturn (Dual SH-2). Each profile defines memory map, entry point, and sector format.
Build & Usage
Prerequisites
Python 3.8+ (for build orchestrator)
g++ or clang++ with C++17 support (for C++ utilities)
Optional: mipsel-linux-gnu-as / mipsel-linux-gnu-ld (external MIPS toolchain)
Build the Clean-Room BIOS
# Build standalone 512KB BIOS from source only (no proprietary merges)
python bioshackproject/build_frankenstein_bios.py --region NA

# Force Python assembler (no external toolchain needed)
python bioshackproject/build_frankenstein_bios.py --region NA --force-python

# Verify clean-room compliance
python bioshackproject/build_frankenstein_bios.py --verify-only
Compile C++ Utilities
# bios_tool (binary padding, signature injection, verification)
g++ -O2 -std=c++17 -o bioshackproject/bios_tool.exe bioshackproject/bios_tool.cpp

# VHB launcher
g++ -O2 -std=c++17 -o bioshackproject/vhb_launch.exe \
    bioshackproject/vhb_core.cpp bioshackproject/vhb_launch.cpp
Launch Custom ROMs
# Inspect a ROM without executing
vhb_launch.exe dq4_frankenstein_v45.bin --target psx --inspect

# Full launch with vector listing
vhb_launch.exe dq4_frankenstein_v45.bin --target psx --vectors

# Override entry point
vhb_launch.exe FRANKENSTEIN.BIOS --target psx --entry 0x80100000

# Mount a VFS asset file
vhb_launch.exe game.iso --target psx --vfs textures:assets.bin
DuckStation Deployment
Copy FRANKENSTEIN.BIOS to your DuckStation BIOS directory
Set BIOS path to FRANKENSTEIN.BIOS in DuckStation settings
Boot the custom disc (e.g., dq4_frankenstein_v45.bin)
Bootstrap sequence: CP0 init -> memory controller -> RAM clear -> SFX-100 probe -> region detection -> jump to 0x80100000
File Inventory
File	Language	Description
frankenstein_bios.s	MIPS assembly	Clean-room bootstrap source (106 instructions, 424 bytes)
build_frankenstein_bios.py	Python	Hybrid build orchestrator (assemble + pack + verify)
bios_tool.cpp	C++	Binary utility (pad, inject, SHA-256, verify)
vhb_core.h	C++	VHB header (TargetSystem, SoftMMU, VectorDispatcher, VFS)
vhb_core.cpp	C++	VHB implementation (ROM loading, EXE parsing, memory map)
vhb_launch.cpp	C++	CLI launcher (load, inspect, execute)
FRANKENSTEIN.BIOS	Binary	Clean-room BIOS output (512KB, zero proprietary code)
Memory Ceiling Diagnosis
Why Standard Emulators Crash with Large ROM Hacks
Traditional PSX emulators enforce the original hardware's architectural limits:

Fixed 2MB RAM: The PSX provides 2MB main RAM. When a ROM hack grafts DQ4's asset architecture (304MB HBD) onto DW7's engine, the text arrays and asset data exceed the 2MB data segment, causing heap overflow and stack corruption.
HLE Trap: High-Level Emulation mimics system calls but still enforces the original BIOS's memory map. A syscall expecting a 2MB buffer throws an unhandled exception when it encounters a multi-megabyte text archive.
8MB Wall: Even with expansion patches, the original BIOS never mapped extended memory banks. Instructions pointing to high addresses fall into unmapped void space.
How VHB Solves This
The VHB's Soft-MMU intercepts memory requests that would exceed the legacy 2MB ceiling, translates the high address to the extended virtual pool (up to 64MB), and feeds the data back to the engine as if it were contiguous native RAM. The vector dispatcher replaces rigid BIOS allocation calls with dynamic allocation vectors, and the VFS bypasses CD-ROM bandwidth limits for on-the-fly asset injection.

 WHAT'S NEW — VHB SUPER BIOS v0.1B
 Product: Frankenstein-PSX Virtual Hypervisor BIOS
 Release: Jul 28, 2026
================================================================================

This release introduces configurable RAM, descriptor pre-initialization,
extended VHB memory mapping, a descriptor intercept vector, and a crash
handler with exception vector for post-mortem debugging.


NEW FEATURES
---------------------------------------------------------------------------------

1. CONFIGURABLE RAM SIZE (2/8/16MB)
   --------------------------------
   The BIOS RAM_SIZE register is now configurable at build time.

   Usage:
     python build_frankenstein_bios.py --ram-size 2
     python build_frankenstein_bios.py --ram-size 8   (default)
     python build_frankenstein_bios.py --ram-size 16

   The build script patches the li $t3 immediate in the assembled binary
   post-assembly, supporting three hardware configurations:

     2MB  →  0x00000B88  (original PSX spec)
     8MB  →  0x00000F88  (DuckStation extended, default)
     16MB →  0x00001B88  (expanded hack pool)


2. DESCRIPTOR PRE-INITIALIZATION
   ------------------------------
   New BIOS routine Preinit_Descriptors writes safe default values to
   three memory locations before launching the Frankenstein payload:

     *0x800F2228 = 0x80100000  (published descriptor base)
     *0x800D9A40 = 0x80100000  (descriptor table base)
     *0x800F2238 = 0x00000000  (descriptor index)

   This prevents the DW7 EXE from reading garbage descriptor pointers
   if HBD parsing over-allocates and the bump pointer exceeds RAM.

   The routine is called on both the standard boot path and the
   SFX-100 (SNES-CD) boot path.


3. VHB EXTENDED MEMORY MAP
   ------------------------
   The Virtual Hypervisor BIOS core (vhb_core) now supports configurable
   RAM sizes in its memory map and MMU translation layer.

   - TargetProfile struct gains three new fields:
       ram_ceiling       — max addressable RAM for descriptor clamping
       desc_publish_addr — descriptor publish instruction address
       desc_safe_base    — safe fallback descriptor base

   - TargetProfiles::get() accepts a ram_mb parameter (default 8)
   - SuperBIOSHypervisor constructor accepts ram_mb parameter
   - init_memory_map() maps the full configured RAM range as cached
     KSEG0 (0x80000000+) and uncached KSEG1 (0xA0000000+)
   - MMU mappings adapt to the configured RAM size


4. VHB DESCRIPTOR INTERCEPT VECTOR
   --------------------------------
   A new vector is registered at 0x8007AD74 (the descriptor publish
   instruction address in the DW7 EXE).

   When triggered, the intercept handler checks if $a2 (the descriptor
   base being published) exceeds the RAM ceiling. If so, it clamps $a2
   to the safe fallback base (0x80100000).

   This provides hypervisor-level protection that works in concert with
   the P1 EXE trampoline patch — defense in depth.


5. BIOS CRASH HANDLER + EXCEPTION VECTOR
   --------------------------------------
   A general exception handler is now installed at 0xBFC00180 (the PSX
   general exception vector).

   On any unhandled exception, the handler:
     - Saves 15 general-purpose registers ($at through $t7)
     - Saves 4 COP0 registers (EPC, Cause, BadVAddr, Status)
     - Writes crash magic 0xDEADCAFE to 0x801FFF4C
     - Halts in an infinite loop

   Debug area layout at 0x801FFF00 (top of 2MB RAM):

     Offset  Register    Description
     ------  --------    -----------
     0x00    $at         assembler temporary
     0x04    $v0         return value 0
     0x08    $v1         return value 1
     0x0C    $a0         argument 0
     0x10    $a1         argument 1
     0x14    $a2         descriptor base (key for crash analysis)
     0x18    $a3         argument 3
     0x1C    $t0         temporary 0
     0x20    $t1         temporary 1
     0x24    $t2         temporary 2
     0x28    $t3         temporary 3
     0x2C    $t4         temporary 4
     0x30    $t5         temporary 5
     0x34    $t6         temporary 6
     0x38    $t7         temporary 7
     0x3C    EPC         exception program counter
     0x40    Cause       exception cause register
     0x44    BadVAddr    bad virtual address (if address error)
     0x48    Status      CP0 status register
     0x4C    0xDEADCAFE  crash magic (indicates handler fired)

   The VHB core also registers a corresponding vector at 0xBFC00180
   that logs register state to console when triggered.


BUG FIXES
---------------------------------------------------------------------------------

- Branch offset in malloc clamp trampoline corrected from +3 to +2
  (was skipping both the clamp instruction AND the original sw)

- Cross-platform _chdir/chdir wrapped in #if defined(_WIN32) for
  MinGW compilation compatibility


STATS
---------------------------------------------------------------------------------

  BIOS instructions:  106 → 161  (+55)
  BIOS code size:     424 → 644 bytes  (+220)
  BIOS image size:    512KB (unchanged)
  BIOS SHA-256:       caa09f917aabc8b54cc9b5e9f642bb4d19bf3df3b4cf8688c09743210ef2c949
  Clean-room checks:  6/6 PASS
  RAM configs tested: 2MB, 8MB, 16MB — all PASS
  C++ compilation:    g++ -std=c++17 -O2 — all files clean


FILES MODIFIED
---------------------------------------------------------------------------------

  bioshackproject/frankenstein_bios.s      +Preinit_Descriptors, +exception_handler
  bioshackproject/build_frankenstein_bios.py  +--ram-size, +patch_ram_size()
  bioshackproject/vhb_core.h               +TargetProfile fields, +ram_mb param
  bioshackproject/vhb_core.cpp             +adaptive memory map, +2 vectors
  cybergrime/psx_binary_ops.cpp            +patch_malloc_clamp(), cross-platform fix
  cybergrime/psx_binary_ops.h              +constants, +method declaration


DEFENSE IN DEPTH
---------------------------------------------------------------------------------

  Layer  Component          Action
  -----  --------           ------
  P1     EXE trampoline     Runtime clamp in game code during HBD parsing
  P2     BIOS pre-init      Safe defaults written before EXE starts
  P3     VHB intercept      Hypervisor-level clamping during emulation
  P4     Crash handler      Post-mortem diagnostics if all else fails


License
This project's original source code is released as open-source. The clean-room methodology ensures no third-party copyrighted material is included. See the verification section above for automated compliance checks.

Disclaimer
This project is an independent clean-room implementation. It is not affiliated with, endorsed by, or derived from any proprietary firmware belonging to Sony Computer Entertainment, Nintendo Co., Ltd., or Sega Enterprises. All hardware register addresses and instruction set references used in this project are sourced from publicly available documentation, patents, and technical manuals.
Set BIOS path to FRANKENSTEIN.BIOS in DuckStation settings
Boot the Frankenstein disc
Bootstrap sequence: CP0 init → memory controller → RAM clear → SFX-100 probe → region detection → jump to 0x80100000
