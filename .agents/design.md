# RFC 0044: `zugoose`, writing R packages in Goose

Adopted from [RFC 0044](https://github.com/pedrobtz/packages/blob/main/rfcs/started/0044-zugoose-goose-for-r-packages.md)
on 2026-10-25. From here on this file is authoritative; the RFC is history.

- **Status:** Proposed. Merging this file adopts the design as the starting
  point for the package's own `.agents/design.md`; each decision in section
  13 stays the maintainer's until the package repository records it.
  D3, shipping generated C, is the one most worth a second look.
- **Date:** 2026-10-25.
- **Packages:** `zugoose` (not yet created; not on CRAN and there is no
  `pedrobtz/zugoose` repository). No `Imports`. No `LinkingTo`.
  `Suggests: testthat`. `zugojson` becomes its first user (§9). MIT for
  `zugoose`'s own code; the pieces of Goose's runtime it copies stay
  Apache-2.0 (§4).
- **Source:** the conversation of 2026-10-25 on why `zugojson` was slower
  than `jsonlite` and on a framework for Goose in R packages;
  `pedrobtz/zugojson` pull requests #1 (the hand-built glue and runtime)
  and #2 (number formatting through Goose's `std` json); Goose at
  `aardappel/goose@e76d5c9`, its `src/runtime/runtime.h` and
  `docs/reference.md` §24; `savvy` (`yutannihilation/savvy`) and `cpp11`'s
  `cpp_register()` as prior art. Nothing has been built.

## 1. What zugoose is

`zugoose` lets an R package keep its internals in Goose. The author writes
`export fn` functions in `.goose` files under `tools/goose/`. `zugoose`
compiles them with a pinned Goose compiler on the author's machine, and
writes four things into the package: the generated C, the `.Call` entry
points and registration, the R wrappers, and an R-safe Goose runtime.
The package ships that C. Installing it needs a C compiler and nothing
else.

The one-line statement that governs every decision below:

> **Goose on the author's machine, plain C on CRAN's, and no Goose
> failure ever reaches past an R error.**

### 1.1 What zugoose is not

- **Not a Goose compiler or distribution.** It fetches and builds the
  pinned compiler for the author (§5), and never at install time (D3).
- **Not a binding of R's C API for Goose.** Goose code never touches an R
  object. Values cross at the `.Call` boundary only (§7).
- **Not `zugojson`.** That package is the first user and the acceptance
  test (§9), and stays a demonstration.

### 1.2 Where it sits among existing tools

`savvy` is the model. A Rust function marked `#[savvy]` gets a generated
C entry point and an R wrapper from `savvy-cli update`, and Rust errors
arrive in R as R errors. It needs a Rust toolchain at install time.
`extendr` does the same job with more of R's API exposed to Rust.
`cpp11`'s `cpp_register()` is the closer match on one axis: it generates
registration code on the author's machine and ships it, so installation
needs only a compiler.

`zugojson` is the hand-made instance this RFC generalises. Its glue is
`src/zugojson_r.c` (144 lines), its runtime `src/goose_r_runtime.c` (245
lines), its build `tools/regenerate.sh` (161 lines) and its CRAN gate
`tools/check-symbols.sh` (19 lines). Every new Goose package would copy
and edit all four. Of them, the runtime is the hard part, and the part
every package needs unchanged.

## 2. Scope

**In v0.1.0:**

- **Set-up** (§5): `use_zugoose()` adds the folders, the pin, the build
  files and a starting `.goose` file to a package.
- **Generation** (§4, §7): `zugoose_update()` compiles the Goose, then writes
  the entry points, the registration and the R wrappers from the header
  the compiler emits.
- **The runtime** (§6): `zugojson`'s runtime with the improvements in
  §6.2, written once and copied into each package.
- **A Goose helper module** (§8): `import zugoose;` for classed errors,
  interrupt checks and output to the R console.
- **Verification** (§5): `zugoose_verify()` regenerates into a temporary
  folder and fails on any difference, for CI.
- **CRAN gates** (§9): the symbol check and the warning patches from
  `zugojson`, shared instead of copied.
- **POSIX and Windows** runtimes (§6.2).
- A README and one article, "Moving `zugojson` to `zugoose`".

**Not in v0.1.0, and why:**

- **R lists, data frames and environments across the boundary.** The
  first version maps vectors and scalars only (§7). Nested values wait for
  a package that needs them (§14).
- **Goose threads.** The runtime does not start workers. Contained workers
  are §14.
- **Compiling at install time.** D3 and Q2.
- **A C API for other packages.** Packages call each other through R.

## 3. Position in the `zu*` family

| | zugoose |
| --- | --- |
| Role | developer tool; its output is a runtime in other packages |
| R prefix | `zugoose_`, and `use_zugoose()` |
| Info function | `zugoose_info()` |
| Root condition class | `zugoose_error` |
| Public C prefix | `zugoose_` for the glue; `gs_` for the runtime (D7) |
| From C | no |
| Hides symbols | yes, in every generated package (R3, D7) |
| Vendored code | none; Goose runtime pieces go into users (§4) |
| `Depends: R` | 4.1 (R2) |
| Language | en-GB (R2) |

**Relationships:**

- **`zugojson`**: the first user. Its hand-written glue, runtime and
  scripts are replaced by `zugoose`'s output, with its test suite unchanged
  (§9).
- **`r-actions`**: a reusable workflow step that runs `zugoose_verify()` and
  the symbol check, in place of `zugojson`'s own `goose.yaml` (R5, R6).
- **`zudiff`**
  ([RFC 0045](https://github.com/pedrobtz/packages/blob/main/rfcs/not-started/0045-zudiff-sequence-diff-in-goose.md)):
  the planned second user, and the first that is a base-layer package
  rather than a demonstration. It needs borrowed `const i32[:]` and
  `const u8[:]` parameters, `raw` results and interrupts, and starts after
  v0.1.0. It packs a vector result into bytes, which makes it the first
  consumer of §14's vector results.
- **Any later Goose package**: built with `use_zugoose()` from its first
  commit.

## 4. Architecture

Three parts, each with a different lifetime.

1. **The tool** (`zugoose`, an R package, used on the author's machine and in
   CI). It reads the pin, fetches and builds that Goose commit into a
   cache, compiles the package's Goose, and writes the generated files.
   It never runs when a user installs the package.
2. **The generated files** (in the user's `src/` and `R/`, committed).
   `<pkg>_goose.c` and `.h` from the compiler; `goose_version.h` with the
   runtime interface hash; `goose_vendor.c` with the pieces of Goose's
   `runtime_impl.h` the generated C calls; `<pkg>_zugoose.c` with the
   `.Call` entry points and `R_init_<pkg>()`; `R/zugoose-wrappers.R` with the
   R functions; `tools/goose/PROVENANCE` with checksums.
3. **The runtime** (`zugoose_runtime.c` and `.h`, copied into the user's
   `src/` by the tool, D5). It replaces Goose's stock runtime inside an R
   process (§6).

The flow, for `zugoose_update()`:

```text
tools/goose/*.goose ──goose -o --header──▶ <pkg>_goose.{c,h}
        │                                       │
  export fn lines                        C signatures
        └──────────────┬────────────────────────┘
                       │
             ┌────────────────┼────────────────────┐
             ▼                ▼                    ▼
     <pkg>_zugoose.c     R/zugoose-wrappers.R    goose_version.h,
     (.Call, init)    (R functions)        goose_vendor.c, PROVENANCE
```

The compiler runs without `--standalone`, which would build the stock
runtime into the C; `zugoose`'s runtime is linked instead, as in
`zugojson`.

The glue takes the C signature of each export from the header the
compiler writes, and the parameter names, `bool` versus `u8`, and the
`// zugoose:` annotations from the `export fn` line in the Goose source
(D4). At the pin the header names parameters `gs_export_arg0_2`, writes
`bool` and `u8` alike as `uint8_t`, and carries no comments, so neither
source is enough alone. The header is the authority on types: the
generator matches the two by function name, and stops if they disagree.
A header that carried names and types would let the source scan go
(§6.3).

## 5. Public R API (v0.1.0 shape)

```r
# set-up and generation, on the author's machine
use_zugoose(path = ".", pin = zugoose_default_pin())
zugoose_update(path = ".", debug = FALSE)
zugoose_verify(path = ".")

# the compiler
goose_fetch(pin = NULL)        # clone and build the pinned commit into the cache
goose_path(pin = NULL)         # where that build lives

zugoose_info()
```

`zugoose_update(debug = TRUE)` compiles with Goose's `--debug`, which checks
signed overflow and narrowing casts and names the source line. It is for
development. `zugoose_verify()` refuses a debug build.

The R functions the generated wrapper file contains are named after the
exported Goose functions, without a prefix the author did not write. A
wrapper checks argument types and lengths, converts to UTF-8 where §7
says so, calls the entry point and turns a failure into a condition (§8).
An author who wants a different R surface keeps the generated functions
internal and writes their own on top.

## 6. The runtime

### 6.1 What it keeps from `zugojson`

Goose's stock runtime ends the process on any failure, writes to
`stdout` and `stderr`, and installs signal handlers for the life of the
process. None of that is allowed inside R. `zugojson`'s runtime replaces
it, and `zugoose` keeps its design:

- Every Goose failure (bounds check, `assert`, `abort`, division by zero,
  overflow in a debug build) jumps back to `goose_r_run()`.
  `goose_r_run()` rewinds every data stack to where it stood before the
  call and returns the message. The entry point raises the R condition
  after that, never inside.
- Each data stack is reserved with a `PROT_NONE` gap after it. Generated
  code has no capacity checks, so an overflow faults in the gap, and the
  fault handler turns it into a failure.
- The fault handlers are installed only while Goose code runs, and a
  fault outside the data stacks is handed back to R's handler.
- The start-up function is named after the runtime interface hash, so C
  from another compiler version fails to link instead of misbehaving.

### 6.2 What it adds

Each item is something `zugojson` lacks.

- **Windows.** `zugojson`'s runtime stops the build with `#error`. `zugoose`
  writes the Windows half: `VirtualAlloc` with `PAGE_NOACCESS` for the
  gap, and a vectored exception handler for the fault.
- **One set-up per call, not per element.** `zugojson`'s `json_valid()`
  calls `goose_r_run()` once for each string, so it saves the stack tops
  and swaps two signal handlers in and out for every element. The
  generated glue installs them once per `.Call` and runs the loop over
  the vector inside (§7). The saving is not measured yet; stage 0 records
  it.
- **Ctrl-C.** A long Goose loop cannot be interrupted today. The runtime
  installs its own `SIGINT` handler (a console control handler on Windows)
  for the duration of the call, which sets a flag. Goose code polls it
  with `zugoose::interrupted()` (§8), which fails the call with an
  interrupt that the entry point re-raises in R. Polling is cooperative:
  code that never polls cannot be interrupted, as with
  `R_CheckUserInterrupt()` in C.
- **Memory given back.** Data stacks are reserved as address space and
  committed by the kernel as they are touched, so a call that touched a
  gigabyte keeps that gigabyte resident until the process ends. After
  each call the runtime releases the pages above each stack's top
  (`madvise(MADV_DONTNEED)`, `VirtualFree` with `MEM_DECOMMIT` on
  Windows). Whether a grow-only global's capacity sits above or below
  its stack's top after `.clear()` decides how much this frees, and is to
  be checked in Goose's runtime before stage 3.
- **Output.** Goose's `print` reaches `gs_out_bytes`, which `zugojson`
  does not route anywhere. The runtime buffers it during the call, and the
  entry point writes it with `Rprintf()` after the call returns, since R's
  API cannot be called inside.
- **Classed errors.** `zugojson` returns `c(kind, text)` and marks a
  syntax error by prefixing the message with `!`. `zugoose` replaces both
  conventions with `zugoose::fail()` (§8): a failure carries a class name, a
  message and an optional integer, and the entry point builds the
  condition from them.
- **Clear limits.** The runtime refuses more data stacks than it can
  track, and reports the stack size it was built with in `zugoose_info()`
  and in the overflow message.

### 6.3 Toward the stock runtime

Most of `zugoose_runtime.c` exists because Goose's runtime has no way for an
embedding host to take over failures, output and memory. A small hook
table upstream would let the stock runtime run inside R:

```c
typedef struct {
    void (*fail)(const char *msg);                 /* must not return */
    void (*out)(const uint8_t *p, int64_t len);
    uint8_t *(*reserve)(gs_stack *stack);
} gs_host;
void gs_set_host(const gs_host *host);
```

With it, `goose_vendor.c` and its anchor-line copying would go, and so
would most of §6.1. `zugoose` would keep only the hooks, the signal handling
and the stack rewind. The same proposal can carry two small fixes that
`zugojson` patches locally: `__extension__` on the `__int128` typedef,
which `-pedantic` reports as ISO C; the one-ulp error in `std`'s
`parse_flt`; and parameter names and `bool` in the `--header` output, so
the generator need not read Goose source (D4). Q1 asks whether to pursue
it.

## 7. Types across the boundary

The generator reads each exported function's C signature from the header
and maps it.

| Goose parameter | C in the header | R argument |
| --- | --- | --- |
| `i32`, `i64` | `int32_t`, `int64_t` | integer or double scalar, checked |
| `f64` | `double` | double scalar |
| `bool` | `uint8_t` | logical scalar, not `NA` |
| `u8[:]` text | `sl_u8` | string scalar, converted to UTF-8 |
| `const f64[:]` | `sl_f64` | double vector, borrowed, not copied |
| `const i32[:]` | `sl_i32` | integer vector, borrowed |
| `const u8[:]` bytes | `sl_u8` | raw vector, borrowed |

| Goose result | R value |
| --- | --- |
| `i64`, `f64`, `bool` | the matching scalar |
| `const u8[:]` | a string, marked UTF-8, or a raw vector (`// zugoose: raw`) |

Text and raw both arrive as `sl_u8`, so the author says which in a
comment the generator reads: `// zugoose: raw`. A result is copied into an
R object after `goose_r_run()` returns, because Goose's memory is reused
by the next call.

At the pin an export returns a scalar or a `const u8[:]` and nothing else
(Goose spec §7.11). Numeric vector results wait for Goose to allow them,
and §14 lists them.

**Vectorised calls.** A parameter marked `// zugoose: each` takes a
character vector, and the generated entry point calls the Goose function
once per element inside one `goose_r_run()` (§6.2), returning a vector of
results, with `NA` in and `NA` out. In v0.1.0 an `each` function returns a
scalar: the entry point allocates the result vector before the run and
the loop writes into its memory, which calls no R function. A `const
u8[:]` result per element would need buffering in C and is §14. A
`zugoose::fail()` or runtime failure on any element fails the whole call, as
a vectorised R function that errors does; an element's own verdict is the
function's result. `zugojson`'s `json_valid()` is this: its check export
returns `bool`, and the loop fills a logical vector.

## 8. Errors and the `zugoose` Goose module

`import zugoose;` gives Goose code three functions, all `extern fn` into the
runtime:

- `zugoose::fail(class: u8[:], msg: u8[:])` and a form with an integer
  field. The call ends at once, the stacks are rewound, and R sees a
  condition of class `<pkg>_<class>`, then `<pkg>_error`, then
  `zugoose_error`.
- `zugoose::interrupted() -> bool` (§6.2), which fails the call with an
  interrupt.
- `zugoose::message(text: u8[:])`, the same buffer as `print`.

A Goose runtime failure that the author did not ask for (a bounds
check, a stack overflow) is `<pkg>_runtime_error`. Bad arguments caught by
the R wrapper are `<pkg>_input_error`. These are `zugojson`'s three
classes, generalised.

## 9. Testing

- **The runtime alone.** A test package under `tests/testthat/fixtures`
  built with `zugoose` exercises each failure: an index out of bounds, an
  `assert`, a division by zero, `zugoose::fail()`, a data stack overflow
  with the stack shrunk at compile time, an interrupt raised from a
  second process, and memory returned after a large call. Each failure is
  followed by a successful call, which proves the stacks were rewound.
- **The generator.** Snapshot tests of the glue and wrappers for headers
  covering every row of §7.
- **`zugojson` moved to `zugoose`.** Its hand-written glue, runtime and
  scripts are deleted. Its `.goose` file changes in two places:
  `zugojson_format` reports syntax errors with `zugoose::fail()` instead of
  a `!` prefix, and `zugojson_check` returns `bool` for `each` (§7). Its
  test suite passes unchanged.
- **CRAN gates.** The symbol check from `zugojson` and `R CMD check
  --as-cran` with `-pedantic` on every r-actions leg, Windows included.

## 10. Performance targets

- **Call overhead.** Stage 0 measures `zugojson`'s cost of an empty call
  and of `json_valid()` over 10⁵ short strings. `zugoose`'s glue is no
  slower on either, and the per-vector set-up of §6.2 should make the
  second faster.
- **No change inside Goose.** The generated Goose C is the compiler's;
  `zugojson`'s benchmarks from its README stay within noise.

## 11. Roadmap

Stages run the quick CI profile unless marked `full`. Each is one pull
request, merged on green.

| Stage | Delivers | Exit criterion |
| --- | --- | --- |
| 0 | repository, CI, `goose_fetch()`, baseline | §10 recorded |
| 1 | set-up, scalars, POSIX runtime of §6.1 | fixture builds, rewinds |
| 2 | §7 vectors and `each`, the `zugoose` module | §8; snapshots |
| 3 | §6.2 interrupts, output, memory, one set-up per call | §9 runtime tests |
| 4 | Windows runtime | §9 on the Windows leg |
| 5 | `zugoose_verify()`, shared gates, `zugojson` moved | §9 passes |
| 6 (`full`) | README, article, release preparation | §12 |

## 12. Acceptance criteria for v0.1.0

- Every behaviour in §6, §7 and §8 has a passing test.
- `zugojson` builds with `zugoose`'s output and none of its own glue, runtime
  or scripts, and passes its tests on every r-actions leg, Windows
  included.
- `zugoose_verify()` passes in `zugojson`'s CI against the pin.
- After any failure, the next call succeeds and the data stacks are where
  they were.
- `R CMD check --as-cran` is clean for `zugoose` and for the fixture package;
  no `Imports`.
- Seven stages. A stage that needs a second pull request moves scope to
  §14, not the release.

## 13. Decisions

Marked **(M)** when the maintainer decided it in the conversation and
**(C)** when this RFC decides it and the maintainer may overturn it.

- **D1 (M).** `zugoose` is to Goose what `savvy` is to Rust: write `export
  fn`, get the C and R glue generated.
- **D2 (M).** The runtime is the centre of the package, and §6.2 lists
  the improvements it makes over `zugojson`'s.
- **D3 (C).** Packages ship the generated C, and Goose runs only on the
  author's machine and in CI. `savvy` needs a Rust toolchain at install
  time; CRAN machines have no Goose compiler, and building one at install
  would mean a C++ compiler, `cmake` and a download. Q2 keeps an opt-in
  install-time path open.
- **D4 (C).** The glue takes C types from the compiler's header and
  names, `bool` and annotations from the `export fn` lines (§4), and
  stops if the two disagree.
- **D5 (C).** The runtime is copied into each package by
  `zugoose_update()`, not reached through `LinkingTo`, which keeps `zugoose` a
  development-only dependency.
- **D5a (C).** Each `zugoose` release supports one Goose pin, recorded in
  `zugoose_default_pin()`, and carries the runtime for that pin's interface
  hash. `use_zugoose(pin =)` accepts another commit only if its hash is the
  same; otherwise `zugoose_update()` stops and names the `zugoose` version to
  use. A package bumps Goose by bumping `zugoose`.
- **D6 (M).** The name is `zugoose`: `zu` for the family and the
  language it serves, said in full.
- **D7 (C).** The runtime's symbols keep Goose's `gs_` names, which the
  generated C calls by name, and are hidden from the shared object (R3).
  Glue symbols take the `zugoose_` and package prefixes.
- **D8 (C).** The `zugoose` Goose module replaces in-band error conventions
  such as `zugojson`'s `!` prefix.
- **D9 (M).** v0.1.0 ships its own runtime, and the upstream hook
  proposal of §6.3 runs in parallel without blocking it.
- **D10 (M).** Windows blocks v0.1.0: stage 4 does not move to §14.
- **D11 (M, 2026-10-25).** The pin of D5a is a Goose release tag, never
  an untagged commit, recorded as the tag and its commit SHA so a moved
  tag is caught. `zugoose_default_pin()` is the newest release when the
  `zugoose` version is cut. An untagged commit is accepted only for
  development, with a warning, and never in a `zugoose` release.
- **D12 (M, 2026-10-25).** Everything `zugoose` takes from Goose is
  reproducible from the pin: a pristine copy of `runtime.h` and
  `runtime_impl.h` at the pin with checksums; each range copied into
  `goose_vendor.c` checksummed against that copy; local changes as patch
  files in `tools/patches/`, verified as "upstream plus exactly these
  patches"; the `gs_` symbols the generated C needs checked against those
  the runtime defines; byte-identical output on Linux and macOS. A
  scheduled job runs the pipeline against Goose's newest release and
  `main`, and opens an issue on the first thing that breaks.

## 14. Later versions, in the order they are expected

- **Vector results**: numeric slices returned from an export, once Goose
  allows them, and `const u8[:]` results from `each` functions. `zudiff`
  is waiting on the first, to drop its packed edit script.
- **Lists and data frames** across the boundary, as Goose structs and
  arrays of structs.
- **Contained workers**: Goose threads that never touch R, with the
  runtime's queues.
- **The stock runtime**, once §6.3's hooks exist upstream.
- **Install-time compilation**, opt-in, if Q2 says yes.
- **Running Goose on its own C stack**, so that deep recursion overflows
  into a guard the runtime catches instead of R's C stack. `zugojson`
  avoids this today with a nesting limit in its parser.

## 15. Open questions

- **Q1.** Should the host hooks of §6.3 be proposed to Goose upstream,
  or should `zugoose` keep its own runtime for good? This RFC proposes them
  and does not wait for them (D9).
- **Q2.** Should a package be able to opt in to compiling Goose at
  install time, as `savvy` does with Rust? This RFC says not in v0.1.0.
- **Q3.** Copy or `LinkingTo` for the runtime? This RFC copies (D5).
- **Q4.** Is v0.1.0 a CRAN submission or a GitHub release? A
  development-only tool is unusual on CRAN; the board's `CRAN intent`
  field records the answer. This RFC assumes `Parked`.

## 16. What this RFC does not decide

The cache location for fetched compilers, the exact comment syntax for
`raw` and `each`, the default data stack size, the wording of generated
messages, and whether the wrapper file is one file or one per `.goose`
source.
