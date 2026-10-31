# ADR 0022: hashing directories (`--recursive`)

Date: 2026-10-31
Status: accepted (released in turboblake 1.8.9)

## Context

The official `b3sum` takes files only: `turboblake adr` prints `turboblake:
adr: Is a directory (os error 21)`, as `b3sum adr` does. To hash a tree,
people write `find DIR -type f -print0 | xargs -0 b3sum`, which works but
splits the list into one process per 5000 paths (BSD xargs' default), each
starting the engine and rayon, and has no answer on Windows.

"Hashing directories" means two different things:

1. **A line per file under a directory**, as `b3sum` prints for files. The
   output is a checkfile `--check` already reads. Nothing about BLAKE3 or the
   line format changes.
2. **One digest for a whole tree.** That needs a format, fixed for good: how
   paths are encoded and ordered, links, empty directories, the executable
   bit, Windows separators. Git trees, Nix's NAR and tools like
   [rblake3sum](https://docs.rs/crate/rblake3sum) each chose differently and
   none interoperate.

Many files is where turboblake wins most (ADR 0016: a batch of more than 16
files goes to the engine unseen), so (1) is also where a flag can be faster
than the pipeline it replaces.

## Decision

`--recursive` does (1) and not (2).

- **The flag.** Long only, no `-r`: b3sum has no such flag, and a short one
  is what upstream would take first if it ever adds one with other meaning.
  It conflicts with `--check` (checkfiles are not walked) and `--raw` (one
  input only). Without it nothing changes: a directory is still an error.
- **What it lists.** The regular files under each directory argument, as
  `find DIR -type f` lists them: links inside are not followed (no loops,
  nothing counted twice), fifos, sockets and devices are skipped (reading a
  fifo blocks), hidden files and `.git` are listed. A link given as an
  argument is followed, as b3sum follows a file argument. File arguments
  and `-` are hashed as without the flag, in their place among the others.
- **The order.** Depth first, each directory's entries in name (byte) order,
  so the output is the same on every run and `turboblake --recursive DIR |
  turboblake` is a stable digest for the tree, which covers most of (2)
  without a format. It is not `find | sort`'s order: `b/c` comes before
  `b-x`, where sort puts `-` (0x2d) before `/` (0x2f).
- **Errors.** A directory it cannot list is reported as a missing file is
  (`turboblake: tree/b/d: Permission denied (os error 13)`); the walk goes
  on and the exit status is 1.

### How it runs (`fast.rs`)

`Inputs` replaces `Batch`: one iterator over the inputs, each with its hash
if the engine took it, taken `BATCH` (2048) at a time, each batch sent to the
engine if `engine_wins` says so. For plain arguments the batches and the
decisions are the ones `Batch` made. With `--recursive` a thread of its own
walks the tree (`walk`: `read_dir`, the entry's type from `d_type`, so no
`stat` per file) and sends paths through a channel up to two batches ahead,
so listing overlaps hashing. std only; no new dependency.

## Results

M5 Pro (18 cores), files in the page cache, load 4-14 from other sessions;
interleaved medians of 21 rounds (15 for the last two trees). Every
contender's sorted output was checked equal before timing.

| tree | files | `find \| xargs b3sum` | `find \| xargs turboblake` | `turboblake --recursive` | vs b3sum |
|---|---:|---:|---:|---:|---:|
| one 5-byte file | 1 | 9.1 ms | 8.7 ms | 6.0 ms | 1.50x |
| five 300 KB files | 5 | 9.5 ms | 9.1 ms | 6.6 ms | 1.44x |
| `bend/lib` | 87 | 18.9 ms | 10.8 ms | 8.1 ms | 2.34x |
| `small4k` (4 KiB each) | 10,000 | 175.6 ms | 102.7 ms | 78.5 ms | 2.24x |
| `small64k` (64 KiB each) | 2,000 | 278.8 ms | 33.2 ms | 27.7 ms | 10.08x |
| `med1m` (1 MiB each) | 1,000 | 255.0 ms | 58.3 ms | 54.7 ms | 4.66x |
| 0-2 KiB in 1000 directories | 100,000 | 2252 ms | 1619 ms | 1258 ms | 1.79x |
| `~/.cargo/registry/src` (929 MB) | 65,365 | 3087 ms | 1243 ms | 810 ms | 3.81x |
| 1 GiB + 5 GiB | 2 | 279.5 ms | 278.9 ms | 278.1 ms | 1.01x |

The first two rows include `find` and `xargs`. Against b3sum given the files
themselves (hyperfine `-N`, medians of 300 runs): one file 1.89 ms for
`b3sum f`, 1.83 for `turboblake --recursive dir` (1.75 for `turboblake f`, so
the walker's thread and `read_dir` cost ~0.08 ms); five files 3.08 against
2.94. So `--recursive` is never slower than b3sum here, and the two big files
are the page-fault tie ADR 0016 describes.

The walk is not the bound: `find` lists the 100,000-file tree in 0.16 s of
`--recursive`'s 1.26, and it runs beside the hashing.

Checks: 9 new CLI tests (`bend/rust/recursive_tests.rs`: order, links,
fifos, file arguments among directories, an unreadable directory, more files
than a batch with a `--check` round trip, `--tag`, `--keyed`, the conflicts)
beside upstream's 17, on macOS and on Linux arm64 (`rust:1-trixie`, as a
non-root user so the permission test runs); `check_vectors.py`; sorted output
identical to `find DIR -type f -print0 | xargs -0 b3sum` on 65,365 Rust
sources, the site with its `node_modules` (25,057 files, 754 links skipped
by both), `bend/lib` and `bend/test`; `--check` of the 65,365-line output;
names with a newline and a backslash escaped as b3sum escapes them.

## Not done

- **One digest for a tree** (2 above): wait for someone to need a format;
  sorted `--recursive` output piped into turboblake covers the rest.
- **A parallel walk.** The walk is ~13% of the time on the tiniest files and
  overlaps the hashing; a parallel one would have to keep the order.
- **`.gitignore`, `--exclude`, following links.** `find` and `xargs` still
  do these.
- **Uncached trees** (macOS needs root to purge the page cache), x86-64 and
  Windows were not raced; Windows has no engine yet, so `--recursive` there
  is upstream's code over the same walk. Re-race on a quiet machine.
