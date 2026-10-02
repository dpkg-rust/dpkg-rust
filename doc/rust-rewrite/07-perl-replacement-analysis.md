# 07 — Replacing or rewriting the Perl code

This page assesses which of the repository's Perl could be replaced, by what, and whether
it is worth it. The Perl code is described in [04](04-perl-toolchain.md); measurements and
citations are in [reference/03-perl-library.md](reference/03-perl-library.md) and
[reference/04-perl-programs.md](reference/04-perl-programs.md).

Facts are referenced. Recommendations, difficulty and value are judgement.

## 1. Conclusions

1. **The Perl library cannot go away.** The `Dpkg::*` modules are a documented stable API
   used by debhelper, lintian, devscripts, sbuild and some thirty other packages. Their
   contract includes Perl behaviour (tied hashes, overloaded operators, callbacks,
   plug-ins loaded by module name), not only functions. Any plan has to start from
   "libdpkg-perl stays".
2. **Perl is not needed to install packages, and never will be removable from package
   building** as long as debhelper and the autotools are written in Perl. Removing Perl as
   a dependency is therefore not an achievable goal in Debian; the realistic goals are
   speed, robustness and a single source of truth.
3. **The language-level benefit of a rewrite is smallest here.** Perl is already
   memory-safe. Ten of dpkg's fourteen CVEs were in this Perl code, and all were logic
   errors (mostly path traversal) that a rewrite can reintroduce.
4. **The measurable problem is start-up latency multiplied by spawn count**: 17 to 32 ms
   per small tool, run dozens of times per build by the make fragments. About 1.2 s of a
   2.8 s minimal build is that overhead. A large part of it can be removed *without*
   rewriting anything.
5. **Worth rewriting natively**, in this order: the small, hot, table-driven tools
   (`dpkg-architecture`, `dpkg-buildflags`, `dpkg-buildapi`, `dpkg-vendor`,
   `dpkg-parsechangelog`); then possibly `dpkg-shlibdeps`; and, for security rather than
   speed, a narrowly scoped hardened source extractor. **Not worth rewriting:**
   `dpkg-buildpackage`, the full `dpkg-source`, the generators of `.changes` and
   `.buildinfo`, the dselect methods.
6. **The main new risk is divergence.** A native tool duplicates logic that must remain in
   Perl for API users. debhelper, for example, calls `Dpkg::BuildFlags` in-process and
   also runs `dpkg-buildflags`. The two must never disagree.

## 2. Where Perl is used, and whether it can be replaced

| Use | Size | Needed when | Replaceable? |
|---|---|---|---|
| `Dpkg::*` public modules (43) | 7,228 code lines | package builds; external tools | **No.** Stable API |
| `Dpkg::*` private modules (49) | 8,005 code lines | by the `dpkg-dev` programs | Only together with the programs that use them; some are used by external tools despite being private |
| `dpkg-dev` programs (19) | 8,353 lines | package builds | Yes, one by one (section 5) |
| Make fragments `scripts/mk/` | 376 lines of make | package builds | They are make, not Perl; they *call* the Perl tools and cause most of the spawn count |
| dselect access methods | about 2,200 lines | using dselect | Not worth it (legacy) |
| `configure` requires perl ≥ 5.36 and runs `dpkg-architecture.pl`, `dpkg-parsechangelog.pl` | — | building dpkg | Yes: a native architecture tool built for the build machine; date from a shell script |
| Substitution of paths and versions into shell scripts (`build-aux/subst.am`) | — | building dpkg | Yes, with `sed` |
| `pod2man`, po4a for manual pages | 63 POD files | building dpkg's documentation | Only by changing the documentation format, which would discard 13 languages of translations. Leave |
| `dselect/mkcurkeys.pl` | 148 lines | building dselect | Yes, trivially |
| TAP test runner, `scripts/t`, `utils/t`, `t/`, three `lib/dpkg/t/*.t` drivers | — | running tests | The runner could be automake's shell TAP driver; the tests that test Perl stay |
| autoconf, automake | — | building from git | No (they are Perl themselves) |
| The Essential `dpkg` binary package | — | installing packages | **Already contains no Perl** |

## 3. Reasons to replace Perl, with their evidence

**Start-up latency.** The cheapest tool takes 17 ms to start against 0.45 ms for a C
binary. The make fragments and debhelper start these tools many times: a minimal debhelper
build that includes `default.mk` runs `dpkg-buildflags` 40 times and `dpkg-buildapi` 8
times, and each `override_dh_*` target adds 22 more runs.

**A single implementation.** The C and Perl sides disagree on version comparison, control
parsing and dependency syntax in verified ways. A core shared with a Rust installer would
end that for every tool built on it.

**Fragile text interfaces.** `dpkg-shlibdeps` parses the text printed by `objdump` and by
`dpkg-query --search`. Native ELF parsing and direct database access would remove both
couplings, and the 69 ms that `dpkg-query` spends loading every file list.

**Security of source extraction.** `dpkg-source -x` processes downloaded, possibly hostile
source packages and accounts for eight CVEs. It delegates extraction to GNU tar and patch
application to GNU patch and relies on their defences. An extractor that performs both
in-process, confined to a directory descriptor, would be safer by construction. This is a
design argument; the language only makes it cheaper.

**Bugs found.** The study found fifteen defects in the Perl programs and modules, three of
them security-relevant ([08](08-findings.md): P1–P12, S1, S2, S7). They are also all
fixable in Perl in a few lines each.

## 4. Reasons to leave Perl alone, with their evidence

**The stable API.** 43 public modules, about 387 functions, 35 reverse dependencies on the
analysis machine. History shows external tools breaking on changes to undocumented and
even private behaviour.

**Plug-ins are Perl modules.** Vendor classes (`Dpkg::Vendor::<Name>`, with 15 hooks),
changelog format parsers (a *stable* interface), source format classes, build drivers and
OpenPGP back-ends are all found and loaded by module name. Derivatives ship or patch their
own: the Ubuntu-patched `dpkg-buildflags` on the analysis machine loads 32 modules where
the upstream one loads 18.

**Perl regular expressions are part of file formats.** The `(regex)` tag in symbols files
is documented as a Perl regular expression. `--extend-diff-ignore` in
`debian/source/options` is one too. Rust's standard `regex` crate has no look-around or
back-references; engines that do (PCRE2, `fancy-regex`) are close to Perl but not
identical. How many packages use these features was not measured. The regular expressions
in the architecture tables are plain POSIX syntax and pose no problem.

**Most logic is in modules that must stay.** Nine of the twenty programs are thin
wrappers. Rewriting the program duplicates the module.

**Thin program-level tests.** Five test files drive programs; 46 test modules through the
Perl API and would not exercise a native implementation.

**It is the most active code.** `scripts/` receives about 190 commits a year, more than
`lib/` and `src/` together. A rewrite chases a moving target.

**Byte-exact outputs.** `DEBIAN/control`, `.dsc`, `.changes` and `.buildinfo` feed
reproducible-builds and archive tooling; field order and wrapping are fixed by tables.

**No dependency is saved.** `perl-base` is Essential, and `dpkg-dev` also needs make,
binutils, patch, tar and xz.

## 5. Options

### Option 0 — Improve the Perl and the make fragments (no rewrite)

| Change | Expected effect | Evidence |
|---|---|---|
| Load `Locale::gettext` lazily, or avoid pulling in POSIX and Encode at start-up | 7–9 ms less per tool invocation; `dpkg-architecture -q` 16.9 → about 8 ms | Measured with `DPKG_NLS=0` |
| Cache `DPKG_BUILD_API` in `buildapi.mk`, or export it from `dpkg-buildpackage` | Removes two `dpkg-buildapi` runs (about 60 ms) per make process | Spawn counts measured; the fix itself was not tried |
| Obtain all build flags with one `dpkg-buildflags --export=make` call instead of one call per flag; do not recompute flags inherited from the environment | Removes up to 20 runs (about 0.5 s) per make process in debhelper builds | Mechanism confirmed by experiment; the fix itself was not tried |
| Fix the bugs listed in [08](08-findings.md): the `Build-Driver` eval, `dpkg-name` path handling, shell quoting in `dpkg-buildflags`, the two `implies()` errors, and the rest | Correctness and security | Verified findings |
| Share one status-file parser between `dpkg-checkbuilddeps` and `dpkg-genbuildinfo` | Less duplication | Both carry their own copy |
| Add command-line golden tests for every program | Makes any later rewrite verifiable | Only five program tests exist |

This option is cheap, upstreamable, and removes most of the measured latency. It should be
done whatever else is decided.

### Option 1 — Native versions of the hot, thin tools

Rewrite as Rust programs on the shared core crates of
[06, section 7](06-rust-rewrite-analysis.md#7-shape-of-a-rust-implementation):

| Tool | Why | Size of the logic | Catch |
|---|---|---|---|
| `dpkg-architecture` | 33 runs per make process outside `dpkg-buildpackage`; also run by `configure` | 315 lines + `Dpkg::Arch` 438 | Output formats and environment-override rules must match exactly |
| `dpkg-buildflags` | 20–80 runs per debhelper build | 146 lines + about 870 lines of flag policy | The policy is Perl *code* in vendor classes that derivatives patch, and debhelper calls the Perl module directly |
| `dpkg-buildapi` | 8–12 runs per build, 32 ms each to print one digit | 29 lines | None, but option 0 removes most runs anyway |
| `dpkg-vendor` | run by `vendor.mk` | 54 lines + `Dpkg::Vendor` | Vendor *classes* stay Perl; only the origins-file queries move |
| `dpkg-parsechangelog` | one run per field in `pkg-info.mk`; widely scripted | 80 lines + about 780 lines of changelog code | Custom formats are Perl plug-ins: the native tool must fall back to the Perl one for any format other than `debian` |

Each would start in about a millisecond instead of 17 to 32.

The catch for `dpkg-buildflags` is the important one. To keep a native tool and
`Dpkg::BuildFlags` from diverging, the flag policy (which feature is on for which
architecture, and which flags it adds) has to move out of Perl code into **data files read
by both implementations**, with vendors overriding data instead of subclassing. That is a
redesign of an extension point, and it needs agreement from the derivatives that use it.

### Option 2 — A Rust core underneath the Perl modules

Keep the Perl API but implement Version, Arch, Deps, the deb822 parser and similar in Rust,
exposed through a compiled binding.

- **For:** one implementation of the core logic.
- **Against:** `libdpkg-perl` is `Architecture: all` pure Perl today and would become a
  compiled, architecture-dependent package, with consequences for cross-building and
  Multi-Arch that were not investigated. A Perl layer must remain on top for the tied
  hashes, overloads and callbacks. And it does not address latency: start-up time is spent
  loading Perl modules, not computing (40,000 version comparisons per second is not a
  bottleneck).
- **Assessment:** poor value. The single-source-of-truth goal is better served by sharing
  *data and test vectors*: generate the field registry, architecture tables and flag
  tables from one source for every implementation, and run one set of cross-implementation
  test vectors against C, Perl and Rust in CI.

### Option 3 — Native versions of the heavy tools

| Tool | Benefit | Difficulty | Risk | Assessment |
|---|---|---|---|---|
| `dpkg-shlibdeps` | 228 ms per package today; native ELF and database access remove two text-parsing couplings | L | High: it generates every package's `Depends`; symbols files contain Perl regexes and C++ names demangled by binutils | Candidate, after option 1, with a corpus comparison over a whole archive |
| `dpkg-gensymbols` | Shares the above | L | High: maintainers read its diff output | With `dpkg-shlibdeps` or not at all |
| `dpkg-gencontrol` | 46 ms per binary package | M to L | High: byte-exact `DEBIAN/control` | Low priority |
| `dpkg-checkbuilddeps` | 55 ms once per build | M | Medium | Low priority |
| `dpkg-genchanges`, `dpkg-genbuildinfo` | once per build | M | High: consumed by archive and reproducibility tooling | Not worth it |
| `dpkg-scanpackages`, `dpkg-scansources` | Faster repository indexing (today one `dpkg-deb` + `tar` + `rm` per package) | S to M | Medium | Reasonable once Rust `.deb` reading exists |
| `dpkg-name`, `dpkg-distaddfile`, `dpkg-buildtree`, `dpkg-mergechangelogs` | negligible | S | Low | Fix in Perl instead |
| `dpkg-buildpackage` | 38 ms once per build | L | High: 66 options, hook language, signing back-ends, used by sbuild and others | Not worth it |
| `dpkg-source` (complete) | The security hot spot | **XL**: five formats, patch generation and application, quilt, git, bzr, signatures | Very high | Not as a whole (see below) |

**A hardened source extractor.** Instead of rewriting `dpkg-source`, write one narrow
program that only *extracts* the common formats (1.0, 3.0 native, 3.0 quilt): verify
checksums, unpack tarballs and apply patches in-process, with every file operation confined
to the target directory by descriptor. It would cover the code path behind eight CVEs
while leaving building, committing and the exotic formats in Perl. The hard part is
matching GNU patch's behaviour on real-world patches, which needs a corpus test over every
source package in an archive before it could be trusted. Difficulty L; value high if the
goal is security.

### Option 4 — Replace all of dpkg-dev

Rewrite all nineteen programs and freeze the Perl library as a compatibility layer.

- This means maintaining two complete implementations indefinitely, because the library
  cannot be removed, in the part of the repository that changes most.
- **Assessment:** not recommended.

## 6. Recommended plan

1. **Now, in Perl and make (option 0):** lazy gettext loading, the make-fragment fixes, the
   bug fixes, command-line golden tests. Offer them upstream.
2. **With the Rust core crates in place (phase 1 of [06](06-rust-rewrite-analysis.md#9-roadmap)):**
   native `dpkg-architecture`, `dpkg-buildapi`, `dpkg-vendor`, and `dpkg-parsechangelog`
   with a Perl fallback for plug-in formats. Use the native `dpkg-architecture` in
   `configure`, which removes one of the two reasons `configure` runs Perl.
3. **After agreeing a data-driven flag policy with vendors:** native `dpkg-buildflags`,
   with `Dpkg::BuildFlags` reading the same data.
4. **Optional, on their own merits:** the hardened source extractor (security), then
   `dpkg-shlibdeps` and `dpkg-gensymbols` (speed and robustness), then the index
   generators.
5. **Leave in Perl:** the module library, `dpkg-buildpackage`, full `dpkg-source`, the
   `.changes` and `.buildinfo` generators, the dselect methods, the documentation tooling
   and the tests.

Throughout: one set of test vectors run against every implementation, in CI.

## 7. Risks specific to this track

| Risk | Mitigation |
|---|---|
| A native tool and the Perl module it mirrors give different answers | Shared data files; shared test vectors; run both in CI and compare on real packages |
| Derivatives' vendor patches no longer take effect | Design the data-driven vendor mechanism with them before switching |
| Custom changelog formats, vendor classes or build drivers stop working | Fall back to the Perl implementation whenever a plug-in is involved |
| Perl regular expressions in package data behave differently | Keep those tools in Perl, or use a Perl-compatible engine and compare over an archive-wide corpus |
| Translations: 916 messages in 11 languages; 433 of them are program help text | Keep message texts and the `dpkg-dev` domain unchanged; support `%1$s` reordering; do not let a command-line framework reformat help |
| Idiosyncratic option parsing (attached values such as `-Vname=value`, `-O` versus `-Ofile`, three different parser styles) | Reproduce per tool; golden tests |
| Output read by other tools (`dpkg-architecture` variable lists, `dpkg-buildflags --status`, `dpkg-parsechangelog` fields) | Golden tests; byte comparison on real packages |
| The Perl code changes about 190 times a year | Rewrite only tools whose logic is small and stable; keep the rest |
