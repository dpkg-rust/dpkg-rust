# 06 — Rewriting dpkg's C code in Rust: possibilities, benefits, risks

This page assesses whether and how the C parts of dpkg could be rewritten in Rust. It
builds on the descriptions in [02](02-libdpkg.md), [03](03-dpkg-programs.md) and
[05](05-build-test-release.md). The Perl side is covered separately in
[07](07-perl-replacement-analysis.md).

Facts come from the code and its history and are referenced. Assessments of difficulty,
value and effort are judgement, and are marked as such.

## 1. Conclusions

1. **A Rust dpkg is feasible, but not as an in-place conversion.** libdpkg's error model
   (`longjmp` with a global cleanup stack) and its data model (a global, cyclic,
   arena-allocated graph that callers access field by field) leave no narrow interface
   behind which Rust could gradually replace C.
2. **The workable route is program by program.** The programs already cooperate only
   through argument vectors, pipes, exit codes and files on disk. A Rust binary with the
   same name and behaviour can replace a C one, and the existing black-box test suites
   exercise it unchanged because they find programs through `PATH`. This repository has
   migrated five programs between languages that way already.
3. **Memory safety is a real benefit but not the largest one.** dpkg's history shows 14
   CVEs, of which one was a C memory-safety bug and ten were logic bugs in Perl. The
   stronger arguments are: hardened parsing of untrusted `.deb` files, an error and
   rollback design that can be unit-tested, and a single implementation of logic that
   exists twice today (in C and in Perl) and disagrees on edge cases.
4. **The largest risks are not about writing code.** They are behavioural compatibility in
   the install engine, platforms where Rust does not exist, dpkg's position at the bottom
   of the bootstrap chain, preserving 44 translations, and keeping up with an upstream
   that one maintainer changes about 400 to 570 times a year.
5. **Most of the early work pays off either way.** A differential test harness, fuzzers for
   the parsers and unit tests for the database layer are prerequisites for a rewrite and
   improve the C code if no rewrite happens.
6. **Recommended order:** tooling first; then the pure-logic crates; then leaf programs and
   `dpkg-deb`; the install engine last, behind a long validation period. Several decisions
   belong to the project owner before any of it starts (section 10).

## 2. What would be rewritten

Sizes are lines of code without comments and blanks unless noted. Difficulty: S = days to
a couple of weeks, M = weeks, L = a month or more, XL = several months including design.
Difficulty and value are judgement.

| Component | Size | Difficulty | Safety value | Existing tests | Notes |
|---|---|---|---|---|---|
| Version, architecture list, `ar`, `deb-version`, strings | ~2,500 | S | low to medium | good unit tests | Pure functions |
| deb822 parser and writer | 2,133 | M | medium | none at unit level | Tokenizer is easy; field semantics and legacy repairs are not |
| tar reader, compression glue, copy loops | ~1,900 | M | **high** (parses untrusted input) | partial | Compression must keep calling the same C libraries (section 5, R6) |
| Database: status journal, `info/`, file lists, diversions, statoverride, triggers, locks | ~3,400 | M to L | medium | none at unit level | Formats are simple; protocols are exact |
| `dpkg-realpath` | 193 | S | low | 26 checks | |
| `dpkg-trigger` | 230 | S | low | 1 group | |
| `dpkg-statoverride` | 470 | S | low | option tests only | |
| `dpkg-split` | 1,094 | S to M | medium | 2 groups | |
| `dpkg-divert` | 715 | M | low | 29 groups | Needs the full database read path |
| `dpkg-query` | 830 | M | low | option tests only | Output formats are a hard contract |
| **`dpkg-deb`** | 1,671 | M | **highest** | 11 groups | The documented boundary for untrusted packages; needs no database |
| `update-alternatives` | 3,369 lines | S to M | low | 712 assertions | Does not use libdpkg |
| `start-stop-daemon` | 3,148 lines | M (Linux only) to L (all systems) | low | **none** | Does not use libdpkg; per-OS process inspection |
| `dpkg`, read-only actions | ~2,000 | S to M | low | few | |
| **`dpkg`, install engine** | ~6,650 | **XL** | high (runs as root on every system) | 89 functional scenarios | Rollback, ordering, durability |
| `dselect` | 7,134 lines C++ | out of scope | — | none | Uses libdpkg's structs directly |

Total in scope: roughly 35,000 lines of C.

## 3. What a rewrite could deliver

Each benefit is listed with the evidence for it and with what limits it.

### 3.1 Memory safety

**Evidence.** In ten years, 70 of 4,301 commits fixed memory-safety-type bugs; 9 of those
were user-facing crashes, overflows or use-after-free. One CVE was a memory-safety bug
(CVE-2015-0860, an off-by-one write in `dpkg-deb` found by fuzzing) and one was a
format-string injection (CVE-2014-8625). This study found two more memory errors with
AddressSanitizer and one NULL dereference ([08](08-findings.md), S6, C5, C6).

**Limits.** The rate is low for a C code base of this age, and the maintainer already uses
sanitizers and static analysers. dpkg's stated position is that installing an untrusted
package is unsafe by design, since its scripts run as root. That leaves *examining*
packages (`dpkg-deb`, and the tar reader inside `dpkg`) as the exposed surface. Some
`unsafe` code will remain for system calls and library bindings.

**Assessment.** Real, concentrated in the archive-reading code, moderate overall.

### 3.2 Bug classes that disappear

Format-string injection, NULL dereferences (several of the nine user-facing bugs),
use-after-free (two fixes), buffers returned from static storage, and undefined behaviour
on integer overflow have no equivalent in safe Rust.

### 3.3 An error and rollback model that can be tested

Today a failure anywhere unwinds through `longjmp` to one of three recovery points, running
a global stack of cleanup entries with flag masks, checkpoints and gating counters
([03, section 5](03-dpkg-programs.md#5-error-unwinding-in-the-install-path)). None of this
has unit tests. In Rust the same semantics become an explicit data structure (section 7)
that can be driven with injected failures, and errors become return values that the
compiler forces callers to handle.

### 3.4 One implementation instead of two

Version comparison, deb822 parsing, dependency syntax, package-name rules and the field
registry exist in both C and Perl and disagree in at least ten ways, most of them
confirmed by running both
([08, section 3](08-findings.md#3-where-the-c-and-perl-implementations-disagree)). A Rust
core library could serve the installer and, in time, the build tools. This benefit only
materialises if the Perl side moves too.

### 3.5 Structural defence against path traversal

Eight of the fourteen CVEs are path traversal. dpkg builds every path by concatenating the
installation root and a name, and this study found that `dpkg --root` accepts `..` in
archive member names (S3). A rewrite can route all access to the installation root through
directory file descriptors (`openat`, `renameat`, `O_NOFOLLOW`, or `openat2` with
`RESOLVE_BENEATH` on Linux), so that escaping the root is impossible by construction rather
than by checking strings. This is a design benefit that a rewrite makes affordable; the
language does not provide it by itself, and it must respect dpkg's deliberate following of
existing symlinked directories.

### 3.6 Testability and fuzzing

The deb822 parser, the status journal, triggers and compression have no unit tests, and the
tree has no fuzz harness. Rust's tooling makes unit tests, property tests and coverage-guided
fuzzers cheap to add and to keep running.

### 3.7 Performance

Little on the C side. C start-up is already 0.45 ms and installation time is dominated by
fsync and by maintainer scripts. Possible gains: reading archives in-process removes three
process launches per installed package (`dpkg-split`, `dpkg-deb` twice), and file lists
could be loaded in parallel. Neither was measured.

### 3.8 Fewer external dependencies

MD5 could come from a Rust crate instead of libmd. `dpkg-deb` could write tar archives
itself instead of running GNU tar, provided the output stays byte-identical.

### 3.9 Contributors and ecosystem

One person writes 97 % of the non-translation commits. A Rust code base might attract more
contributors, as uutils did, but that is speculation, and a fork does not inherit the
maintainer's knowledge. APT's maintainers have announced Rust for exactly the `.deb`, `ar`
and `tar` parsing code, which makes shared format crates a plausible point of cooperation.

## 4. What stands in the way in the code

| # | Obstacle | Why it matters |
|---|---|---|
| O1 | **`longjmp` error handling.** About 700 raising call sites in `lib/dpkg/` and `src/` (more counting usage errors); every allocator can raise. | Jumping over Rust stack frames that own values is undefined behaviour, and Rust cannot host a `setjmp` site. Mixed C and Rust call stacks need a C shim at every crossing, in both directions. |
| O2 | **The data model is the API.** Global hash tables, one arena never freed piecewise, cyclic intrusive lists, per-program `clientdata`, lookups that insert. | There is no encapsulated interface to put Rust behind. Mirroring the structs in Rust with raw pointers gives up most of what Rust offers. |
| O3 | **The undo stack.** Ordered undo steps with masks, checkpoints, success-only fallbacks and failure gating, triggered from any depth. | `Result` plus `Drop` does not express "run only on error", "skip if an earlier undo failed" or "past this point, behave as on success". It has to be redesigned explicitly. |
| O4 | **Translations are C format strings.** 1,323 messages in 44 languages, 848 of them printf formats, some reordered with `%1$s`. | Rust's `format!` works at compile time with a different syntax. Keeping the catalogs requires a small run-time printf-style formatter and unchanged message texts. |
| O5 | **Children that fork without exec.** The compression filter and the `dpkg-deb` pipeline run library code in forked children. | Rust's standard library assumes fork is followed by exec. These become in-process streaming or threads. |
| O6 | **Quirks are behaviour.** Legacy status repairs, `strtol` epoch parsing, prefix matching of priorities, fixed field order, the escalating dependency passes. | Each must be found, written down and tested ([08, section 4](08-findings.md#4-behaviour-that-looks-like-a-bug-but-is-a-contract)). |
| O7 | **Byte-exact outputs.** Reproducible `.deb` files, `dpkg-query` layouts, status-fd lines, log lines, messages compared literally by the autotests. | Any deviation is a regression for someone. |
| O8 | **Callbacks across the library boundary.** Trigger hooks, tar callbacks, option tables, cleanup functions. | Each crossing would need error-model translation in a mixed build. |
| O9 | **`dselect`** is C++ and uses libdpkg's structs and the C++ wrappers in `varbuf.h`. | It keeps C libdpkg alive unless it is retired or ported. |

O1, O2, O8 and O9 are obstacles to *mixing* C and Rust in one process. They do not apply
to replacing whole programs. O3 to O7 apply to any reimplementation.

## 5. Risks

Likelihood and impact are judgement.

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | **A behavioural regression in install, upgrade or removal breaks systems.** | high without countermeasures | severe: unbootable systems, corrupted database | Differential testing against C; the functional suite; crash-injection tests; archive-wide install tests; staged, opt-in rollout with the C binary as fallback; install engine last |
| R2 | **Platforms without Rust.** Rust has no target for alpha, hppa, sh4 or ia64; m68k, the MIPS family, 32-bit sparc, powerpc-spe and Hurd are tier 3; there is none for kFreeBSD. dpkg also builds today on the BSDs, macOS, Solaris and AIX. | certain | high for those users | A policy decision: keep the C implementation for them (double maintenance), or drop them. Track `rustc_codegen_gcc` and gccrs |
| R3 | **Bootstrap position.** dpkg is Essential and installs the compiler that builds it. A Rust dpkg adds rustc, cargo and LLVM to what a new architecture must have before it has a package manager. | certain | medium | Cross-compilation; a minimal, packaged crate set; coordination with the Debian Rust and bootstrap teams |
| R4 | **Packaging constraints for an Essential package.** Static linking of the Rust standard library means security fixes in it or in crates need rebuilds; every crate must be in Debian main or vendored into a native source package; binaries grow. | certain | medium | Few dependencies; one multi-call binary; size-optimised profile; `Static-Built-Using` |
| R5 | **Upstream keeps moving.** Roughly 100 to 130 commits a year touch the C code in `lib/dpkg/` and `src/`, and nothing in the tree suggests upstream plans to adopt Rust. | high | high: permanent fork, growing divergence | Decide the relationship with upstream first; port changes continuously; treat upstream's test suites as the contract |
| R6 | **Reproducibility of `.deb` output.** Compressed bytes depend on the compressor implementation. | high if pure-Rust codecs are used | high: every package hash changes | Bind the same C libraries (liblzma, zlib, libzstd, libbz2); reproduce GNU tar's header bytes exactly or keep running tar |
| R7 | **Translations lost.** Changed message texts become untranslated in 44 languages. | medium | medium | Keep texts byte-identical; run-time formatter with positional arguments; extraction needs gettext ≥ 0.24 (the analysis machine has 0.23.2) |
| R8 | **New logic bugs.** Race conditions and path-handling errors in fresh code. | medium | high | Independent security audit before the code becomes the default; directory-descriptor file access; fuzzing |
| R9 | **Mixed installations.** During migration, C and Rust programs share one database, and dpkg upgrades itself while running. | certain | medium | Identical lock primitive (`fcntl` record locks) and file formats; test every pairing |
| R10 | **Residual `unsafe`.** SELinux, compression libraries, `fork`/`chroot`/`exec`, `ioctl`, `fcntl` locks. | certain | low to medium | Confine to small modules; review separately |
| R11 | **Cost and opportunity cost.** The same effort could fuzz and harden the C code. | — | — | Phase 0 delivers that hardening first; decide on later phases afterwards |
| R12 | **Consumers of libdpkg and dselect.** `libdpkg-dev` ships the static library; dselect needs it. | medium | low | Keep C libdpkg for them, or retire both |

Two notes on R4, from experiments on the analysis machine:

- A trivial Rust program is 342 KB stripped, about the size of the whole C `dpkg` binary
  (331 KB). Ten separate Rust binaries would multiply the Essential package's size; one
  multi-call binary avoids that.
- Rust binaries link `libgcc_s.so.1`, which no dpkg binary does today. It would become a
  new `Pre-Depends`. On the analysis machine (Ubuntu) `libc6` already depends on
  `libgcc-s1`, so the set of packages in the base system would not grow.

## 6. Strategies compared

| | Strategy | For | Against | Verdict |
|---|---|---|---|---|
| A | **In-place hybrid**: replace libdpkg modules with Rust behind the existing C headers | Ships continuously inside the existing build; could be offered upstream piece by piece | O1, O2 and O8 apply in full: only leaf modules are practical; needs cargo inside automake and libtool from day one; the shipped `libdpkg.a` would contain Rust objects; the result in the core would be C written in Rust syntax | Only for individual parsers, and only if upstream integration is the goal |
| B | **Program-by-program replacement** on a new Rust library | No `longjmp` interop at all; existing black-box suites apply unchanged; each program ships on its own; five precedents in this repository | Library logic exists twice during the transition; the C and Rust programs must interoperate on the database | **Recommended** |
| C | **Complete rewrite released at once** | Clean design | No intermediate validation; years without users; highest risk | No |
| D | **Machine translation** (c2rust or similar) followed by clean-up | Fast first result | `setjmp`/`longjmp`, the central mechanism, cannot be translated; output is unsafe Rust that mirrors the C design | No |
| E | **Harden the C code instead**: fuzzers, unit tests for the database layer, fix the findings, directory-descriptor file access | Cheapest; no platform, bootstrap or translation risk; upstreamable | Does not remove the bug classes or the duplicated logic | Do this first in any case (phase 0) |

## 7. Shape of a Rust implementation

A sketch to make strategy B concrete. Crate and type names are suggestions.

```
rust/
  crates/
    dpkg-version      version parsing and ordering (no dependencies)
    dpkg-arch         architecture names, tables from data/
    deb822            tokenizer, stanza model, writer
    dpkg-deps         dependency syntax and predicates
    dpkg-ar           .deb container reader and writer
    dpkg-tar          tar reader (and a writer matching GNU tar's bytes)
    dpkg-compress     streaming bindings to liblzma, zlib, libzstd, libbz2
    dpkg-i18n         gettext lookup and a printf-style run-time formatter
    dpkg-cli          dpkg's option-parsing rules and configuration files
    dpkg-fsops        root-relative file access through directory descriptors; atomic files; fsync policy
    dpkg-db           status and journal, info/, file lists, diversions, statoverride, triggers, locks
    dpkg-txn          the undo log
  bins/               one multi-call binary dispatching on argv[0], or thin per-program binaries
  fuzz/               fuzz targets for every parser
  difftest/           harness that runs the C and Rust programs side by side
```

Design rules:

1. **Errors are values.** Every fallible function returns a result. The per-archive and
   per-package loops are the only places that catch and continue. Panics abort.
2. **Rollback is data.** A transaction owns an ordered list of undo steps. Each step says
   when it runs (on error, on success, always). A checkpoint converts earlier steps to
   success semantics. A failed undo step suppresses later package-level abort scripts.
   Journal writes are marked as non-recoverable: a failure there exits without rollback.
   This transcribes today's semantics and, unlike today, can be unit-tested by injecting a
   failure after each step.
3. **The package graph uses indices, not pointers.** Packages, package sets and path nodes
   live in vectors and refer to each other by typed identifiers; strings are interned.
   This keeps the "everything lives until the database is closed" lifetime without
   reference-counted cycles.
4. **The installation root is a directory descriptor.** No path strings are concatenated
   with the root. The cases where dpkg deliberately follows an existing symlinked
   directory are explicit and tested.
5. **Message texts do not change.** A translation macro looks the C-format message up with
   gettext and formats it at run time, including positional arguments.
6. **No fork without exec.** Decompression streams in-process. Maintainer scripts run
   through a process builder with a pre-exec hook for `chroot` and the SELinux context.
7. **Few dependencies.** Bindings for libc, the four compression libraries and libselinux,
   an MD5 crate, and little else; no command-line framework (dpkg's rules are its own);
   everything available as a Debian package.
8. **A compatibility register.** Every known difference from the C behaviour is recorded
   with a decision (keep or fix) and feeds the differential harness's list of accepted
   differences.

## 8. Verification

| Layer | What | Exists today? |
|---|---|---|
| Black-box suites | `src/at/` (80 groups), `tests/` (89 scenarios), `utils/t/` (712 assertions), selected by `PATH` | yes |
| Test vectors | Version pairs, architecture tables, control-file and changelog fixtures from the C and Perl unit tests | yes, need extracting |
| Differential harness | Run C and Rust on the same input in scratch roots; compare stdout, stderr, exit status, status-fd stream, log, and the resulting database and file tree (names, types, modes, owners, content hashes) | **to build** |
| Fuzzing | Coverage-guided fuzzers per parser, with the C implementation as oracle | **to build** |
| Corpus runs | Every `.deb` of a Debian release through list, info, extract and tar-stream output in both implementations | **to build** |
| Crash consistency | Kill the process at chosen system calls; check that the next run recovers | **to build**; nothing like it exists for the C code either |
| Archive-wide runs | Install, upgrade and remove large package sets in containers | **to build** |
| Audit | Independent review of the unpack path, locking and privilege handling | — |

The analysis already showed that the differential approach is cheap: every
maintainer-script sequence in [03](03-dpkg-programs.md#47-maintainer-scripts) was
reproduced as a normal user in a scratch root within seconds.

## 9. Roadmap

Each phase ends with a criterion that can be checked.

**Phase 0 — Foundations (no Rust shipped).**
Run the functional suite in CI. Build the differential harness and prove it by comparing
two builds of the C code. Add fuzzers for the C parsers. Report the findings in
[08](08-findings.md) upstream. Take the decisions in section 10.
*Exit:* the harness reports no differences between two C builds across all suites; fuzzers
run in CI.

**Phase 1 — Pure-logic crates.**
Version, architecture, deb822 tokenizer, dependencies, `ar`, tar reader, message formatter,
option parser.
*Exit:* all extracted test vectors pass; differential fuzzing against C finds nothing for an
agreed amount of machine time; every accepted difference is in the compatibility register.

**Phase 2 — Leaf programs.**
`dpkg-realpath`, `dpkg-trigger`, `dpkg-split`, `dpkg-statoverride`, `update-alternatives`.
*Exit:* their autotest groups and the 712 update-alternatives assertions pass unchanged;
the harness is clean; C `dpkg` works with the Rust helpers in the functional suite.

**Phase 3 — `dpkg-deb` and the read side of the database.**
`dpkg-deb` (all actions), `dpkg-query`, `dpkg-divert`, the read-only `dpkg` actions.
*Exit:* identical results for every package of a Debian release; packages built from the
same tree are byte-identical to those built by C `dpkg-deb`; C `dpkg` installs packages
using Rust `dpkg-deb`.

**Phase 4 — The install engine.**
Unpack, configure, remove, purge, triggers, selections.
*Exit:* all functional scenarios pass, including those that need root; status-fd streams
and resulting systems match C in the harness; crash-consistency and archive-wide runs are
clean; an external audit has been done.

**Phase 5 — Rollout.**
`start-stop-daemon` (write its tests first, on Linux); packaging; opt-in installation
beside the C programs; default only after a soak period; then decide whether C remains as
the implementation for platforms without Rust.

The Perl tools can move on a parallel track once phase 1 exists, because they share the
same core crates ([07](07-perl-replacement-analysis.md)).

## 10. Effort

Rough, judgement-based ranges in engineer-months, for people who know both dpkg and Rust:

| Phase | Estimate |
|---|---|
| 0 Foundations | 1–2 |
| 1 Pure-logic crates | 2–4 |
| 2 Leaf programs | 2–3 |
| 3 `dpkg-deb`, read side | 3–5 |
| 4 Install engine | 8–14 |
| 5 Rollout | 2–4 |
| **Total, C side** | **18–32** |

Not included: the Perl tools, and the continuing cost of following upstream. Writing the
code is the smaller part; in phase 4 most of the time goes into validation, audit and
soak. Tooling can shorten implementation but not the calendar time a soak period needs.

## 11. Lessons from comparable efforts

**APT.** In October 2025 APT's maintainer announced that APT would require Rust "no earlier
than May 2026", starting with the code that parses `.deb`, `ar` and `tar` files and with
signature verification through Sequoia. The discussion that followed is a preview of what
a Rust dpkg would meet: four Debian ports (alpha, hppa, m68k, sh4) had no Rust toolchain
and were told to get one or be retired; reviewers raised Debian's limited ability to issue
security updates for statically linked Rust code and the unfinished `Static-Built-Using`
tracking; an APT co-maintainer asked whether the parsing code should be removed from APT
rather than rewritten. Whether the Rust code has since shipped was not verified for this
study.

**Ubuntu's coreutils and sudo.** Ubuntu 25.10 switched to the Rust `uutils` coreutils and
`sudo-rs`. A bug in the Rust `date` broke automatic updates; some programs were slower;
self-extracting archives failed. Before 26.04 a commissioned security audit found race
conditions in `cp`, `mv` and `rm`, which stayed on the GNU versions for that release; the
transition completed in 26.10. `sudo-rs` needed two security fixes within weeks of
becoming the default.

**This repository.** `dpkg-statoverride`, `update-alternatives`, `dpkg-divert`, the
`dpkg-split` helper and `dpkg-realpath` were each rewritten in another language, one
program at a time, and `update-alternatives` was carried by a black-box test suite written
before the rewrite.

What these suggest:

- Passing an upstream test suite does not prevent regressions in real use; staged rollout
  and a fallback are necessary.
- The bugs that appear in new Rust system tools are logic and race bugs, not memory bugs.
  An audit of exactly those is worth more than one of `unsafe` blocks.
- Platform and packaging policy cause more friction than code. Decide and communicate them
  early.
- Write the black-box tests before the rewrite, not after.

## 12. Decisions for the project owner

1. **What is the goal?** A contribution to upstream dpkg, an independent drop-in
   replacement in the manner of uutils, or an exploration? This decides between strategies
   A and B and whether C must be maintained in parallel.
2. **Platform policy.** What happens on architectures and operating systems without a Rust
   toolchain?
3. **Compatibility policy.** Reproduce known bugs or fix them? Item by item, recorded.
4. **Scope.** Installer only, or the Perl build tools as well?
5. **`dselect` and `libdpkg-dev`.** Keep on the C library, port, or retire.
6. **Dependency policy** for an Essential package: which crates, vendored or packaged, and
   the minimum Rust version (the one in Debian stable).
7. **Licensing.** dpkg is GPL-2-or-later. Crates under MIT or dual MIT/Apache-2.0 fit;
   Apache-2.0-only crates would make the combined binaries GPL-3-or-later.
8. **Rollout owner.** A Rust dpkg matters only if a distribution ships it. Which one, and
   who there agrees to the staging plan?

## Sources for section 11

- LWN, "APT Rust requirement raises questions": <https://lwn.net/Articles/1046841/>
- Julian Andres Klode on debian-devel, 31 October 2025: <https://lists.debian.org/debian-devel/2025/10/msg00288.html>
- LWN, "Date bug affects Ubuntu 25.10 automatic updates": <https://lwn.net/Articles/1043103/>
- OMG! Ubuntu, "Ubuntu 26.10 completes transition to Rust-based coreutils": <https://www.omgubuntu.co.uk/2026/09/ubuntu-2610-rust-coreutils-complete>
- The Register, "Ubuntu 25.10's Rusty sudo holes quickly welded shut": <https://www.theregister.com/2025/11/13/ubuntu_rust_sudo_hole/>
- Rust platform support: <https://doc.rust-lang.org/nightly/rustc/platform-support.html>
- GNU gettext 0.24 release notes (Rust support in xgettext): <https://lists.gnu.org/archive/html/info-gnu/2025-02/msg00010.html>
