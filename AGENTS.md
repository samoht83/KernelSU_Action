# KernelSU Action — Agent Guide

This repo is a **GitHub Actions workflow** that cross-compiles a non-GKI Android kernel with KernelSU. You do not run anything locally; all work happens via `workflow_dispatch` on GitHub.

## Architecture

- Single workflow: `.github/workflows/build-kernel.yml` (manual trigger only)
- All user-facing config is in `config.env` — boolean vars check for the literal string `true`
- KernelSU for non-GKI kernels: last supported version is **v0.9.5** (v1.0+ is GKI-only). Pin with `KERNELSU_TAG=v0.9.5`
- Supported kernel versions: 5.4 / 4.19 / 4.14 / 4.9
- If kprobes don't work in the target kernel, enable `APPLY_KSU_PATCH=true` — this runs `patches/patches.sh` against the kernel source

## Build flow (from CI workflow)

1. Install system deps (bc, flex, libssl-dev, etc.)
2. `git clone --depth=1` the kernel source at the configured branch
3. Run KernelSU setup script: `curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -s -- $KERNELSU_TAG`
4. Append `CONFIG_KSU=y`, `CONFIG_KPROBES=y`, etc. to defconfig (appended, not inserted — overrides earlier values)
5. Fetch toolchains: Proton Clang (kdrag0n) + AOSP GCC 4.9 (aarch64 + arm32)
6. Build: `make O=out <defconfig>` then `make -j$(nproc --all) O=out`
7. Copy resulting `Image.gz-dtb` or `Image` to `output/`
8. **Upload** as a workflow artifact (not a release)

## Key variables in config.env

| Variable | Purpose |
|---|---|
| `KERNEL_SOURCE` | Git URL of kernel source |
| `KERNEL_SOURCE_BRANCH` | Branch to clone |
| `KERNEL_CONFIG` | Defconfig path (e.g. `n200_defconfig`) |
| `KERNEL_IMAGE_NAME` | Binary to flash: `Image`, `Image.gz`, or `Image.gz-dtb` |
| `CLANG_BRANCH` / `CLANG_VERSION` | AOSP Clang branch/version (see README table) |
| `ENABLE_GCC_ARM64` / `ENABLE_GCC_ARM32` | Use AOSP GCC 4.9 cross-compilers |
| `KERNELSU_TAG` | Must be `v0.9.5` for non-GKI (not `main`) |
| `KSU_EXPECTED_SIZE` / `KSU_EXPECTED_HASH` | Custom manager signature; get with `ksud debug get-sign <apk_path>` |
| `DISABLE_LTO` / `DISABLE_CC_WERROR` | Workarounds for build failures |
| `ADD_KPROBES_CONFIG` / `ADD_OVERLAYFS_CONFIG` | Auto-inject defconfig entries |
| `APPLY_KSU_PATCH` | Runs `patches/patches.sh` to manually patch kernel source |
| `BUILD_BOOT_IMG` / `SOURCE_BOOT_IMAGE` | Build a full boot.img (needs source boot.img from same kernel tree) |

## Output & flashing

- Output artifact is AnyKernel3 zip (device check already disabled) — flash in TWRP
- If `BUILD_BOOT_IMG=true`, a `boot.img` is produced instead
- DTBO is optionally uploaded via `NEED_DTBO=true`

## Important gotchas

- All `config.env` values are read as-is by the workflow, but boolean checks only match `true` — anything else is falsy
- Custom clang source supports git URLs (include `.git`) or direct zip links
- Custom AnyKernel3 source supports the same
- The `config.env` in this repo is pre-configured for **OnePlus Nord N200 (dre/holi, sm4350)** — change all values before for other devices
- The workflow uses `workflow_dispatch` only — no push/schedule triggers
- ccache is available (`ENABLE_CCACHE=true`) and significantly speeds rebuilds
