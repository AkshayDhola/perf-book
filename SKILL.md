---
name: rust-performance
description: Optimize Rust programs for runtime speed, memory usage, binary size, or compile times. Use when profiling Rust code, tuning Cargo build configuration (opt-level, LTO, codegen-units, allocators, linkers), reducing heap allocations, shrinking type sizes, speeding up hashing/iterators/IO, removing bounds checks, or diagnosing slow builds. Triggers: "make this Rust code faster", "why is my Rust program slow", "reduce binary size", "reduce allocations", "faster compile times", "profile Rust", "benchmark Rust".
---

# Rust Performance

Distilled from *The Rust Performance Book* by Nicholas Nethercote
(<https://nnethercote.github.io/perf-book/>, dual MIT/Apache-2.0). Consult the
book itself for full prose, caveats, and its many real-world example links.
This file is a condensed working reference, not a substitute for it.

Scope: intermediate/advanced Rust. Techniques are practical and proven, but the
book is biased toward compiler-style workloads (rustc) over, say, scientific
computing.

## The Governing Rule

**Measure, then change, then measure again.** Every technique below has a
trade-off, and most have been observed to *hurt* some programs. Never apply a
list of optimizations wholesale. Change one thing at a time and benchmark it.

Optimize only hot code — optimized code is more complex and more expensive to
maintain. When a function is hot there are two moves: make it faster, or call
it less often. The largest wins usually come from algorithm and data-structure
changes, not micro-optimization.

## Workflow

1. **Confirm it's a release build.** `cargo build --release`. 10–100x speedups
   over dev builds are routine, and "my Rust is slow" is very often just this.
2. **Run Clippy.** `cargo clippy` — the "Perf" lint group catches many common
   problems automatically, and its suggestions usually make code simpler too.
   Assume anything Clippy catches by default is already handled.
3. **Benchmark**, so you can tell whether a change helped.
4. **Profile**, to find what's actually hot.
5. **Apply targeted techniques** from the sections below.
6. **Re-benchmark.**

## Benchmarking

Need realistic workloads (real-world inputs beat microbenchmarks, though
microbenchmarks and stress tests help in moderation) and a harness:

- **Criterion**, **Divan** — sophisticated `cargo bench` harnesses.
- **Hyperfine** — general-purpose CLI benchmarking tool.
- **Bencher** — continuous benchmarking on CI.
- **Gungraun** — `cargo bench` integration with high-precision measurements.
- Built-in benchmark tests — nightly-only (unstable features).
- Custom harnesses (e.g. rustc-perf) for large projects.

Wall-time matches user perception but has high variance; tiny memory-layout
changes cause real but ephemeral swings. Lower-variance metrics such as cycles
or instruction counts are often a better signal.

Mediocre benchmarking beats no benchmarking. Don't stall on building a perfect
setup.

## Profiling

Pick by platform and question:

| Tool | Good for |
|---|---|
| perf (+ Hotspot / Firefox Profiler) | general-purpose, hardware counters; Linux |
| samply | sampling profiler viewed in Firefox Profiler; Mac/Linux/Windows |
| flamegraph | Cargo command, perf/DTrace → flame graph |
| Instruments | general-purpose; macOS/Xcode |
| Intel VTune, AMD μProf | general-purpose; Windows/Linux (VTune also macOS) |
| Cachegrind / Callgrind | per-function and per-line instruction counts, simulated cache/branch data |
| DHAT | allocation sites, allocation rates, peak memory, hot `memcpy` |
| dhat-rs | portable, less powerful DHAT alternative; also does heap-usage *tests* |
| heaptrack, bytehound | heap profiling; Linux |
| `counts` | ad hoc profiling: `eprintln!` + frequency post-processing |
| Coz (coz-rs) | causal profiling — measures optimization *potential* |

Use more than one; they have different strengths.

Profiling setup, all in `Cargo.toml` / `RUSTFLAGS`:

```toml
[profile.release]
debug = "line-tables-only"   # source line info for release profiles
```

```bash
RUSTFLAGS="-C force-frame-pointers=yes" cargo build --release  # better stacks
RUSTFLAGS="-C symbol-mangling-version=v0" cargo build --release  # if demangling misbehaves
```

Shipped std is built without debug info, so std frames stay opaque; the
reliable fix is building your own compiler/std with `debuginfo-level = 1` in
`bootstrap.toml`. (`build-std` compiles std with your config but doesn't fetch
its source, so source-requiring profilers like Cachegrind and samply still
won't fully work.) Mangled names beginning `_ZN` or `_R` can be decoded with
`rustfilt`.

## Build Configuration

Cargo reads profile settings only from the workspace-root `Cargo.toml` —
dependencies' profile settings are ignored, so this chapter is mostly about
binary crates. `cargo-wizard` can pick a configuration for you.

**Maximize runtime speed:**

```toml
[profile.release]
codegen-units = 1     # fewer missed optimizations; slower compiles
lto = "fat"           # whole-program opt, 10-20%+; "thin" is a milder middle ground
panic = "abort"       # only if you don't need unwinding / catch_unwind
```

`lto` values: `"off"` (disabled), `false` (thin *local* LTO — the release
default, not the same as `"off"`), `"thin"`, `"fat"`. Fat isn't always better
than thin — measure.

Plus an alternative allocator — often a large speed and memory win, at some
binary-size and compile-time cost:

```toml
[dependencies]
tikv-jemallocator = "0.5"   # jemalloc; Linux and Mac
# or
mimalloc = "0.1"            # mimalloc; many platforms
```

```rust,ignore
#[global_allocator]
static GLOBAL: tikv_jemallocator::Jemalloc = tikv_jemallocator::Jemalloc;
```

On Linux, jemalloc can additionally use transparent huge pages via
`MALLOC_CONF="thp:always,metadata_thp:always"` set before building (the running
system must also be configured for THP).

Also consider:
- `-C target-cpu=native` if you don't need portable binaries — enables newer
  instructions (AVX etc.) and helps vectorization. Verify it took effect by
  diffing `rustc --print cfg` against `rustc --print cfg -C target-cpu=native`;
  fall back to specific `-C target-feature` flags if not.
- **PGO** (10%+ possible) via `cargo-pgo`, which also wraps BOLT. Not usable
  for crates.io binaries installed with `cargo install`.

**Minimize binary size:**

```toml
[profile.release]
opt-level = "z"    # or "s", which is less aggressive and allows more inlining
                   # plus loop vectorization
codegen-units = 1
lto = "fat"
panic = "abort"
strip = "symbols"  # harms debuggability; backtraces get less useful
```

Debug info needs no stripping — it isn't generated for local release builds by
default, and std's has been stripped from release builds since Rust 1.77. For
more, see the `min-sized-rust` repository.

**Minimize compile times:**

- **Use a faster linker.** This is the one change in the whole chapter with no
  trade-off — if it works for your program, it's free. lld is default on Linux
  since Rust 1.90 and works on Windows; mold (Linux) is often faster than lld;
  wild (Linux) may be faster still but is less mature. macOS's system linker is
  already fast. Via `RUSTFLAGS="-C link-arg=-fuse-ld=lld"` or `config.toml`
  `[build] rustflags = ["-C", "link-arg=-fuse-ld=lld"]`.
- **Drop debug info in dev builds** if you rarely use a debugger — 20–40%
  faster dev builds. `[profile.dev] debug = false`, or
  `debug = "line-tables-only"` to keep line numbers in stack traces and most of
  the win.
- **Nightly parallel front-end**: `RUSTFLAGS="-Zthreads=8"`. Up to 50% in the
  best cases, nothing in others; costs compile-time memory, doesn't change
  codegen quality. 8 tends to be the best value.
- **Cranelift back-end** (nightly, dev builds only — worse generated code):
  `rustup component add rustc-codegen-cranelift-preview --toolchain nightly`,
  then `RUSTFLAGS="-Zcodegen-backend=cranelift" cargo +nightly build`.

Custom profiles are available if you want something between `dev` and
`release`.

## Heap Allocations

Allocation typically means a global lock, non-trivial data-structure work, and
possibly a syscall. Small allocations aren't necessarily cheap. If `malloc` /
`free` show up hot, cut the allocation rate or swap the allocator. DHAT is the
right tool; in rustc, cutting ~10 allocations per million instructions has been
worth ~1%.

**`Vec`** — three words (len, capacity, pointer). Growth is quasi-doubling:
0, 4, 8, 16, 32… so pushing 20 items one at a time costs four allocations.
- Know the size? `Vec::with_capacity` / `reserve` / `reserve_exact`.
- Know the max? Same functions avoid excess capacity; `shrink_to_fit` trims
  later (but may reallocate).
- Unsure of the distribution? `eprintln!` the length at the hot site and
  post-process with `counts`.
- Many short vectors → `SmallVec<[T; N]>` from `smallvec` (spills to the heap
  past `N`; slightly slower per-op, and can be *bigger* than `Vec` if `N` or
  `T` is large — remember `vec![]` → `smallvec![]`).
- Many short vectors with a known maximum → `ArrayVec` from `arrayvec`, faster
  still since there's no heap fallback.

**`String`** — behaves like `Vec<u8>`; `with_capacity` and friends apply.
`smallstr::SmallString` mirrors `SmallVec`; `smartstring::String` is a drop-in
that inlines strings under three words (≤23 ASCII chars on 64-bit). `format!`
always allocates — a string literal, `format_args!`, or `lazy_format` may avoid
it.

**Hash tables** — `HashSet`/`HashMap` are a single contiguous allocation that
reallocates as they grow; `with_capacity` applies as with `Vec`.

**`Box`** — simple; mainly useful for shrinking types (see Type Sizes).

**`Rc`/`Arc`** — two refcounts alongside the value. `clone` is just a refcount
bump, no allocation. But using them for values that are rarely actually shared
adds allocations that wouldn't otherwise exist.

**`clone`** — usually allocates. `a.clone_from(&b)` is `a = b.clone()` that can
reuse `a`'s existing allocation:

```rust
let mut v1: Vec<u32> = Vec::with_capacity(99);
let v2: Vec<u32> = vec![1, 2, 3];
v1.clone_from(&v2); // v1's allocation is reused
assert_eq!(v1.capacity(), 99);
```

`clone` is fine and often simplifies code — target only the hot ones. Some are
simply unnecessary and can be deleted outright.

**`to_owned`/`to_string`** — same story. Sometimes avoidable by storing a
reference in a struct instead of an owned copy, at the cost of lifetime
annotations. Only when profiling justifies it.

**`Cow`** — for mixtures of borrowed and owned data, avoiding
promote-everything-to-owned allocations:

```rust
use std::borrow::Cow;
let mut errors: Vec<Cow<'static, str>> = vec![];
errors.push(Cow::Borrowed("something went wrong"));
errors.push(Cow::Owned(format!("something went wrong on line {}", 100)));
errors.push(Cow::from("something else went wrong"));
errors.push(format!("something else went wrong on line {}", 101).into());
```

Works for `&str`/`String`, `&[T]`/`Vec<T>`, `&Path`/`PathBuf`. `Cow::to_mut`
gives clone-on-write mutation. `Cow` derefs to the inner data. Fiddly, often
worth it.

**Reuse collections.** Prefer modifying one passed-in `Vec` over building
several and combining. A "workhorse" collection declared outside a loop and
`clear`ed each iteration avoids per-iteration allocation (at the cost of
obscuring that iterations are independent). Same idea for a workhorse
collection held in a struct across repeated method calls.

**Reading lines** — `BufRead::lines` allocates a `String` per line. A workhorse
`String` with `read_line` cuts that to a handful, often one:

```rust
# fn blah() -> Result<(), std::io::Error> {
# fn process(_: &str) {}
use std::io::{self, BufRead};
let mut lock = io::stdin().lock();
let mut line = String::new();
while lock.read_line(&mut line)? != 0 {
    process(&line);
    line.clear();
}
# Ok(())
# }
```

Only works if the body can operate on `&str`.

**Guard against regressions** with dhat-rs's heap usage testing.

## Type Sizes

Shrinking oft-instantiated types cuts peak memory, memory traffic, and cache
pressure. Types over 128 bytes are copied with `memcpy` rather than inline
code — if `memcpy` is hot, DHAT's copy-profiling mode names the culprits, and
getting under 128 bytes removes the calls.

Measure with `std::mem::size_of`, or get full layout from nightly:

```text
RUSTFLAGS=-Zprint-type-sizes cargo +nightly build --release
```

It reports size, alignment, enum discriminant size, per-variant sizes (largest
first), field ordering, and padding. `top-type-sizes` renders it more compactly.

Ways to shrink:
- **Field ordering**: nothing to do — the compiler already reorders fields to
  minimize size, unless `#[repr(C)]`.
- **Box an outsized enum variant**: `Z(i32, LargeType)` → `Z(Box<(i32,
  LargeType)>)`. Costs an allocation for that variant, so it's a win mainly
  when that variant is rare; also slightly worse ergonomics in `match`.
- **Smaller integers**: `u32`/`u16`/`u8` indices coerced to `usize` at use
  points, instead of storing `usize`.
- **Boxed slices**: `Vec::into_boxed_slice` drops the capacity word (3 words →
  2). `let bs: Box<[u32]> = (1..3).collect();` builds one directly, with no
  reallocation when the iterator length is known. `slice::into_vec` converts
  back with no cloning or reallocation.
- **`ThinVec`** (`thin_vec` crate): `Vec`-equivalent that stores len and
  capacity in the elements' allocation, so it's one word. Good for
  often-empty vectors inside hot types, or shrinking an enum's largest variant.

Lock in the win with a static assertion, gated on architecture since sizes vary
by platform:

```rust,ignore
#[cfg(target_arch = "x86_64")]
static_assertions::assert_eq_size!(HotType, [u8; 64]);
```

## Hashing

Default hasher is SipHash 1-3: high quality, collision-resistant, but slow —
especially for short keys like integers. If hashing is hot **and HashDoS is not
a threat for your application**, swap it:

- **`rustc-hash`** (`FxHashSet`/`FxHashMap`) — low-quality but very fast,
  especially for integer keys; the best performer inside rustc. (`fxhash` is
  the older, less-maintained implementation.)
- **`fnv`** — higher quality than rustc-hash, a little slower.
- **`ahash`** — can exploit hardware AES instructions.
- **`nohash-hasher`** — for newtypes wrapping already-random integers.

Real rustc numbers, showing the choice matters and doesn't transfer: fnv →
fxhash gave up to 6% speedups; fxhash → ahash gave 1–4% *slowdowns*; fxhash →
default hasher gave 4–84% slowdowns. Try more than one.

**Byte-wise hashing** (advanced): `#[derive(Hash)]` hashes fields separately;
for types with no padding it can be faster to hash the whole value as a byte
stream. `zerocopy` and `bytemuck` provide `#[derive(ByteHash)]`; see the
`derive_hash_fast` README. Highly dependent on hash function and type layout —
measure carefully.

## Standard Library Types

- `vec![0; n]` is the best way to build a zero-filled `Vec` — it can use OS
  assistance and beats `resize`/`extend`/`unsafe` alternatives.
- `Vec::remove` is O(n) (shifts); `Vec::swap_remove` is O(1) but doesn't
  preserve order. `Vec::retain` removes many items efficiently (equivalents
  exist for `String`, `HashSet`, `HashMap`).
- Eager vs lazy: `ok_or(expensive())` always evaluates its argument; use
  `ok_or_else(|| expensive())`. Same for `Option::map_or`,
  `Option::unwrap_or`, `Result::or`, `Result::map_or`, `Result::unwrap_or`.
- `Rc::make_mut`/`Arc::make_mut` give clone-on-write: clone only if the
  refcount exceeds one, otherwise mutate in place.
- `parking_lot` offers alternative `Mutex`/`RwLock`/`Condvar`/`Once`. It used
  to be reliably better, but std has improved a lot on some platforms —
  measure before switching.
- Read the docs for `Vec`, `Option`, `Result`, `Rc`/`Arc` generally; they're
  full of useful methods.

Committed to an alternative type everywhere? Enforce it with Clippy's
`disallowed_types` so std versions can't creep back in:

```toml
disallowed-types = ["std::collections::HashMap", "std::collections::HashSet"]
```

## Iterators

- Avoid `collect` when the collection is only iterated again. Returning `impl
  Iterator<Item = T>` beats returning `Vec<T>` (sometimes needs extra
  lifetimes).
- `extend` an existing collection rather than `collect`-then-`append`.
- Implement `Iterator::size_hint` or `ExactSizeIterator::len` on iterators you
  write — downstream `collect`/`extend` then allocate less.
- `chain` can be slower than a single iterator; consider avoiding in hot
  iterators. `filter_map` may beat `filter` + `map`.
- Prefer `slice::chunks_exact` over `slice::chunks` when the chunk size divides
  the length exactly; even when it doesn't, `chunks_exact` +
  `ChunksExact::remainder` (or manual leftover handling) can still win. Same
  for the `rchunks` / `_mut` / `into_remainder` family.
- `iter().copied()` over collections of small types (integers) hands values by
  value rather than by reference, and LLVM may do better. Advanced — verify in
  the generated machine code.

## Inlining

Four attributes: none (compiler decides), `#[inline]` (suggestion),
`#[inline(always)]` (strong — inlines in all but exceptional cases),
`#[inline(never)]`. Inlining is **not transitive**: for `f` calling `g` to
inline together at `f`'s call site, both need attributes.

Best candidates: very small functions, or functions with a single call site —
though the compiler usually handles those itself.

Cachegrind tells you whether a function was inlined: it was inlined **iff** its
first and last lines carry no event counts.

Always re-measure. Adding an attribute can have no effect (because a nearby
function stopped being inlined), or slow things down. Cross-crate inlining also
costs compile time.

**Large function, multiple call sites, only one hot?** Split it:

```rust
# fn one() {};
# fn two() {};
# fn three() {};
// Use this at the hot call site.
#[inline(always)]
fn inlined_my_function() {
    one();
    two();
    three();
}

// Use this at the cold call sites.
#[inline(never)]
fn uninlined_my_function() {
    inlined_my_function();
}
```

**Outlining** is the inverse: move rarely executed code into its own function
and mark it `#[cold]`, improving codegen on the hot path.

## I/O

- `print!`/`println!` lock stdout on **every call**. In a loop, lock once
  (`let mut lock = stdout.lock();`) and `writeln!` into it. Same for stdin and
  stderr.
- Rust file I/O is **unbuffered by default** — wrap in `BufWriter`/`BufReader`
  for many small operations. Forgetting is most common on the write side,
  because buffered and unbuffered writers both implement `Write` so the code
  looks identical; readers differ (`Read` vs `BufRead`), which makes the
  omission obvious.
- Call `flush()` explicitly. Dropping flushes too, but silently swallows any
  error.
- Buffering composes with locking — do both for heavy stdout writing.
- `String` costs UTF-8 validation on read. For byte-oriented work (e.g. ASCII),
  `BufRead::read_until` avoids it; see also the `linereader` and `bstr` crates.

## Bounds Checks

Slice/vector accesses are bounds-checked; this matters in hot loops, though
less often than expected. Safe fixes, in order of preference:

1. Replace direct indexing in a loop with iteration.
2. Take a slice of the `Vec` before the loop and index the slice inside it.
3. Add assertions constraining index-variable ranges.

Tricky to get right — see the Bounds Check Cookbook. `get_unchecked` /
`get_unchecked_mut` are the unsafe last resort.

## Machine Code

For small, very hot snippets, read the generated assembly: Compiler Explorer
(godbolt.org) for snippets, `cargo-show-asm` for whole projects. `core::arch`
exposes architecture-specific intrinsics, many SIMD-related.

## Logging and Debugging

Logging/debugging code — and the data collection feeding it — can dominate.
Make sure nothing unnecessary is computed when logging is disabled.

`assert!` always runs; `debug_assert!` runs only in dev builds. A hot assertion
that isn't needed for *safety* should be a `debug_assert!`.

## Wrapper Types

Accessing `RefCell`, `Mutex`, etc. costs non-trivial time. Values usually
accessed together are often better under a single wrapper:

```rust
# use std::sync::{Arc, Mutex};
struct S {
    xy: Arc<Mutex<(u32, u32)>>,   // instead of two Arc<Mutex<u32>> fields
}
```

Depends entirely on access patterns.

## Parallelism

Out of scope for the book, but: `rayon` and `crossbeam` for thread-based
parallelism, *Rust Atomics and Locks* (marabos.nl/atomics) as the reference.
For fine-grained data parallelism, see the state-of-SIMD-in-Rust survey
(Shnatsel, November 2025).

## Compile Times (code changes)

Build-configuration fixes are above; these need code changes. Corrode's "Tips
for Faster Rust Compile Times" covers more.

- **Visualize**: `cargo build --timings` emits an HTML Gantt chart of crate
  dependencies — shows how much parallelism your crate graph allows and which
  large crates serialize the build.
- **Macros**: `cargo +nightly rustc -- -Zmacro-stats` (one leaf crate) or
  `RUSTFLAGS="-Zmacro-stats" cargo +nightly build` (all crates) reports how
  much code each macro generates; proc macros are usually the notable ones.
  `cargo-expand` shows the generated code. Ignore small ones; a macro
  generating code comparable to your hand-written code is worth removing,
  replacing, or slimming down.
- **LLVM IR bloat**: `cargo llvm-lines` shows which functions generate the most
  IR. Generic functions dominate, since they're instantiated many times. Fixes:
  make the function smaller, or move the non-generic body into an inner
  non-generic function instantiated once —

```rust,ignore
pub fn read<P: AsRef<Path>>(path: P) -> io::Result<Vec<u8>> {
    fn inner(path: &Path) -> io::Result<Vec<u8>> {
        // ... real work here, instantiated once ...
        # unimplemented!()
    }
    inner(path.as_ref())
}
```

  Heavily-instantiated helpers like `Option::map` and `Result::map_err` can be
  replaced with `match` expressions. Effects are usually small, occasionally
  large; these changes also tend to shrink binaries.

## General Principles

- Rust is already fast and memory-lean once the obvious pitfalls (dev builds)
  are avoided — especially versus Python/Ruby or GC'd Java/C#.
- Most optimizations yield small speedups. They add up if you land enough.
- Eliminating silly slowdowns is usually easier than inventing clever speedups.
- Avoid computing anything unnecessary; lazy/on-demand computation often wins.
- Check optimistically for simple common cases before the complex general one.
  Special-casing collections of 0, 1, or 2 elements is often a win when small
  sizes dominate.
- Repetitive data compresses: compact representation for common values, a
  fallback table for the rest.
- With multiple cases, measure their frequencies and handle the most common
  first.
- High-locality lookups can benefit from a small cache in front of the data
  structure.
- Mind cache misses and branch mispredictions.
- Optimized code has non-obvious structure — comment it, ideally citing the
  measurement: "99% of the time this vector has 0 or 1 elements, so handle
  those cases first."
