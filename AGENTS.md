# AGENTS.md

This repo is **not** a kernel source tree. It contains only a GitHub Actions workflow
(`.github/workflows/build-kernel.yml`) that fetches and compiles the OnePlus Nord N200
(SM4350) kernel on demand. There is no local source to edit and no real build/test target
to run outside the CI job.

## What the workflow actually does
- Clones `https://github.com/tangalbert919/android_kernel_oneplus_sm4350.git` at branch
  `lineage-20` into `kernel-source/` at build time (`--depth=1`).
- Fetches three toolchains: **Proton Clang** (`kdrag0n/proton-clang`, tag `20210522`,
  LLVM/Clang 13.0.0), `aarch64-linux-android-4.9` (gcc64), and
  `arm-linux-androideabi-4.9` (gcc32). The GCC binaries are only for the legacy GAS
  assembler; compilation is done with Proton Clang.
- Builds `vendor/holi-perf_defconfig` (an alternative is `vendor/dre_defconfig`).
- KernelSU is patched in pre-build, **pinned to `v0.9.5`**: KernelSU 1.0+ is GKI-only
  and will not compile against this non-GKI (~4.19) kernel; the build also enables the
  kprobe backend (`CONFIG_KPROBES`/`CONFIG_KPROBE_EVENTS`/`CONFIG_MODULES`) it needs.
- Uploads `Image.gz-dtb` (fallback `Image`) from `out/arch/arm64/boot/` as an artifact.

## Required build environment (from the preflight step)
- `ARCH=arm64`, `SUBARCH=arm64`, `CC=clang`
- `CLANG_TRIPLE=aarch64-linux-gnu-`, `CROSS_COMPILE=aarch64-linux-android-`,
  `CROSS_COMPILE_ARM32=arm-linux-androideabi-`
- `PATH` must include the clang, gcc64, and gcc32 `bin/` dirs.
- apt deps: `bc bison build-essential flex gcc-multilib g++-multilib libelf-dev
  libncurses5-dev libssl-dev ...` (full list in the workflow's System Pre-flight step).
- `make O=out <defconfig>` must run before `make -j$(nproc) O=out`.

## Notes for agents
- Any "kernel change" must happen in the cloned `tangalbert919` source, not here; this
  repo only controls toolchain version, defconfig, and build flags.
- To reproduce a build locally, replicate the workflow steps in order:
  install deps → fetch source + 3 toolchains → export env → `defconfig` → `make`.
- The workflow is `workflow_dispatch`-only (manual trigger), not on-push.
