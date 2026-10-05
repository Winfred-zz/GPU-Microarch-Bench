# GPU-Microarch-Bench

> **Fork notice.** This is a fork of
> [peterzheng98/GPU-Microarch-Bench](https://github.com/peterzheng98/GPU-Microarch-Bench)
> by Wenxin Zheng (research intern, Apex Lab / IPADS, SJTU). All benchmark
> code and the probing methodology are the upstream author's work; this
> fork's initial commit is byte-identical to upstream `main`. The fork exists
> to extend the benchmarks for **RTX 3090 (GA102, 24 GB GDDR6X) memory
> address-structure characterization and thermal diagnostics** of individual
> GDDR6X devices. The upstream author is not affiliated with, endorsing, or
> responsible for this fork or its additions. The upstream repository does not
> publish a license; this fork preserves full attribution and makes no license
> claim over upstream code. See `FORK-NOTES.md` for what is changed and why.

CUDA micro-benchmarks for probing GPU DRAM micro-architecture.

## Benchmarks

### `dram_bank_row` — DRAM Bank & Row Size Discovery

Discovers DRAM bank count and row size by detecting row-buffer conflicts
and bank-level serialisation through latency measurements.

**Supports both HBM and GDDR GPUs** — the memory type is auto-detected
from the device properties and the probing strategy adapts accordingly.
A single launch can test every GPU in the system.

| Feature | HBM (A100 / H100 / H800) | GDDR (A6000 / RTX) |
|---------|---------------------------|---------------------|
| Memory type | Explicitly reported (HBM2 / HBM2e / HBM3) | Explicitly reported (GDDR6) |
| Phase 1 | Stride P-chase — shows L2→DRAM transition | Stride P-chase — reveals bank stride & row size |
| Phase 2 | **Hammer** — automatic, ld.lu 2-element alternating P-chase accounting for 3-D stack structure | Pair-wise conflict test with L2 flush (opt-in) |
| Page policy | Closed-page (flat DRAM latency in Phase 1) | Open-page (row-buffer hits / conflicts visible) |

#### How it works

**Phase 1 — Stride P-chase** (both HBM and GDDR):

Pointer-chase at varying strides through a buffer larger than L2.  Each
step uses `ld.global.cg` (bypass L1, use L2).  When the working set
exceeds L2, every step is a DRAM access.

- **GDDR** (open-page): strides that place consecutive accesses in the
  same bank / different row cause row-buffer conflicts → elevated latency.
- **HBM** (closed-page): DRAM latency is ~flat regardless of stride.
  The P-chase still reveals the L2→DRAM transition.

**Phase 2 — Memory-specific probing**:

- **GDDR pair-wise test** (`--pair-test`): for each candidate offset,
  flush L2, load the reference (opens its DRAM row), then time the
  candidate load.  Same-bank / different-row → conflict → higher latency.

- **HBM hammer** (automatic): 2-element alternating P-chase using
  `ld.global.lu` (last-use cache eviction) so every step hits DRAM.
  Same-bank pairs are serialised by the row-cycle time (tRC), producing
  measurably higher per-step latency than different-bank pairs.  The scan
  covers three granularity bands to expose the full **3-D HBM structure**:

  | Band | Range | What it probes |
  |------|-------|----------------|
  | Fine | 64 B – 4 KB (step 64 B) | Channel interleave unit |
  | Medium | 4 KB – 256 KB | Bank stride within a channel |
  | Coarse | 256 KB – 64 MB+ | Stack-level structure |

  A diagnostic check (`ld.cg` vs `ld.lu` 2-element chase) runs first to
  verify that `ld.lu` effectively bypasses L2.  If not, the tool falls
  back to the L2-flush approach.

#### Logging

Every run writes a detailed log file (default `dram_bench.log`) containing
per-iteration raw data, diagnostic results, and all measurements.  The log
is written alongside the summary printed to stdout.

## Building

Requires CUDA Toolkit (≥ 11.0) and `nvcc`.

```bash
# Fat binary (both sm_80 + sm_90, default — needed for multi-GPU)
make

# Single architecture
make ARCH=ampere
make ARCH=hopper
```

## Running

```bash
cd dram_bank_row

# Test all GPUs (Phase 1 + automatic HBM hammer)
./dram_bank_row

# Test a specific GPU
./dram_bank_row --device 0

# Also run GDDR pair test (for GDDR GPUs)
./dram_bank_row --pair-test

# Custom allocation / scan range
./dram_bank_row --alloc 8192 --hbm-range 128

# CSV + custom log
./dram_bank_row --csv results.csv --log my_run.log
```

### Command-line options

| Flag | Default | Description |
|------|---------|-------------|
| `--device <N\|all>` | all | GPU to test |
| `--log <file>` | `dram_bench.log` | Detailed log file |
| `--csv <file>` | — | CSV output |
| `--alloc <MB>` | 4096 | GPU buffer size |
| `--stride-min <B>` | 128 | Min P-chase stride |
| `--stride-max <B>` | 4 MB | Max P-chase stride |
| `--steps <N>` | 200000 | Timed P-chase steps |
| `--warmup <N>` | 10000 | Warmup steps |
| `--iters <N>` | 3 | Averaging iterations |
| `--pair-test` | off | Enable GDDR pair test |
| `--pair-range <MB>` | 32 | Pair scan range |
| `--pair-stride <B>` | 256 | Pair scan granularity |
| `--pair-iters <N>` | 10 | Pair measurement repeats |
| `--hbm-steps <N>` | 20000 | Hammer steps per probe |
| `--hbm-range <MB>` | 64 | Hammer scan range |
| `-v` | off | Verbose |

## Interpreting results

### GDDR GPUs

In Phase 1, look for a band of strides with noticeably higher latency:

- **First conflict stride** ≈ inter-bank stride.
- **Last conflict stride** ≈ row size.
- **Bank count** ≈ last / first.

Phase 2 (pair test) gives per-address conflict data.

### HBM GPUs

Phase 1 will show flat DRAM latency (closed-page policy).  Phase 2
(hammer) is where the bank structure is revealed:

- **Same-bank latency** is higher than **different-bank latency** by
  ~10–25% due to tRC serialisation.
- The fine-scan region (64 B – 4 KB) reveals the **channel interleave**
  unit (e.g., 256 B ⇒ addresses 256 B apart map to different channels).
- Periodicity in same-bank offsets reveals the **bank stride**.
- The estimated HBM 3-D structure (stacks, channels, banks) is printed
  for reference.

> **Note on NUMA effects:** large HBM configurations have non-uniform
> access latency depending on physical proximity.  This can widen the
> latency distribution.  The hammer averages over many steps to reduce
> noise.

## Initial RTX 3090 Testing

### Windows / CUDA compatibility

`dram_bank_row` was built and tested on:

- GPU: EVGA GeForce RTX 3090 FTW3 Ultra V2
- GPU architecture: GA102 / SM 8.6
- VRAM: 24 GB GDDR6X
- Memory bus: 384-bit
- L2 cache: 6 MB
- CUDA: 13.3
- Compiler: Visual Studio 2022 / MSVC 14.42

The upstream source required several small compatibility changes to compile with CUDA 13.3 and MSVC.

Added:

```cpp
#include <string>
```

Replaced a GCC statement-expression used when formatting the scan range with a normal local buffer:

```cpp
char range_buf[32];
fmt_bytes(range, range_buf, sizeof(range_buf));
emit("    Scan points: %zu  (up to %s)\n",
     n_tests, range_buf);
```

CUDA 13.3 no longer exposes `cudaDeviceProp::clockRate`, so the SM clock is obtained using:

```cpp
int clock_khz = 0;
CUDA_CHECK(cudaDeviceGetAttribute(&clock_khz, cudaDevAttrClockRate, dev));
double ghz = clock_khz / 1.0e6;
```

The benchmark was compiled directly with `nvcc` rather than using the supplied GNU Makefile:

```powershell
nvcc -O3 -std=c++17 -lineinfo -gencode arch=compute_80,code=sm_80 -o .\dram_bank_row\dram_bank_row.exe .\dram_bank_row\dram_bank_row.cu
```

These changes are intended only as Windows/CUDA compatibility fixes. The benchmark algorithm has not been modified.

### GDDR6X detection limitation

The benchmark reports the RTX 3090 as `GDDR6`:

```text
NVIDIA GeForce RTX 3090  GDDR6  24575 MB  sm_86
```

The source currently classifies non-HBM GPUs as GDDR6 even though `MEM_GDDR6X` exists in the enum. This is currently only a labeling issue: the RTX 3090 still follows the intended GDDR benchmark path.

The detection code has deliberately been left unchanged until a baseline has been established.

### First baseline run

GPU-intensive LLM workloads were stopped before testing to avoid interference with VRAM availability, cache state, memory traffic, clocks, and latency measurements.

A reduced Phase 1 test was used instead of the much larger defaults:

```powershell
.\dram_bank_row\dram_bank_row.exe --device 0 --alloc 512 --steps 20000 --warmup 2000 --iters 1
```

The GPU was detected as:

```text
Compute      : 8.6
Memory       : 24575 MB total, 23335 MB free
L2 Cache     : 6144 KB
SM Clock     : 1800 MHz
Mem bus      : 384 bit
Buffer       : 512 MB
```

Phase 1 produced a clear address-dependent latency signal:

```text
DRAM latency range : 424.2 - 645.5 cycles
Conflict ratio     : 1.52x

First conflict stride : 128 B
Last conflict stride  : 8.0 KB
Approx bank count     : ~64
Approx row size       : 8.0 KB
```

Particularly strong latency peaks appeared at:

```text
1.5 KB : 645.5 cycles
8.0 KB : 638.8 cycles
```

The `~64 banks` and `8 KB row` values are heuristic results from the benchmark and should not yet be treated as confirmed GA102 DRAM geometry.

The important initial result is that the benchmark produces a strong, structured DRAM latency signal on the RTX 3090, indicating that it should be possible to proceed with address-mapping experiments.

### Next step

The next experiment is the GDDR pair test. This compares individual candidate addresses against a fixed reference address after flushing L2 and is intended to identify addresses exhibiting same-bank/different-row conflict behavior.

The initial test will use a relatively small 2 MB search range before expanding the experiment:

```powershell
.\dram_bank_row\dram_bank_row.exe --device 0 --alloc 512 --steps 20000 --warmup 2000 --iters 1 --pair-test --pair-range 2 --pair-stride 256 --pair-iters 10 --csv pair-2mb.csv --log pair-2mb.log
```

The immediate goal is to determine whether the conflicting offsets show a repeatable pattern that can be used to begin reverse-engineering the GA102 physical-address-to-DRAM mapping.

This is still an address-characterization experiment, not yet a workload capable of selectively stressing an individual GDDR6X package.