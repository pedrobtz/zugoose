# zugoose: roadmap

Status: draft 2026-10-25. Implements [design.md](design.md).
Target: v0.1.0, the first CRAN release (Q4 settled 2026-10-25: CRAN).
Tracking: [v0.1.0 (#1)](https://github.com/pedrobtz/zugoose/issues/1), with
one sub-issue per stage;
[CRAN-submission (#11)](https://github.com/pedrobtz/zugoose/issues/11).

The design's §11 has seven stages. This roadmap keeps its order and
splits two of them to fit the portfolio's brackets. Design stage 0 becomes
Stage 0 (identity and CI) and Stage 1 (the compiler and the baseline).
Design stage 6 becomes Stage 7 (`Full CI`) and Stage 8 (readiness, with the
README and the article). Each stage is still one pull request, as §12
requires; scope that needs a second one moves to "After 0.1.0".

## Stages

| Stage | Title | CI | Issue | Pull request | Done |
| --- | --- | --- | --- | --- | --- |
| 0 | [Package identity](#stage-0--package-identity) | quick | [#2](https://github.com/pedrobtz/zugoose/issues/2) | | |
| 1 | [Goose compiler and baseline](#stage-1--goose-compiler-and-baseline) | quick | [#3](https://github.com/pedrobtz/zugoose/issues/3) | | |
| 2 | [Set-up, scalars and the POSIX runtime](#stage-2--set-up-scalars-and-the-posix-runtime) | full | [#4](https://github.com/pedrobtz/zugoose/issues/4) | | |
| 3 | [Vectors, each and the zugoose module](#stage-3--vectors-each-and-the-zugoose-module) | quick | [#5](https://github.com/pedrobtz/zugoose/issues/5) | | |
| 4 | [Interrupts, output and memory](#stage-4--interrupts-output-and-memory) | quick | [#6](https://github.com/pedrobtz/zugoose/issues/6) | | |
| 5 | [Windows runtime](#stage-5--windows-runtime) | full | [#7](https://github.com/pedrobtz/zugoose/issues/7) | | |
| 6 | [Verification, shared gates and zugojson](#stage-6--verification-shared-gates-and-zugojson) | quick | [#8](https://github.com/pedrobtz/zugoose/issues/8) | | |
| 7 | [Full CI](#stage-7--full-ci) | full | [#9](https://github.com/pedrobtz/zugoose/issues/9) | | |
| 8 | [v0.1.0 readiness](#stage-8--v010-readiness) | full | [#10](https://github.com/pedrobtz/zugoose/issues/10) | | |

## Stage 0 — Package identity

**Issue.** [#2](https://github.com/pedrobtz/zugoose/issues/2), sub-issue of
[v0.1.0](https://github.com/pedrobtz/zugoose/issues/1).

**CI.** `quick`.

**Goal.** The scaffold says what the package is, and CI checks it quickly
on the platforms later stages need.

**Work.**

- `DESCRIPTION`: real Title and Description, `Authors@R` with `aut`, `cre`
  and `cph`, `Depends: R (>= 4.1)`, `Language: en-GB` (design §3).
- `R-CMD-check.yaml`: add the Windows R-devel runner row.
- Callers for the r-actions checks a compiled package needs in the full
  profile: `gctorture.yml`, `valgrind.yml`, `sanitizers.yml`, `rchk.yml`.
- Pin `coverage.yml` to a commit with the tag in a comment, since it holds
  a write token.
- Replace the placeholder test with one for `zugoose_info()` returning the
  package version.
- `CLAUDE.md` from the template and `.Rbuildignore` with the entries its
  "Working with skills" lists (done by `pkg-init`).

**Exit criteria.**

- `R CMD check` passes on the quick r-actions profile.
- Every check of the pull request's quick run finishes in under five
  minutes. Anything tuned to get there is recorded here.
- `DESCRIPTION` carries no scaffold placeholder text.

**Design.** §3.

## Stage 1 — Goose compiler and baseline

**Issue.** [#3](https://github.com/pedrobtz/zugoose/issues/3), sub-issue of
[v0.1.0](https://github.com/pedrobtz/zugoose/issues/1).

**CI.** `quick`.

**Goal.** `zugoose` can fetch and build the pinned Goose compiler on the
author's machine, and the cost of today's hand-built glue is measured.

**Work.**

- `zugoose_default_pin()`, `goose_fetch()` and `goose_path()`: clone the
  pinned commit, build it into a user cache, and report where it is. Fail
  with a classed `zugoose_error` when `git`, a C++ compiler or `cmake` is
  missing.
- Choose and record the cache location (design §16).
- `zugoose_info()` reports the pin and whether a compiler is cached.
- Tests that need the network or a toolchain skip on CRAN and when the
  tools are absent; one CI leg runs them.
- Measure `zugojson`'s empty-call cost and `json_valid()` over 10⁵ short
  strings, and record the numbers and the method in
  `.agents/baseline.md`.

**Exit criteria.**

- On the CI leg that has the tools, `goose_fetch()` builds the pin and
  `goose_path()` points at a working compiler.
- Without the tools, those tests skip and `R CMD check` passes on the quick
  profile.
- `.agents/baseline.md` holds the §10 measurements.

**Design.** §5 (the compiler), §10, §16.

## Stage 2 — Set-up, scalars and the POSIX runtime

**Issue.** [#4](https://github.com/pedrobtz/zugoose/issues/4), sub-issue of
[v0.1.0](https://github.com/pedrobtz/zugoose/issues/1).

**CI.** `full — first runtime code: longjmp out of Goose, PROT_NONE stack
gaps and fault handlers need gctorture, valgrind and ASan before anything
builds on them.`

**Goal.** A fixture package made with `use_zugoose()` and
`zugoose_update()` calls Goose functions on scalars, and any Goose failure
arrives as an R error with the stacks rewound.

**Work.**

- `use_zugoose()`: folders, pin, build files and a starting `.goose` file.
- `zugoose_update()`: compile, then generate the entry points, the
  registration and the R wrappers for the scalar rows of §7, matching the
  header against the `export fn` lines and stopping on disagreement (D4).
- The runtime of §6.1 for POSIX, copied into the package (D5), with hidden
  `gs_` symbols (D7) and the interface-hash start-up name; `goose_vendor.c`
  and `PROVENANCE`.
- A fixture package under `tests/testthat/fixtures` with its generated C
  committed, so the tests build it without Goose.
- Settle the default data stack size and the wrapper file layout
  (design §16).

**Exit criteria.**

- The fixture builds and its scalar functions return the right values.
- An index out of bounds, an `assert`, a division by zero and a data stack
  overflow each give `<pkg>_runtime_error`, and the next call succeeds.
- A pin with another interface hash stops `zugoose_update()`.
- The full r-actions profile is green on the pull request.

**Design.** §4, §5, §6.1, §7 (scalars).

## Stage 3 — Vectors, each and the zugoose module

**Issue.** [#5](https://github.com/pedrobtz/zugoose/issues/5), sub-issue of
[v0.1.0](https://github.com/pedrobtz/zugoose/issues/1).

**CI.** `quick`.

**Goal.** Every row of §7 crosses the boundary, `each` functions loop
inside one run, and Goose code raises classed errors.

**Work.**

- The vector rows of §7: borrowed `f64`, `i32` and raw slices, UTF-8 text,
  string and raw results (`// zugoose: raw`).
- `// zugoose: each`: one run per call, `NA` in and `NA` out, the result
  vector allocated before the run.
- Settle the comment syntax for `raw` and `each` (design §16).
- The `zugoose` Goose module: `fail()` in both forms and `message()`;
  conditions classed `<pkg>_<class>`, `<pkg>_error`, `zugoose_error`.
- `<pkg>_input_error` from the R wrappers.
- Snapshot tests of the generated glue and wrappers for every §7 row.

**Exit criteria.**

- The fixture exercises every §7 row and every §8 class, each followed by
  a successful call.
- Snapshot tests cover every §7 row.
- `R CMD check` passes on the quick profile.

**Design.** §7, §8.

## Stage 4 — Interrupts, output and memory

**Issue.** [#6](https://github.com/pedrobtz/zugoose/issues/6), sub-issue of
[v0.1.0](https://github.com/pedrobtz/zugoose/issues/1).

**CI.** `quick`.

**Goal.** The POSIX runtime has every §6.2 improvement except Windows.

**Work.**

- Signal handlers installed once per `.Call`, not per element.
- `SIGINT` sets a flag during the call; `zugoose::interrupted()` polls it
  and the entry point re-raises an R interrupt.
- `print` and `zugoose::message()` buffered and written with `Rprintf()`
  after the run.
- Pages above each stack's top released after the call
  (`madvise(MADV_DONTNEED)`), after checking where grow-only globals keep
  their capacity (§6.2).
- The runtime refuses too many data stacks and reports its stack size in
  `zugoose_info()` and the overflow message.
- Re-measure the §10 cases against `.agents/baseline.md`.

**Exit criteria.**

- §9's runtime tests pass on POSIX: an interrupt raised from a second
  process ends the call with an R interrupt and the next call succeeds;
  memory is returned after a large call.
- Output reaches the R console in order.
- The §10 measurements are no slower than the baseline, recorded in
  `.agents/baseline.md`.
- `R CMD check` passes on the quick profile.

**Design.** §6.2, §9, §10.

## Stage 5 — Windows runtime

**Issue.** [#7](https://github.com/pedrobtz/zugoose/issues/7), sub-issue of
[v0.1.0](https://github.com/pedrobtz/zugoose/issues/1).

**CI.** `full — Windows-only code path: VirtualAlloc guard pages, a
vectored exception handler and a console control handler run only on the
Windows legs.`

**Goal.** The runtime works on Windows with the same behaviour as POSIX
(D10).

**Work.**

- `VirtualAlloc` with `PAGE_NOACCESS` gaps, a vectored exception handler
  for the fault, a console control handler for Ctrl-C, `VirtualFree` with
  `MEM_DECOMMIT` to give memory back.
- Remove the `#error` that `zugojson`'s runtime has.

**Exit criteria.**

- §9's runtime tests pass on the Windows release and R-devel legs.
- The full r-actions profile is green on the pull request.

**Design.** §6.2 (Windows), D10.

## Stage 6 — Verification, shared gates and zugojson

**Issue.** [#8](https://github.com/pedrobtz/zugoose/issues/8), sub-issue of
[v0.1.0](https://github.com/pedrobtz/zugoose/issues/1).

**CI.** `quick`. The `zugojson` pull request runs its own full profile.

**Goal.** `zugojson` is built from `zugoose`'s output alone, and CI checks
that the generated files are current.

**Work.**

- `zugoose_verify()`: regenerate into a temporary folder, fail on any
  difference, refuse a debug build.
- The symbol check and warning patches from `zugojson`, shared.
- An r-actions step that runs `zugoose_verify()` and the symbol check
  (a pull request to `pedrobtz/r-actions`).
- A pull request to `pedrobtz/zugojson`: delete its glue, runtime and
  scripts, switch its two Goose changes to `zugoose::fail()` and `each`
  (§9), and replace `goose.yaml` with the r-actions step.

**Exit criteria.**

- `zugoose_verify()` passes on the fixture and fails on a hand-edited
  generated file.
- The `zugojson` pull request passes its unchanged test suite on every
  r-actions leg, Windows included, and `zugoose_verify()` passes in its CI.
- `R CMD check` passes on the quick profile.

**Design.** §5 (`zugoose_verify()`), §9, §12.

## Stage 7 — Full CI

**Issue.** [#9](https://github.com/pedrobtz/zugoose/issues/9), sub-issue of
[v0.1.0](https://github.com/pedrobtz/zugoose/issues/1).

**CI.** `full`.

**Goal.** No new features. The full r-actions profile is green on the
default branch's code.

**Work.**

- Run the full profile (`full-ci` label) and fix everything it finds:
  gctorture at the sensitive step, valgrind, ASan, rchk, every platform
  leg, `nosuggests`, `nold`, cran-special.
- Fix the failures recorded as comments on this issue by earlier merges.

**Exit criteria.**

- A full-profile run is green on the pull request and on its merge commit.

**Design.** §9 (CRAN gates), §12.

## Stage 8 — v0.1.0 readiness

**Issue.** [#10](https://github.com/pedrobtz/zugoose/issues/10), sub-issue of
[v0.1.0](https://github.com/pedrobtz/zugoose/issues/1).

**CI.** `full`.

**Goal.** Only the submission itself is left.

**Work.**

- `README.Rmd` and the article "Moving `zugojson` to `zugoose`".
- `Version: 0.1.0` and the matching `NEWS.md` heading.
- `cran-comments.md` listing the r-actions CI legs as test environments.
- The `pkg-critical-review` skill run, with its blocking and required
  findings fixed; then the `pkg-cran-review` skill run, with every finding
  resolved.

**Exit criteria.**

- `urlchecker::url_check()` and `spelling::spell_check_package()` are
  clean.
- `R CMD check --as-cran` is clean for `zugoose` and the fixture package,
  with no `Imports`.
- Every §12 criterion holds.
- A full-profile r-actions run is green on the release commit.

**Design.** §2 (documentation), §12.

## After 0.1.0

From design §14, in the expected order:

- Vector results, and `const u8[:]` results from `each` functions: Goose
  does not allow numeric slice results at the pin.
- Lists and data frames across the boundary: no package needs them yet.
- Contained Goose workers: v0.1.0 starts no threads.
- The stock Goose runtime, once the §6.3 host hooks exist upstream (D9).
- Opt-in install-time compilation, if Q2 says yes.
- Running Goose on its own C stack, for deep recursion.

## Open questions

- **D3, generated C shipped** (marked for a second look in the RFC).
  Overturning it reshapes Stage 2 onwards; settle it before Stage 2.
- **Q3, copy or `LinkingTo` for the runtime.** Decides how Stage 2 delivers
  the runtime; the design copies (D5).
- **Q4, CRAN or GitHub release.** Settled 2026-10-25: v0.1.0 goes to
  CRAN, and the board's `CRAN intent` is `Planned`. Record it in
  `design.md` §15 in Stage 0.
- **Q1, upstream host hooks.** Does not block a stage (D9).
- **ASan and the fault handlers.** ASan installs its own `SIGSEGV` handler,
  which may clash with the runtime's stack-gap handler. Stage 2 finds out;
  the fix may be an ASan option in the caller workflow.
