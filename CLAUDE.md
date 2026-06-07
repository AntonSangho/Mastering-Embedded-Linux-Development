# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is the companion code for the book *Mastering Embedded Linux Development – Fourth Edition* (Packt), targeting Linux 6.6 and the Yocto Project 5.0 (Scarthgap). It is **not a single application**: each `ChapterNN/` directory is a self-contained set of examples for that chapter of the book. There is no top-level build, test, or CI — code is built and run per-example, almost always cross-compiled for and executed on target hardware (BeaglePlay, Raspberry Pi 4) or QEMU, not on the development host.

When changing code, stay within the chapter that owns it. The errata section of `README.md` is authoritative — it records corrections to specific files keyed by book page number, and edits to those files (e.g. `Chapter06/buildroot/board/meld/nova/post-image.sh`, the renamed `dummy-char` module) must stay consistent with what the errata documents.

## Cross-compilation model

Almost nothing here is meant to run on the host. The C examples are plain Makefiles that respect the standard toolchain variables, so you cross-compile by overriding `CC` (and `CFLAGS`/`LDFLAGS`):

```bash
make CC=aarch64-buildroot-linux-gnu-gcc          # cross-compile a C example
make CC=arm-linux-gnueabihf-gcc                   # 32-bit Arm
make                                              # host build (testing only)
```

The toolchains the book uses are the Bootlin aarch64 toolchain (`~/aarch64--glibc--stable-2024.02-1/bin`) and the Arm GNU AArch32 toolchain. `list-libs` (root of repo) inspects a binary's dynamic dependencies and honors `$CROSS_COMPILE` for the cross `readelf`.

## Common build patterns by example type

**Userspace C examples** (most of `Chapter02`, `Chapter13`, `Chapter14`, `Chapter17`, `Chapter18`): a Makefile with `PROGRAM = <name>` building a single binary. `make` builds, `make clean` removes it. Note `LOCAL_CFLAGS` carries example-specific flags (e.g. `-pthread` for the threading demos) and is kept separate from the overridable `CFLAGS`.

**Out-of-tree kernel modules** (`Chapter19/mbx-driver`, `mbx-driver-oops`): `obj-m` Makefiles built against a kernel tree, not the host:

```bash
make -C <KERNEL_SRC> M=$(pwd) modules
```

**Yocto layers** (`meta-*` directories, e.g. `Chapter06/meta-nova`, `Chapter07/meta-mackerel`, `Chapter10/meta-ota`, `Chapter11/meta-device-drivers`): standard BitBake metadata layers — `conf/layer.conf` declares the layer, recipes live under `recipes-*/<name>/<name>_<ver>.bb`, and `LAYERSERIES_COMPAT_* = "scarthgap"` pins the Yocto release. These are added to a `bblayers.conf` and built inside a `poky` build environment (`source poky/oe-init-build-env <build-dir>`, then `bitbake <image>`), not from this repo directly. Out-of-tree kernel module recipes `inherit module`.

**Buildroot packages and board support** (`Chapter06/buildroot`, `Chapter19/buildroot`, `Chapter20/buildroot`): these mirror the layout of a Buildroot tree (`package/<name>/{Config.in,<name>.mk}`, `board/<vendor>/<board>/`, `configs/<board>_defconfig`) and are copied into a Buildroot checkout. `*_SITE` paths in `.mk` files may contain author-specific absolute paths (e.g. `/home/frank/MELD/...`) that must be adjusted for the local checkout.

## Target hardware / runtime

Code is exercised on real targets and QEMU, so "running" usually means deploying to a board:

- `format-sdcard.sh <drive>` — partitions a microSD for the BeaglePlay (64 MiB FAT boot + 1 GiB ext4 root). Takes a raw block device name like `sdb` or `mmcblk0`; it runs `dd`/`sfdisk`/`mkfs` with `sudo` and refuses devices larger than ~120 GiB as a safety check.
- `Chapter04/build-linux-rpi4.sh` — reference script for building the RPi 4 kernel + boot files with the Bootlin toolchain (note: the book's snippet has `$` prompt prefixes on the `git clone` lines — these are illustrative, not runnable verbatim).
- `Chapter05/run-qemu-initramfs.sh` — boots a built aarch64 kernel + initramfs under `qemu-system-aarch64 -M virt -cpu cortex-a53`.

The per-chapter hardware/software matrix (which board, OS, and toolchain each chapter expects) is in the "Software and hardware list" table in `README.md`.

## Languages present

Mostly C (userspace + kernel modules) and Python (`Chapter12` sensor/NMEA parsing, `Chapter15` packaging, `Chapter17/zeromq` IPC, `Chapter19/tp.py`), plus shell scripts and BitBake/Buildroot makefile metadata. There is no shared library or common module across chapters — treat each example independently.
