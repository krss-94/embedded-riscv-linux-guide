<div align="center">

# Embedded RISC-V Linux — Step by Step

**Bring up a Linux system on RISC-V from nothing: cross toolchain → kernel → rootfs → firmware → bootloader → boot.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Made with](https://img.shields.io/badge/stack-HTML%2FCSS%2FJS-1f6feb)](index.html)
[![Kernel](https://img.shields.io/badge/kernel-6.1.177-informational)](#)
[![U--Boot](https://img.shields.io/badge/U--Boot-2024.10-informational)](#)
[![No dependencies](https://img.shields.io/badge/dependencies-zero-success)](#)

**[▶ View the live guide](https://<your-username>.github.io/<repo-name>/)**

</div>

---

## Why this exists

Most "embedded Linux" tutorials either hand you a pre-built image, or a wall of shell
commands with no context. Neither one teaches you what's actually happening at each
layer of the stack. This guide is the opposite: every command is there, but grouped
into the same boot chain a real embedded board goes through, so you understand *why*
each piece exists before you run it.

## Why RISC-V?

RISC-V is an **open, royalty-free instruction set architecture** — unlike ARM or x86,
nobody needs a license to implement or extend it. That makes it the architecture of
choice for a lot of current chip research and embedded silicon work: you can inspect,
modify, and simulate the entire hardware/software stack without hitting an IP wall.
For anyone doing computer architecture or embedded systems work, RISC-V is where you
can actually see every layer, from the ISA spec to the bootloader to the kernel.

## Why build Linux from scratch instead of flashing an image?

Because the value isn't the working system at the end — it's everything you're forced
to understand to get there:

- How a CPU goes from power-on to a login prompt (firmware → bootloader → kernel → init)
- Why cross-compilation exists and how a toolchain targets a foreign architecture
- What a root filesystem actually is, and how it's assembled rather than just "installed"
- How QEMU stands in for real RISC-V silicon during development

## The boot chain, and why each stage exists

```
 power on
    │
    ▼
 OpenSBI   →  RISC-V's supervisor firmware layer (like ARM's EL3/PSCI firmware).
    │          Initializes the hart, hands control to the bootloader in supervisor mode.
    ▼
 U-Boot    →  The bootloader. Finds the kernel + device tree on the SD image, loads
    │          them into memory, and jumps to the kernel entry point.
    ▼
 Linux     →  The kernel proper, cross-compiled for rv64, mounts the root filesystem
 kernel        and starts init.
    │
    ▼
 rootfs    →  The actual userspace — shell, libraries, utilities — built with
              debootstrap so it's a real (if minimal) Debian system, not a toy.
```

Every step in the guide builds one link in that chain, on a host machine that's
also being set up correctly to build for a *different* architecture than itself
(x86_64 Debian host → rv64 RISC-V target), which is why the toolchain steps come
before anything RISC-V-specific.

## Glossary

If you're new to embedded Linux, these terms show up constantly in the guide:

- **Kernel** — the core of the OS. It talks directly to the hardware (CPU, memory,
  disks, network), manages processes, and is the one piece of software that runs
  in privileged mode the whole time the machine is on. Everything else (your
  shell, `ls`, apps) runs on top of it and asks it for resources.
- **Rootfs (root filesystem)** — the userspace: `/bin`, `/etc`, `/lib`, all the
  actual programs and config files the kernel mounts as `/` once it's done
  initializing. The kernel alone can't do anything useful — it needs a rootfs
  to hand off to (that handoff is literally what "boots to a login prompt" means).
- **SD image** — a single file that's a byte-for-byte layout of what would be on
  a physical SD card: partition table, bootloader, kernel, rootfs, all in one
  blob. QEMU can boot directly from this file, simulating what a real board
  would do reading from a physical SD card. Building it (steps 18–19) means
  manually creating the partitions and copying each piece into the layout a
  real bootloader expects — nothing is done for you here.
- **OpenSBI** — RISC-V's supervisor firmware. Before an OS can run, something has
  to initialize the CPU core ("hart" in RISC-V terminology) and set up the
  privilege-mode handoff. OpenSBI is that "step zero" firmware, conceptually the
  RISC-V equivalent of ARM Trusted Firmware.
- **U-Boot** — the bootloader. Once firmware hands off, something has to find the
  kernel image on disk, load it into RAM, and jump to it. That's U-Boot's only
  job — it doesn't run Linux, it just launches it.
- **Cross toolchain** — a compiler (and linker, binutils, etc.) that runs on your
  host architecture (x86_64) but produces binaries for a *different* architecture
  (RISC-V rv64). You can't just use your normal `gcc` here — it only knows how
  to emit x86_64 instructions.
- **QEMU** — a machine emulator. Since most people don't have a physical RISC-V
  board, QEMU emulates one entirely in software, letting the SD image built in
  this guide boot exactly as it would on real hardware.
- **debootstrap** — a Debian tool that builds a working Debian rootfs from
  scratch for a target architecture, by downloading and unpacking packages
  directly rather than "installing" onto an existing system.
- **"leenix"** — the hostname (`riscvleenix`) and login username baked into this
  particular class build's rootfs — not a typo, just what the built system
  identifies itself as once it boots. If you rebuild the rootfs yourself with a
  different `debootstrap` config, you can name it whatever you want.

## What the 22 steps cover

| Steps | Phase | Why it's there |
|---|---|---|
| 0–2 | **Host setup** | Get a clean Debian 13 VM — the machine everything else is built on |
| 3–6 | **Linux fundamentals** | Basic commands, updates, package management, system status — the baseline skills the rest of the guide assumes |
| 7–9 | **Toolchains** | Install both a native x86_64 toolchain (to build host tools) and a RISC-V cross toolchain (to build target binaries), then prove it works with a real cross-compiled test program |
| 10–10b | **Networking** | Bridge the VM so the emulated RISC-V machine gets real network access, plus a fix for a routing conflict that shows up in practice |
| 11–12 | **Emulation setup** | Install QEMU (stands in for physical RISC-V hardware) and pull the class build scripts |
| 13 | **Kernel** | Configure and cross-compile Linux 6.1.177 for rv64 |
| 14 | **Root filesystem** | Build a real Debian userspace for RISC-V with `debootstrap` |
| 15 | **OpenSBI** | Compile the firmware layer that initializes the hart before anything else runs |
| 16 | **U-Boot** | Compile the bootloader that will load the kernel from the SD image |
| 17 | **Mirror fix** | Optional fix if the class APT mirror config needs updating |
| 18–19 | **SD image** | Create a virtual SD card image and lay down kernel + rootfs + U-Boot in the right layout |
| 20–21 | **Boot & verify** | Boot the whole stack in QEMU and confirm you're actually running rv64 Linux (`uname -a`, `/proc/cpuinfo`, memory, disk usage) |

## What this guide is (the tool itself)

A single-file, zero-dependency interactive command reference — no build step, no
server, no external libraries. Every command line shows the exact prompt
(`user@host:path$`) it was run from, so you always know which machine, user, and
directory a command belongs to.

**Features:**

- **Copy buttons** — copy a single command or an entire step's block in one click
- **Progress tracking** — check off completed steps, persisted in `localStorage`
- **Sidebar TOC** — auto-generated, deep-linkable, scroll-spy highlighted
- **Search/filter** — instantly filter steps and commands by keyword
- **Mobile-friendly** — collapsible sidebar, sticky progress bar, back-to-top button

## Usage

```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>
open index.html   # or: python3 -m http.server, then visit localhost:8000
```

### Hosting on GitHub Pages

1. Push to GitHub.
2. Repo **Settings → Pages → Deploy from branch → `main` / root**.
3. Live at `https://<your-username>.github.io/<repo-name>/`.

## Notes

- Default credentials throughout: user `hunter`, password `abc123` (root password
  is also `abc123`).
- Commands reference a local class network (`10.0.1.147`) and shared class build
  scripts — swap those endpoints if following this outside that environment.
- Tested for kernel `6.1.177`, U-Boot `2024.10`.

## Author

Built by **KRSS**, B.E. ECE, Sathyabama Institute of Science and Technology.

## License

MIT — see [LICENSE](LICENSE).
