# dpkg: how it works, and what a Rust rewrite would involve

This directory documents the dpkg code base as of upstream 1.23.x (commit `27f661e21`,
analysed on 2026-10-02) and assesses two questions:

- Could the C code be rewritten in Rust, and what would that gain and risk?
- Which of the Perl code could be replaced or rewritten, and is it worth it?

## Contents

| Document | What it covers |
|---|---|
| [01-repository-overview.md](01-repository-overview.md) | The whole repository on one page: components, how installing and building work end to end, the database layout, glossary, where to start reading |
| [02-libdpkg.md](02-libdpkg.md) | The C library: error handling, memory and data model, on-disk formats, parsers, archives, triggers |
| [03-dpkg-programs.md](03-dpkg-programs.md) | `dpkg`, `dpkg-deb` and the smaller programs: the install sequence, rollback, maintainer scripts, external contracts |
| [04-perl-toolchain.md](04-perl-toolchain.md) | The Perl modules and `dpkg-dev` programs, their public API, measured performance, translations |
| [05-build-test-release.md](05-build-test-release.md) | Build system, the six test suites, CI, Debian packaging, architecture coverage, what the git history shows |
| [06-rust-rewrite-analysis.md](06-rust-rewrite-analysis.md) | **Rust rewrite of the C code:** benefits, obstacles, risks, strategies, design sketch, roadmap, effort |
| [07-perl-replacement-analysis.md](07-perl-replacement-analysis.md) | **Replacing the Perl code:** what can and cannot go, options, recommended plan |
| [08-findings.md](08-findings.md) | Defects, C/Perl divergences and compatibility quirks found along the way |
| [reference/](reference/) | Five sets of detailed working notes (5,300 lines) with `path:line` citations for everything above |

Start with 01. Read 06 and 07 for the assessments; they can be read on their own.

## Summary

**The code base.** About 61,000 lines of C and C++ and 50,000 lines of Perl. The C side
installs packages and keeps the database; it is the only part needed on a running system.
The Perl side builds packages and doubles as a public library for other Debian tools. The
two share almost no code and reimplement several things twice.

**State of health.** All test suites pass: 3,989 C unit assertions, 80 autotest groups,
12,745 Perl assertions, 712 update-alternatives assertions and 81 functional scenarios.
Fourteen CVEs on record: ten in the Perl code (eight of them path traversal in
`dpkg-source`), four in C (one memory-safety bug, one format-string bug, two denial of
service). One maintainer writes 97 % of the non-translation commits, at 400 to 570 commits
a year.

**Rewriting the C code in Rust** ([06](06-rust-rewrite-analysis.md)):

- Feasible, but only program by program. The library's `longjmp`-based error handling and
  its global, pointer-linked data model rule out replacing it gradually from inside.
- The programs already communicate only through processes and files, and the black-box
  test suites find programs through `PATH`, so a Rust binary can be dropped in and tested
  unchanged. Five programs in this repository have changed language that way before.
- Benefits, in order of weight: hardened reading of untrusted `.deb` files; a rollback
  mechanism that can be unit-tested; one implementation of logic now duplicated in C and
  Perl; memory safety (real, but the historical bug rate is low).
- Main risks: behavioural regressions in the install engine; architectures and operating
  systems without Rust (alpha, hppa, sh4, ia64 and several non-Linux systems dpkg supports
  today); dpkg's place at the bottom of the bootstrap chain; preserving 44 translations and
  byte-identical `.deb` output; following a fast-moving upstream.
- Suggested order: test tooling and fuzzers first (useful even with no rewrite), then
  pure-logic crates, leaf programs and `dpkg-deb`, and the install engine last. Rough
  effort for the C side: 18 to 32 engineer-months, most of it validation.

**Replacing the Perl code** ([07](07-perl-replacement-analysis.md)):

- The module library is a stable public API with about 35 dependent packages and cannot be
  removed. Perl is not needed to install packages, and cannot be removed from package
  building while debhelper and the autotools are Perl.
- The measurable problem is start-up time (17 to 32 ms per tool) multiplied by dozens of
  invocations per build: about 1.2 s of a 2.8 s minimal build. Much of that can be removed
  by small changes to the Perl and the make fragments.
- Worth rewriting natively: the small hot tools (`dpkg-architecture`, `dpkg-buildflags`,
  `dpkg-buildapi`, `dpkg-vendor`, `dpkg-parsechangelog`), possibly `dpkg-shlibdeps`, and a
  narrowly scoped hardened source extractor. Not worth it: `dpkg-buildpackage`, the whole
  of `dpkg-source`, the `.changes` and `.buildinfo` generators.

**Findings** ([08](08-findings.md)): eight items with possible security relevance, about
thirty correctness bugs, and ten disagreements between the C and Perl implementations.

## Decisions needed

The analyses stop where the project owner's choices begin. The main ones
([06, section 12](06-rust-rewrite-analysis.md#12-decisions-for-the-project-owner)):

1. Is the goal a contribution to upstream dpkg, an independent drop-in replacement, or an
   exploration?
2. What happens on platforms without a Rust toolchain?
3. Are known bugs reproduced or fixed?
4. Is the Perl build toolchain in scope?

## How this was produced, and how far to trust it

Five parallel automated analyses read the code area by area (library, C programs, Perl
modules, Perl programs, build and history), ran the test suites and timing experiments,
and wrote the notes in `reference/`. The chapters were then written from those notes.

Checked by running code during the analysis:

- `make check` and the functional suite (results in [05](05-build-test-release.md#2-test-suites));
- every maintainer-script sequence in [03](03-dpkg-programs.md#47-maintainer-scripts),
  in a scratch root as a normal user;
- start-up timings and process-spawn counts in [04](04-perl-toolchain.md#4-measured-performance);
- every finding marked "Verified" in [08](08-findings.md).

Independently re-checked while writing the chapters: the three `setjmp` recovery sites, the
static-only library build, the CVE list from git history, the contributor shares, the
version-comparison divergence and finding P1 (by running them), the source lines behind
findings P2, S1, S2, S3 and S7 (by reading them), and the earlier language rewrites in git
history.

Known limits:

- The memory-safety commit classification read commit messages, not diffs.
- Which dpkg outputs apt and other front-ends rely on was not checked against their source.
- Who uses `libdpkg-dev`, the make fragments, symbols-file regexes or custom vendor classes
  across the Debian archive was not measured.
- Timings are from one machine (Ubuntu 26.04, perl 5.40).
- Statements about Rust's platform support, about APT's Rust plans and about Ubuntu's Rust
  coreutils come from public sources listed in [06](06-rust-rewrite-analysis.md#sources-for-section-11);
  the current status of APT's Rust work was not verified.
- Difficulty ratings and effort estimates are judgement, not measurement.

## Before publishing

[08-findings.md](08-findings.md) and the reference notes describe several unreported
issues in upstream dpkg, some with possible security relevance. Report those upstream and
allow time for assessment before pushing these files to a public repository.
