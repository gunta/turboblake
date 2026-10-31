# Architecture decision records

How the Bend2 BLAKE3 got fast, and what we measured on the way.

Each record is also told as news on the site's changelog
(https://gunta.github.io/turboblake/changelog/). With a new record, add its
`headline`, `dek`, `stat` and `points` to `techniques` in
`bend/bench/results/results.json` (site/BRAND.md, "The changelog").

| ADR | decision |
|-----|----------|
| [0001](0001-bend2-blake3-baseline.md) | BLAKE3 core in Bend2, the baseline we race |
| [0002](0002-eight-lane-compress-and-slp.md) | 8-lane compress8 shaped for clang's SLP vectorizer; inline threshold |
| [0003](0003-b3sum-io-path.md) | b3sum I/O: parallel one-pass loader, zero-copy mmap, one-write output |
| [0004](0004-big-files-and-the-gpu.md) | files past 4 GiB as 1 GiB segments; the CLI stays on the CPU |
| [0005](0005-gpu-flat-leaves.md) | flat GPU leaves sized to the 16384-lane cube |
| [0006](0006-native-gpu-references.md) | native Metal and CUDA references; where Bend's GPU time goes (read-bound) |
| [0007](0007-laws-and-proofs.md) | LAWS.bend / PROOF.bend: known answers, lane laws, tree theorems; Array.fork moved to hash.bend |
| [0008](0008-daemon-map-zone-and-iroh-outboards.md) | daemon race, a zone for mapped files, iroh-blobs outboards, a progressive load |
| [0009](0009-streaming-uncached-files-and-gpu-daemon.md) | files not in the page cache streamed at SSD speed; a GPU daemon |
| [0010](0010-more-laws-outboards-counts-streaming.md) | more laws: the outboard tree's hash for every input, leaf and parent counts, the streaming stack |
| [0011](0011-strided-gpu-lanes-and-cpu-gpu-split.md) | strided GPU lanes past 512 MiB (1 GiB 13.2 -> 9.4 ms); the CPU and the GPU on one input, split by slices (8.0 ms, 1.06-1.12x the GPU alone) |
| [0012](0012-the-engine-as-a-library-and-every-binding.md) | the engine as a library: one hook in the official C and Rust, a Bend program on its own thread, drop-ins for C, Rust, Python, Node/Bun/Deno, browsers and Go |
| [0013](0013-native-gpu-kernels-where-they-measure-faster.md) | native Metal and CUDA kernels ship where they measure faster, with the CPU + GPU split; SHA-256 measured beside them (1 GiB: 574 ms SHA-256, 0.683 ms native CUDA on a B200) |
| [0014](0014-files-past-4-tib.md) | files past 4 TiB: every span carries its chunk counter's high word, down to compress8; no span crosses a 2^32-chunk block (law `span.nocarry`, checked by the engine) |
| [0015](0015-proofs-for-every-input.md) | the trees that run proven for every input: the CPU, GPU and keyed trees are the scalar tree, and that is BLAKE3's tree; tree1's split rule for every chunk count; the streaming loop as b3io runs it (stack.bend) is BLAKE3's tree over the segments; all for both words of the chunk counter (40 laws, 412 lemmas, gate 4:12) |
| [0016](0016-turboblake-never-slower-than-b3sum.md) | turboblake never slower than b3sum: the engine only where it wins, the macOS window over the host's memory, stdin / --no-mmap / --check rebuilt (up to 11x) |
| [0017](0017-the-gpu-backend.md) | the GPU backend: bend/ref's kernels in the library, the crate and the bindings, loaded at run time; GPU memory hashed in place (B200: 1 GiB in 0.686 ms), host memory split with the engine (M5 Pro 5.25 ms, 1.11x the GPU alone; B200 pinned 11.1 ms, 1.59x the CPU) |
| [0018](0018-one-name-turboblake.md) | one name, turboblake: the library and the CLI in one crate (`turboblake = "1.8"`, still `use blake3`), libturboblake, `TURBOBLAKE_*` |
| [0019](0019-the-keyed-loop-starts-from-constants.md) | the keyed loop starts from constants: the key comes in as an xor delta, so clang vectorizes it like the default loop (keyed 1 GiB 70-75 -> 34-36 ms, 2x per core) |
| [0020](0020-every-package-on-its-registry.md) | every package on its registry, prebuilt, on every CPU: the engine checks the CPU (an x86-64-v3 build runs on a Core 2); one npm package with 8 addons; wheels, crate, Go tag, C library (CMake, pkg-config, Homebrew); Windows as the official code; one tag publishes it all by trusted publishing |
| [0022](0022-hashing-directories.md) | `--recursive`: every regular file under a directory, one b3sum line each, in name order (up to 10x `find \| xargs b3sum` on many files, 3.8x on 65k Rust sources, a tie on two big ones); no single tree digest |
