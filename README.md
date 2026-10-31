# turboblake

**BLAKE3 checksums, with a parallel [Bend2](https://bend-lang.com) engine.**
`turboblake` is a drop-in replacement for `b3sum`: the same flags, output and
BLAKE3 digests, with faster paths for batches of files and large inputs.

This fork of [BLAKE3-team/BLAKE3](https://github.com/BLAKE3-team/BLAKE3) adds
[`bend/`](bend/): the Bend implementation, C and Rust libraries, bindings for
Python, JavaScript and Go, iroh-compatible bao outboards, and native Metal
and CUDA kernels for GPU memory. The command line uses the CPU. The tables
below show wins, ties and slower cases against the official implementation.

Site with every chart: **[gunta.github.io/turboblake](https://gunta.github.io/turboblake/)**

| on an Apple M5 Pro (6 Super + 12 Performance cores, 20-core GPU) | official BLAKE3 1.8.7 | turboblake |
|---|---|---|
| `b3sum` a 101.5 GB DeepSeek-V4.1 shard (bigger than RAM) | 211 s (526 s first run) | **7.65 s** (27.6x, ~13 GB/s off the SSD) |
| hash 1 GiB already in GPU memory, native Metal | (no GPU path) | **5.85 ms (~184 GB/s)** |
| hash 1 GiB in shared memory, CPU + native Metal | (no GPU path) | **5.25 ms (~205 GB/s)**; GPU alone on the same memory: 5.68 ms |
| hash 1 GiB in memory, all CPU cores | 27.8 ms (NEON + rayon) | 26.9 ms |
| hash 1 GiB, one CPU core | 448 ms (NEON) | **377 ms** (1.2x) |
| `b3sum` 2,000 x 64 KiB files | 275 ms | **25.7 ms** (10.7x) |
| `b3sum` 8,000 cargo source files | 260 ms | **73.5 ms** (3.5x) |
| `b3sum` one 5 GiB file | 249 ms | **242 ms** |
| iroh-blobs ingest of 1 GiB (hash + bao outboard) | 837 ms (iroh-blobs) | **51.7 ms** (16x) |
| re-hash 1 GiB in a long-running process | 43.6 ms (rayon) | **29.4 ms** on the GPU (1.48x) |

These are medians from separate benchmark suites on the same machine, with
matching BLAKE3 digests within each comparison. GPU results exclude initial
setup and use GPU-accessible memory; they are not file-to-checksum timings.
The full tables, methods, ties and losses are below and on the site.

## Install

**turboblake** is the command: BLAKE3, turbocharged. It is the official
`b3sum` (every flag, format and message) with many files and big ones hashed
at once by a Bend2 engine: BLAKE3 is a Merkle tree, and Bend2 runs
tree-shaped work in parallel on every CPU core and the GPU.

```bash
brew install gunta/tap/turboblake
```

Homebrew on macOS or Linux, prebuilt for arm64 and x86-64. Or build it from
source, tuned for your CPU (needs clang: Apple's, or LLVM clang 19+ on Linux;
no Bend toolchain, the crate ships the C that Bend2 emits):

```bash
cargo install turboblake
```

`cargo binstall turboblake` fetches the prebuilt binaries instead, and every
archive is on the [releases page](https://github.com/gunta/turboblake/releases).

```bash
turboblake FILE...           # one "<hash>  <name>" line per file, as b3sum prints
turboblake FILE... > B3SUMS  # write checksums
turboblake --recursive DIR   # every file under DIR, one line each
turboblake --check B3SUMS    # verify them, like sha256sum -c
```

`--recursive` (new in 1.8.9; b3sum has no such flag) lists the regular files under each
directory, as `find DIR -type f` does, in name order and without following
links; its output is a checkfile for `--check`. On 65k Rust sources it is 3.8x
faster than `find DIR -type f -print0 | xargs -0 b3sum`
([ADR 0022](adr/0022-hashing-directories.md)).

`cargo install turboblake --features replace` also installs it as `b3sum`.

**BLAKE3 and SHA-256 checksums are different.** `turboblake --check` reads
BLAKE3 checksum files created by `b3sum` or `turboblake`. Keep using
`sha256sum -c SHA256SUMS` for existing SHA-256 manifests.

Prebuilt for macOS, Linux (glibc 2.28+, or static musl) and Windows, arm64
and x86-64. The x86-64 builds use AVX2 where the CPU has it and run on any
x86-64 CPU; on Windows turboblake is the official b3sum until the engine
runs there ([ADR 0020](adr/0020-every-package-on-its-registry.md)).

### Libraries

The C and Rust libraries keep the upstream BLAKE3 API, with hooks for the
Bend CPU engine and native GPU backends. Bindings expose these libraries
to other languages
([ADR 0012](adr/0012-the-engine-as-a-library-and-every-binding.md)).

| language | replaces | swap in |
|---|---|---|
| Rust | `blake3` | `cargo add turboblake`; the library is named `blake3` ([README](bend/rust/README.md)) |
| C | the official `libblake3` | `brew install gunta/tap/libturboblake`, a release archive, or CMake; then `pkg-config --cflags --libs libturboblake` or `find_package(turboblake)` ([README](bend/lib/README.md)) |
| Python | `blake3` on PyPI | `pip install turboblake` ([README](bend/python/README.md)) |
| Node, Bun, Deno | `@noble/hashes/blake3`, `blake3`, `@napi-rs/blake-hash` | `npm install turboblake` (or `bun add`, `deno add npm:turboblake`) ([README](bend/node/README.md)) |
| browsers, workers | `@noble/hashes/blake3`, `hash-wasm` | Install the JavaScript package above, then import `turboblake/js`; compatible hashing output, not every package's API |
| Go | `lukechampine.com/blake3`, `github.com/zeebo/blake3` | `go get github.com/gunta/turboblake/bend/go` ([README](bend/go/README.md)) |

Your code keeps its imports (`use blake3`, `from blake3 import blake3`) or
changes one import (JavaScript, Go); some alternative packages also require
API changes. Remove the existing Python `blake3` package before installing
this binding, since both provide the same module. The Python and JavaScript packages are
`turboblake` on PyPI and npm once published; until then they install from
this repository. `TURBOBLAKE=0` disables the native CPU engine;
`TURBOBLAKE_GPU=0` disables the native GPU backend. The browser build is
JavaScript only and does not read these environment variables.

## What is in here

| path | what |
|---|---|
| [`bend/blake3.bend`](bend/blake3.bend) | the hash, in Bend2 (generated by [`bend/tools/gen_blake3.py`](bend/tools/gen_blake3.py)) |
| [`bend/b3sum.bend`](bend/b3sum.bend) | the CLI |
| [`bend/b3io.bend`](bend/b3io.bend), [`bend/effs/`](bend/effs/) | the I/O effects: parallel loading, zero-copy `mmap`, output |
| [`bend/crate/`](bend/crate/) | the `turboblake` crate, the library and the CLI, generated by [`bend/rust/gen.sh`](bend/rust/gen.sh) from upstream's crate and `b3sum` |
| [`bend/lib/`](bend/lib/) | libturboblake: the official C library with the hook, and the engine ([`engine.bend`](bend/lib/engine.bend)) |
| [`bend/rust/`](bend/rust/) | the crate's generator and its Rust: the engine hook ([`bend.rs`](bend/rust/bend.rs)) and the CLI's fast path ([`fast.rs`](bend/rust/fast.rs)) |
| [`bend/python/`](bend/python/), [`bend/node/`](bend/node/), [`bend/go/`](bend/go/), [`bend/js/`](bend/js/) | the Python, Node/Bun/Deno, Go and browser packages |
| [`bend/bench/`](bend/bench/) | in-memory, one-shot, daemon, iroh-blobs and Modal GPU benchmarks |
| [`bend/test/`](bend/test/) | the official test vectors on both paths, big-file tests |
| [`adr/`](adr/) | every decision that made it faster, with its measurements |
| [`site/`](site/) | the GitHub Pages site |
| [`scripts/`](scripts/) | syncing with upstream BLAKE3 and re-checking everything |
| `src/`, `c/`, `b3sum/`, `reference_impl/`, ... | upstream BLAKE3, unchanged ([its README](README.upstream.md)) |

## How it is fast

The parallel CPU engine implements BLAKE3 compression and tree traversal in
Bend2, compiled to C. The libraries keep upstream code for small inputs and
fallbacks, and use hand-written Metal or CUDA kernels for GPU-accessible
memory. The Bend proofs cover the Bend implementation, not the native GPU
kernels, integration code or compiler/runtime; those paths are checked
against reference outputs.

- **8 lanes in registers** ([ADR 0002](adr/0002-eight-lane-compress-and-slp.md)).
  Bend has no vector types, so `compress8` hashes eight chunks per call in a
  shape clang's SLP vectorizer packs into NEON (or AVX) registers: word-major
  lane records, one call site, tuned `-mllvm` flags. 2.95 GB/s per core against
  hand-written NEON's 2.40.
- **The BLAKE3 tree is a fork tree** ([ADR 0001](adr/0001-bend2-blake3-baseline.md)).
  Each node's halves are a Bend parallel let; `Array.fork` hands the input down
  in O(1); 32-chunk leaves keep tasks coarse.
- **GPU paths** ([ADR 0005](adr/0005-gpu-flat-leaves.md),
  [0017](adr/0017-the-gpu-backend.md)). Bend’s `hash1!` uses flat
  4/16/64-chunk leaves. The libraries now also load native Metal and CUDA
  kernels for GPU buffers and can split GPU-accessible host memory between
  the CPU engine and GPU. Ordinary CLI file hashing stays on the CPU.
- **I/O that gets out of the way** ([ADR 0003](adr/0003-b3sum-io-path.md)).
  One parallel pass opens and reads every file; big files are mapped read-only
  into a zone of their own (the array *is* the page cache); one `write` prints
  everything.
- **Any size** ([ADR 0004](adr/0004-big-files-and-the-gpu.md),
  [0014](adr/0014-files-past-4-tib.md)). Files past 1 GiB are 1 GiB segments,
  each a complete BLAKE3 subtree, hashed in parallel and merged as parents;
  each segment carries its chunk counter's high word, so files go past 4 TiB
  (2^32 chunks), as BLAKE3 does.
- **SSD speed for files not in RAM** ([ADR 0009](adr/0009-streaming-uncached-files-and-gpu-daemon.md)).
  Page faults read the SSD 16 KiB at a time; a file mostly not in the page
  cache streams instead, 1 GiB at a time, read by 16 `F_NOCACHE` threads into
  one buffer while every core hashes the other, its segment CVs merged as
  BLAKE3's own `Hasher` merges chunks.
- **A GPU daemon** ([ADR 0009](adr/0009-streaming-uncached-files-and-gpu-daemon.md)).
  A long-running process pays the GPU's set-up once, refills reused GPU-heap
  blocks each round and hashes every file in one bang.
- **iroh-blobs outboards** ([ADR 0008](adr/0008-daemon-map-zone-and-iroh-outboards.md)).
  The CPU tree also emits the bao outboard iroh-blobs stores beside a blob
  (every parent pair above its 16 KiB chunk groups, in pre-order),
  byte-identical to `bao-tree`'s.

## Benchmarks

Apple M5 Pro, macOS, Bend2 2.0.39, official BLAKE3 1.8.7 (`b3sum`, and the
`blake3` crate with NEON + rayon).

**Bend engine in memory**, warm, same data and digest. The GPU column is
Bend’s GPU implementation, not the newer native Metal backend:

| input | Rust blake3 | Bend CPU | Bend GPU |
|---|---|---|---|
| 1 GiB, 1 core | 448 ms | **377 ms** | |
| 16 MiB, all cores | 1.13 ms | **0.50 ms** | 0.88 ms |
| 64 MiB, all cores | 1.95 ms | 1.74 ms | **1.21 ms** |
| 256 MiB, all cores | 6.9 ms | 6.6 ms | **2.43 ms** |
| 1 GiB, all cores | 27.8 ms | 26.9 ms | **9.4 ms** |

**Libraries**, one-shot hash of 1 GiB in memory through each package's API
(`bend/bench/collect.py` folds the runs into results.json):

| language | turboblake | the packages it replaces |
|---|---|---|
| C | **38.7 ms** | official C 481 ms (one thread) |
| Rust | 37.8 ms | `blake3::hash` 507 ms; `update_rayon` **31.9 ms** |
| Python | 41.9 ms | `blake3` 505 ms; `max_threads=AUTO` **34.0 ms** |
| Node | **38.2 ms** | `@napi-rs/blake-hash` 482 ms, hash-wasm 964 ms, `@noble/hashes` 10.9 s |
| Bun | **46.7 ms** | `@napi-rs/blake-hash` 479 ms, hash-wasm 1.9 s, `@noble/hashes` 3.1 s |
| Go | **40.5 ms** | lukechampine 99.6 ms (every core), zeebo 1144 ms |
| Go, Linux x86-64 (Modal) | **23.9 ms** | lukechampine 57.4 ms, zeebo 392 ms |

These library measurements predate the macOS address-window reservation in
[ADR 0016](adr/0016-turboblake-never-slower-than-b3sum.md): they copied large inputs
into Bend's heap, and the official crate on every core won from 16 MiB up.
The current library reserves an address window at load time so subsequent
allocations can be read in place; inputs outside it still need a copy. Linux
also reads input directly when it lies in the engine’s window. These tables
have not been rerun for the new macOS path.

**One-shot `b3sum FILE...`**, process start included, output byte-identical:

| workload | b3sum | turboblake | |
|---|---|---|---|
| 1 GiB file | 49.8 ms | 48.4 ms | 1.03x |
| 64 MiB file | 7.4 ms | 7.3 ms | 1.01x |
| 5 GiB file | 249.4 ms | 242.0 ms | 1.03x |
| 10,000 x 4 KiB | 148.5 ms | 77.1 ms | 1.93x |
| 2,000 x 64 KiB | 275.4 ms | 25.7 ms | 10.7x |
| 1,000 x 1 MiB | 241.7 ms | 70.6 ms | 3.42x |
| macOS SDK headers (3,460 files) | 129.1 ms | 37.5 ms | 3.45x |
| cargo sources (8,000 files) | 259.5 ms | 73.5 ms | 3.53x |

**Real model weights**: shards of
[deepseek-ai/DeepSeek-V4.1-Flash-1017](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash-1017),
one-shot, identical hashes:

| input | b3sum | turboblake | |
|---|---|---|---|
| shard 48, 101.5 GB (bigger than this Mac's 64 GB of RAM) | 211.4 s (526 s first run) | **7.65 s** | 27.6x |
| shards 1-6, 31.3 GB | 23.4 s | **4.17 s** | 5.6x |

**Daemon** (a long-running process re-hashing the same files every round, as a
sync service would; start-up excluded), against Rust that is parallel across
files as well as within them, the strongest baseline we could write with the
official crate. Bend's best side wins six of seven, by up to 1.48x (the GPU
on big inputs); the SDK headers are a tie at the kernel's `open()` floor
(medians, ms per round, a busy machine):

| workload | Rust, files in order (b3sum) | Rust, rayon across files | Bend CPU | Bend GPU |
|---|---|---|---|---|
| 1 GiB file | 44.7 | 43.6 | 44.6 | **29.4** |
| 64 MiB file | 4.1 | 3.3 | 3.3 | **3.0** |
| 2,000 x 64 KiB | 422.5 | 16.3 | 16.0 | **14.1** |
| 1,000 x 1 MiB | 522.9 | 39.2 | 49.2 | **31.8** |
| 10,000 x 4 KiB | 152.8 | 54.0 | **51.3** | 57.4 |
| cargo sources (8,000 files) | 307.0 | 43.9 | **42.9** | 49.2 |
| macOS SDK headers | 164.7 | **21.5** | 22.0 | 26.0 |

**iroh-blobs** 0.103 ingest (`add_path`: BLAKE3 + bao outboard, plus its store
bookkeeping) against Bend computing the same hash and the same outboard: a
1 GiB file takes iroh 837 ms in its fastest mode (its outboard is computed on
one core), Bend 51.7 ms; 1,000 x 1 MiB files, 88.5 ms against 57.3 ms.

**Where the floors are.** Page faults on a mapped file cost ~15 ms per GiB and
serialize in the kernel; all 18 cores top out near 38 GB/s for BLAKE3 (Rust and
Bend both reach it); macOS `open()` costs ~48 ms per 10,000 files however many
threads ask; and a GPU-enabled process pays ~50 ms per GiB of Metal heap at
start-up, so the one-shot CLI stays on the CPU. Details in [ADR 0003](adr/0003-b3sum-io-path.md)
and [ADR 0004](adr/0004-big-files-and-the-gpu.md).

**NVIDIA** (Modal; Bend's CUDA heap built from 1 GiB managed pieces, since the
sandbox grants no bigger one, [`bend/bench/modal_gpu.py`](bend/bench/modal_gpu.py)):
on a B200, Bend hashes 1 GiB in 3.2 ms and 2 GiB in 4.8 ms (444 GB/s), against
0.68 and 1.28 ms for a hand-written CUDA kernel and 13.9 and 24.2 ms for the
Rust crate on the host's 16 cores. Native kernels are faster on every GPU
measured in this suite; Bend’s throughput exceeds the SSD and network rates
used in these comparisons, and it is
the path covered by the Bend proofs. The libraries also ship the tested
native kernels: 1 GiB already in GPU memory takes 0.686 ms on the B200 and
5.85 ms on the M5 Pro in the newer library suite. These exclude setup and
host-to-device transfers. See [the GPU backend write-up](adr/0017-the-gpu-backend.md)
and [the GPU page](https://gunta.github.io/turboblake/gpu/) for the separate
suites and their memory conditions.

## Reproduce

From a clone, build the current command line without the Bend toolchain:

```bash
cargo build --release --manifest-path bend/crate/Cargo.toml
bend/crate/target/release/turboblake --help
```

To rebuild the Bend implementation and rerun the reference comparisons:

```bash
bend/build.sh bend/b3sum.bend bend/build/b3sum
scripts/refresh.sh
```

`refresh.sh` rebuilds the official `b3sum` and the Rust references, regenerates
and rebuilds the Bend code, checks the official test vectors on both paths,
`b3sum` output byte for byte, files around the 1 GiB segment edges, and bao
outboards against `bao-tree`, then runs the one-shot and daemon races. Needs
[Bend2](https://bend-lang.com) (`curl -fsSL https://bend-lang.com/install.sh | sh`),
Rust and clang. The GPU benchmark on NVIDIA hardware runs on Modal, and costs
money: `modal run bend/bench/modal_gpu.py --gpu H100!`.

## Keeping up with upstream

```bash
scripts/update-upstream.sh
```

One command for everything turboblake copies or mirrors
([scripts/upstream/upstreams.json](scripts/upstream/upstreams.json)): BLAKE3
itself, blake3-py (the Python package), the Go and JavaScript packages the
bindings stand in for, and Bend2. It says what moved, merges upstream BLAKE3
on a `sync/upstream-YYYYMMDD` branch, three-way merges the files we copy,
writes every diff that matters to `.upstream/REPORT.md`, and hands the branch
to a coding agent (`claude`; `--headless`, or `--no-agent` to stop at the
report). The agent decides what to adopt by the rules in
[scripts/upstream/AGENT.md](scripts/upstream/AGENT.md), regenerates and checks
every package (`scripts/upstream/verify.sh`), records the new pins, commits on
the branch, and leaves pushing and releasing to you.
`python3 scripts/upstream/upstream.py status` only reports what moved.

The BLAKE3 step alone is `scripts/sync-upstream.sh`: it rebases this fork onto
`BLAKE3-team/BLAKE3` master (or merges, with `--merge`, as the update does),
keeps this README (upstream's lives on in
[README.upstream.md](README.upstream.md)), and runs `scripts/refresh.sh`.

## Credits and license

BLAKE3 is by Jack O'Connor, Jean-Philippe Aumasson, Samuel Neves and Zooko
Wilcox-O'Hearn; everything outside `bend/`, `adr/`, `site/` and `scripts/` is
their code, unchanged. Bend2 is by [bendlang](https://github.com/bendlang/bend)
(Apache-2.0). This fork is licensed like BLAKE3: CC0-1.0 OR Apache-2.0 OR
Apache-2.0 WITH LLVM-exception.
