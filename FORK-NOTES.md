# Fork Notes

## What this fork is

Fork of <https://github.com/peterzheng98/GPU-Microarch-Bench> (Wenxin Zheng).
Initial commit byte-identical to upstream `main` (tree `d28172f`, commits
`ef26ddc` + `0482b77`).

## Why the fork exists

Goal: characterize the GDDR6X address structure of an RTX 3090 (GA102,
384-bit bus, 24 GDDR6X packages) to selectively stress individual
partitions/channels and localize a suspected cooling problem in one GDDR6X
device. Planned additions (NOT present in upstream, NOT yet implemented):

- RTX 3090 / GA102-specific address-mapping characterization
- per-partition / per-channel stress selection
- thermal diagnostics (temperature vs. latency correlation per device)

## Upstream code status

No upstream source file has been modified. The only changes in this fork's
history so far are documentation (README fork notice + this file).

## Build findings (verified 2026-10-05, target: Windows + CUDA 13.3 + VS2022 + RTX 3090)

1. **CUDA 13.3 compile blocker.** `dram_bank_row.cu` uses `prop.clockRate`
   (lines 819, 834). `cudaDeviceProp::clockRate` was **removed in CUDA 13.0**
   (release notes, "Removed Fields and Their Replacements": use
   `cudaDeviceGetAttribute(&clk, cudaDevAttrClockRate, dev)`). The
   unmodified code therefore **fails to compile under CUDA 13.3**; it
   compiles unchanged under CUDA 12.x. Proposed first functional change
   (pending approval): query the SM clock via `cudaDeviceGetAttribute`.

2. **Architecture flags.** The Makefile emits `sm_80` and/or `sm_90` cubins
   only — never `sm_86`, no PTX fallback. An `sm_80` cubin runs on `sm_86`
   (binary compatibility: same major, equal-or-higher minor), so
   `make ARCH=ampere` suffices for the 3090. CUDA 13.x still supports
   Ampere; only pre-Turing (Maxwell/Pascal/Volta) was dropped in 13.0.

3. **Windows build.** Makefiles are GNU-make syntax. On Windows use WSL or
   MSYS2 `make`, or invoke nvcc directly:
   `nvcc -O3 -std=c++17 -lineinfo -gencode arch=compute_80,code=sm_80 -o dram_bank_row.exe dram_bank_row.cu`
   CUDA 13.3 supports Windows x86_64 with VS2022 integration; since CUDA
   13.1 the display driver is NOT bundled — install a driver >= 580
   separately.

4. **Memory-type label.** `detect_mem_type()` keys on bus width: >= 1024-bit
   -> HBM variants, otherwise `MEM_GDDR6`. The 3090 (384-bit) is printed as
   "GDDR6", never "GDDR6X" (the enum exists but is never assigned). The GDDR
   probing path is still the correct strategy for GDDR6X; only the label is
   imprecise.

5. **Pair-test behavior on the 3090.** `--pair-test` runs the GDDR phase-2
   kernel: L2 flush, row-opening reference load, data-dependent timed
   candidate load (`clock64`), 128 tests per batch, threshold = midpoint of
   observed min/max latency, bank-count estimate from same-bank fraction.
   HBM hammer paths are not used on GDDR devices.

## Planned first functional change (pending approval)

Replace the two `prop.clockRate` uses with a
`cudaDeviceGetAttribute(cudaDevAttrClockRate)` query so the tool compiles on
CUDA 13.x. No other source changes.
