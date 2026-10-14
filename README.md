# GBEmu — a Game Boy emulator for the Casio Graph 35+E II

🇬🇧 English · 🇫🇷 [Version française](README.fr.md)

A Game Boy (DMG, monochrome) emulator written in C for the **Casio Graph 35+E II**
(SH-4A processor, 128×64 screen), built with the **fxSDK / gint** toolchain from
the Planète Casio community.

> **Project status: experimental.** An `.g1a` add-in builds, launches on a real
> calculator, runs a 64 KB MBC1 ROM and displays the picture. It is **very
> slow** (speed not yet quantified, see [Benchmark](#2-benchmark)) and has **no
> sound**. This is not a complete emulator.

> **Note:** source-code comments and the calculator's on-screen messages
> (e.g. `GO !`, `FINI`, error screens) are currently in French.

## Contents

0. [Before you start](#0-before-you-start) — legal, requirements, build
1. [Emulation](#1-emulation) — architecture, what works, what is missing
2. [Benchmark](#2-benchmark) — measuring speed, tuning options
3. [Possible errors](#3-possible-errors) — symptom → cause → fix
4. [Tests](#4-tests) — validating the core on PC and on big-endian
5. [Ideas for improvement](#5-ideas-for-improvement)

---

## 0. Before you start

### Legal

**No ROM is provided and none must be published in this repository.** The
`.gitignore` excludes `*.gb`, `src/rom.c` and `include/rom.h` (the latter two
contain the ROM data). Only use ROMs you dumped yourself from cartridges you
own, and check what your country's law allows regarding private copies. The
Blargg test ROMs used to validate the core are freely available testing tools,
downloaded on demand by a script and not included here.

*Game Boy* is a trademark of Nintendo and *Casio* a trademark of Casio. This
project is not affiliated with either company.

### Requirements

- A **Casio Graph 35+E II** (tested on this model only).
- Linux, or **Windows 11 with WSL2 (Ubuntu)**. The fxSDK does not exist for
  native Windows.
- System packages:
  ```sh
  sudo apt install curl git python3 build-essential pkg-config cmake \
      python3-pil libpng-dev libusb-1.0-0-dev libudisks2-dev libsdl2-dev \
      libncurses-dev libarchive-dev libzip-dev libpopt-dev
  ```
- **fxSDK + gint** (≥ 2.9; tested with fxSDK 2.11.0, gint 2.11.0,
  sh-elf-gcc 14.1.0), installed through GiteaPC:
  ```sh
  curl "https://git.planet-casio.com/Lephenixnoir/GiteaPC/raw/branch/master/install.sh" -o /tmp/giteapc-install.sh
  bash /tmp/giteapc-install.sh        # accept the PATH change, then reopen your terminal

  giteapc install Lephenixnoir/fxsdk Lephenixnoir/sh-elf-binutils Lephenixnoir/sh-elf-gcc
  giteapc install Lephenixnoir/OpenLibm Vhex-Kernel-Core/fxlibc
  giteapc install Lephenixnoir/sh-elf-gcc      # 2nd time: expected, it finishes libstdc++
  giteapc install Lephenixnoir/gint
  ```
  Building GCC takes a long time (15–30 min).

### Build

```sh
git clone <url-of-this-repository> gbemu && cd gbemu

# 1. Convert YOUR ROM into a C array (stored in flash, not in RAM).
#    Quote the path if it contains spaces or '&'.
python3 tools/rom2c.py "path/to/your_rom.gb" src/rom.c include/rom.h

# 2. Build
fxsdk build-fx
```

`rom2c.py` prints the title and the cartridge type (MBC), and warns if the type
is not supported. The build produces `GBEmu.g1a` (find it with
`find . -name '*.g1a'`).

> This repository has a structure close to a project created by `fxsdk new`. If
> `fxsdk build-fx` complains about the structure with your fxSDK version, create
> a blank project (`fxsdk new MyProject`), copy `src/` and `include/` into it and
> replace its `CMakeLists.txt` with this one.

### Sending it to the calculator

- **Native Linux**: `fxsdk send GBEmu.g1a` (requires `p7`/`fxlink`).
- **WSL**: `fxsdk send` does not work (no USB access inside WSL). Copy the
  `.g1a` to the Windows side (Explorer → `\\wsl$\Ubuntu\home\<you>\gbemu\…`)
  and transfer it with Casio's official software, with the calculator in
  **LINK → RECV** mode.

### Controls

| Calculator | Game Boy |
|---|---|
| Arrow keys | D-pad |
| SHIFT | A |
| ALPHA | B |
| EXE | START |
| OPTN | SELECT |
| F1 | Toggle **scaled** ⇄ **cropped** display |
| F2 / F3 | Move the window left / right (cropped mode) |
| F5 / F6 | Move the window up / down (cropped mode) |
| EXIT | Quit |

---

## 1. Emulation

### Target constraints

| | Game Boy | Graph 35+E II |
|---|---|---|
| CPU | SM83, 4.19 MHz | SH-4A, **big-endian** |
| Screen | 160×144, 4 shades | 128×64, 1 bit (4 levels simulated by gint's `dgray` engine) |
| Usable RAM | 8 KB + 8 KB VRAM | very small static region; gint's heap is larger |

### Architecture

```
include/gb.h     structures (CPU, MMU, PPU, timer), register accessors
src/cpu.c        SM83 core: x/y/z/p/q field decoding, CB prefix, interrupts
src/mmu.c        memory bus, MBC 0/1/2/3/5, I/O, OAM DMA, joypad, serial port (hook)
src/ppu.c        LCD state machine (modes 0-3) + scanline renderer
src/timer.c      DIV / TIMA / TMA / TAC
src/gb.c         machine assembly and one-frame loop
src/main.c       gint frontend: display, keyboard, compile-time options
tools/rom2c.py   ROM .gb → src/rom.c + include/rom.h
tools/test_host.c, tools/blargg_test.c, tools/run_blargg.sh   tests on PC
```

### Technical decisions that matter

**CPU registers without unions (endianness).** The `AF/BC/DE/HL` pairs are plain
`uint16_t` values; the high/low bytes are extracted with shifts and masks
(`r_a()`, `set_b()`…). An earlier version used `union`s, which silently assume
little-endian: it passed every test on a PC (x86) but **broke emulation on the
calculator** (the SH-4 is big-endian; the target is `sh3eb-elf`). Reproduced and
verified under big-endian QEMU: see [Tests](#4-tests).

**Memory.** `gb_t` (~22 KB) is allocated with `malloc`, never as a static
variable: as a static, linking fails (`region 'ram' overflowed`). The
framebuffer is stored at **2 bits per pixel** (5,760 bytes instead of 23,040).
The ROM is a `const` array in flash.

**160×144 → 128×64 display.** Two modes (F1):
- *scaled*: nearest-neighbour, the whole Game Boy screen is visible, but roughly
  every other line is lost — text is often hard to read;
- *cropped*: native 1:1 pixels in a 128×64 window you move by hand — sharp, but
  partial.

The four Game Boy shades map exactly to the four `dgray` levels (written
directly into the two bit planes). Without `dgray` (`GBEMU_NO_GRAY`), it falls
back to 1-bit ordered dithering (4×4 Bayer).

**PPU.** The background is rendered tile by tile (one VRAM read every 8 pixels,
direct access to `vram[]` without going through the bus). When the LCD is off,
the mode bits of `STAT` are forced to 0, as on real hardware.

### What is implemented

- Complete SM83 instruction set, CB prefix, interrupts, HALT, delayed EI.
- MBCs: none, MBC1, MBC2 (ROM banking), MBC3 (no clock), MBC5.
- Background, window, 8×8 and 8×16 sprites, X/Y flips, background/sprite
  priority, BGP/OBP0/OBP1 palettes, OAM DMA.
- Timer, joypad with interrupt, serial port (captured for the tests).
- Direct start at `0x0100` with post-boot-ROM register values (the boot ROM is
  not emulated).

### Missing or approximate features

- **No sound.**
- No saves: cartridge SRAM is allocated (capped at 32 KB) but never written to
  the calculator's storage.
- MBC3: no real-time clock. MBC2: no built-in RAM.
- No Game Boy Color (DMG mode only; `rom2c.py` warns).
- OAM DMA is instantaneous; memory accesses are not cycle-exact.
- Sprite priority: OAM order, not the "lowest X wins" rule.
- Simplified window line counter.
- `STAT` interrupts without edge detection (may be requested repeatedly).
- Timer: no one-cycle delay when `TIMA` reloads.
- Games tested: **one** (a 64 KB MBC1 ROM). General compatibility is unknown.
  ROMs over 1 MB: untested.

### Validation

Blargg suites run on the core compiled for PC:

| Suite | Result |
|---|---|
| `cpu_instrs` (the 11 individual tests) | ✅ pass (x86 **and** big-endian) |
| `instr_timing` | ✅ pass (x86 **and** big-endian) |
| `mem_timing` / `01-read_timing` | ❌ fails (memory accesses not cycle-exact) |
| other suites (`interrupt_time`, `oam_bugs`, `halt_bug`, …) | not run |

These suites validate the instruction core, **not** the PPU nor compatibility
with real games.

---

## 2. Benchmark

Speed has **not been measured yet**: all that is known is that it is far below
100%. A benchmark mode is provided to quantify it on your calculator, with no
special tooling (just a stopwatch).

### How it works

It runs exactly **300 Game Boy frames** (≈ 5 s of game time at 59.7 frames/s)
with **rendering and display on every frame** (a deliberate worst case),
between a `GO !` screen and a `FINI` screen.

### Procedure

1. In `CMakeLists.txt`, uncomment `set(GBEMU_BENCHMARK ON)`.
2. Rebuild from scratch:
   ```sh
   rm -rf build-fx && fxsdk build-fx
   ```
3. Transfer and launch the add-in. Press **EXE**.
4. Start your stopwatch when **`GO !`** appears, stop it when **`FINI`**
   appears.
5. Compute: **speed = 5 / T × 100 %** (T in seconds). Real time = 5 s.

### Results

To be filled in with your measurements:

| Configuration | T (s) | Speed |
|---|---|---|
| Default (gray, scaled) | | |
| `GBEMU_NO_GRAY` | | |

Also note which game you used (the cost depends on the game) and your
calculator model.

### Tuning options

Edit the top of `CMakeLists.txt`, then `rm -rf build-fx && fxsdk build-fx`:

| Option | Effect |
|---|---|
| `set(GBEMU_FRAMESKIP N)` | Displays 1 frame out of N+1 (default 1). Emulation keeps its pace, only the display is spaced out. Has no effect on the benchmark, which always displays everything. |
| `set(GBEMU_NO_GRAY ON)` | Disables the 4 gray levels: less work per frame, less faithful rendering. |
| `set(GBEMU_BENCHMARK ON)` | Builds the benchmark instead of the game. |

`main.c` also calls `clock_set_speed(CLOCK_SPEED_F3)` to ask gint for a faster
frequency mode; its real effect on this model has not been measured. If the
add-in becomes unstable, remove that call (and the `CLOCK_SPEED_DEFAULT` one on
exit).

---

## 3. Possible errors

All of these were encountered during development.

### Toolchain installation

| Symptom | Cause | Fix |
|---|---|---|
| `curl … install.sh \| bash` installs nothing | URL uses `/src/branch/`: it returns the HTML page, not the script | Use `/raw/branch/` (see §0); download the file, then run it with `bash` |
| `fxsdk: command not found` although `giteapc list` shows fxsdk | `~/.local/bin` is not in `PATH` | `echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.bashrc && source ~/.bashrc` |
| `Package 'libpng16' … not found` | Missing dev package | `sudo apt install libpng-dev` |
| `Package 'ncurses' … not found` | same | `sudo apt install libncurses-dev` |
| Any other `Package 'XXX' not found` | same | `sudo apt install libXXX-dev` |
| `sh-elf-gcc` gets installed twice | Expected | Nothing to do |

### Build

| Symptom | Cause | Fix |
|---|---|---|
| `fatal error: gb.h: No such file or directory` | `include/` not declared, or the template's `CMakeLists.txt` left in place | Use this repository's `CMakeLists.txt` (`target_include_directories(... include)`) |
| Only `main.c` gets compiled | Template `SOURCES` list not replaced | Same |
| `region 'ram' overflowed by N bytes` | Large structure as a static variable (`.bss`) | Allocate on the heap: already done (`malloc(sizeof(gb_t))`). Do not make `gb_t` `static` again |
| `implicit declaration of function 'ppu_set_pixel'` | Old `gb.h` in place (mixed versions) | Replace all files with the same version, then `rm -rf build-fx` |
| `implicit declaration of function 'sprintf'` | Missing `<stdio.h>` (if you add debug code) | `#include <stdio.h>` |
| `redefinition of 'xxx'` / odd error inside `stddef.h` | File hand-edited in several passes, duplicated function or unbalanced braces | Start again from a clean file from the repository |
| `rom.c`/`rom.h` not found | ROM not converted yet | `python3 tools/rom2c.py "your_rom.gb" src/rom.c include/rom.h` |
| `rom2c.py`: `No such file or directory` with a path | Unquoted space or `&` | Put the path in quotes |

### Sending

| Symptom | Cause | Fix |
|---|---|---|
| `p7 is not installed or not available` | `p7` missing | Not blocking for the build; transfer the `.g1a` another way |
| `fxsdk send` fails under WSL | No USB access in WSL2 | Copy the `.g1a` to Windows and use Casio's software |

### At runtime

| Symptom | Likely cause | Fix |
|---|---|---|
| The add-in shows up but won't launch / returns to the menu | Crash at startup | Build a version with progress messages (`dtext` + `getkey()` at each step of `main`) to locate the failing step |
| Blank screen, CPU running (consistent cycle counter) but `LCDC` bit 7 is 0 | **Endianness bug** if you change the CPU registers using `union`s; or a game waiting for missing hardware behaviour | Never read the bytes of a register pair through a `union` (see §1); validate on big-endian (§4) |
| `malloc(gb_t) a echoue` displayed | Not enough heap | Close other add-ins; do not enlarge `gb_t` |
| The game is very slow | Interpreted core on a modest CPU | §2: measure first, then `GBEMU_FRAMESKIP`, `GBEMU_NO_GRAY`; see also §5 |
| Unreadable text | Scaled mode (144 lines → 64) | **F1** for cropped mode, **F2/F3/F5/F6** to move the window |
| Wrong graphics / game goes haywire | Unsupported cartridge type or feature | Read the `rom2c.py` warning, check [the limitations](#missing-or-approximate-features) |
| `rom2c.py` warns "Game Boy Color" | CGB ROM | Only DMG mode is emulated: display will probably be wrong |

---

## 4. Tests

The core (`cpu/mmu/ppu/timer/gb`) does not depend on gint and builds on a PC.

```sh
make test      # CPU unit tests + rendering of a hand-made mini ROM
make blargg    # downloads Blargg's test ROMs and runs cpu_instrs + instr_timing
```

### Checking on a big-endian architecture

This is the test that would have caught the endianness bug. Example with s390x
(big-endian) emulated by QEMU, on Ubuntu:

```sh
sudo apt install gcc-s390x-linux-gnu qemu-user
s390x-linux-gnu-gcc -static -std=c11 -O2 -Iinclude \
    src/cpu.c src/mmu.c src/ppu.c src/timer.c src/gb.c tools/blargg_test.c -o blargg_be
qemu-s390x ./blargg_be "tools/gb-test-roms/cpu_instrs/individual/06-ld r,r.gb" 15
```

The output must end with `[OK]`. **Do this after any change to the CPU core.**

---

## 5. Ideas for improvement

Not implemented, not tested:

- **Only render the lines that are displayed.** In scaled mode only ~64 of the
  144 lines are used by the screen; the PPU draws all of them.
- **Write pixels a whole byte at a time** instead of a read-modify-write for
  each 2-bit pixel.
- **A fast path for `mmu_read`** (page table) to avoid the chain of comparisons
  on every access.
- Sound, SRAM saving, MBC3 clock, exact sprite priority.
- Run more test suites (`mem_timing`, `interrupt_time`, `halt_bug`,
  `oam_bugs`).

## Credits

- [fxSDK / gint](https://git.planet-casio.com/Lephenixnoir/fxsdk) and GiteaPC,
  by Lephenixnoir and the [Planète Casio](https://www.planet-casio.com)
  community.
- Blargg's test ROMs (validation only, not included), fetched through the
  `retrio/gb-test-roms` repository.
- This was coded by claude (I showed him what was displayed on my screen and shared my comments, and he debugged, fixed, and improved the emulator)

## License

The License is AGPL-3.0


