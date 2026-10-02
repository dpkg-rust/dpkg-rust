> **Reference notes.** Detailed working notes behind the chapters in the parent directory,
> produced by automated code analysis of dpkg 1.23.x (commit `27f661e21`) on 2026-10-02.
> Every non-trivial claim cites `path:line`; items marked "(unverified)" were inferred, not run.
> `<repo>` is the source tree, `<build>` an out-of-tree build of it, `<scratch>` a temporary
> directory that no longer exists. See [../README.md](../README.md) for how these notes were checked.

# 04 — Perl programs, other Perl, i18n and docs (dpkg-rust @ 27f661e21, dpkg 1.23.12~)

Analyst scope: `scripts/*.pl`, `scripts/mk/*.mk`, `scripts/completion`, Perl in
`dselect/` (methods, `mkcurkeys.pl`), `build-aux/`, `t/*.t`, `utils/t`, Perl used
by build/packaging, i18n (`po/`, `scripts/po/`, `dselect/po/`, `man/po/`), and
`man/`, `doc/`. The `scripts/Dpkg/**` library is covered by another analyst; here
it is only characterised as "where the logic lives".

All paths are relative to `<repo>` unless they start
with `build/` (= `<build>`, the configured out-of-tree build)
or `/`. Line references are `path:line`.

## 0. Method, environment, reproducibility

* Host: Intel i9-13900 (P-cores = CPUs 0-15, E-cores = 16-31; `/sys/devices/cpu_core/cpus`),
  32 threads, governor `powersave`, Linux 7.0, perl 5.40.1, GNU make 4.4.1,
  strace 6.19, xgettext 0.23.2. System packages: dpkg-dev/libdpkg-perl
  1.23.7ubuntu1, debhelper 13.31ubuntu1 (Ubuntu 26.04 "resolute").
* In-tree runs use the same environment `build-aux/run-script` sets
  (`build-aux/run-script:14-16`): `PERL5LIB=$src/scripts:$src/dselect/methods`,
  `DPKG_DATADIR=$src/data`, then `perl $src/scripts/<tool>.pl` (I call perl
  directly instead of through the `/bin/sh` wrapper so the wrapper's cost is not
  timed). The processed `build/scripts/dpkg-*` are byte-identical to the `.pl`
  sources here (`diff` → identical; only the shebang would change,
  `build-aux/subst.am:29-49`). `Dpkg.pm` reads `DPKG_DATADIR`,
  `DPKG_PROG{MAKE,TAR,PATCH}` from the environment (`scripts/Dpkg.pm:96-105`);
  `CONFDIR` stays `/etc/dpkg`, so the system `/etc/dpkg/origins/default`
  (→ vendor **Ubuntu**) and `/var/lib/dpkg/status` (2215 packages, 2.28 MB) are used.
* Timing harness: `scratchpad/timing/bench.py` — `os.posix_spawnp` + `waitpid`,
  stdout/stderr → `/dev/null`, 3 warm-up runs then N timed runs, reports
  min/median/p90/max wall-clock ms. Headline numbers are from runs **pinned to one
  P-core** (`taskset -c 4`), which gave tight distributions (p90 within ~2-5% of
  median). An unpinned run (n=50) is kept for reference: medians were 10-60% higher
  and noisy because the scheduler mixes P/E cores (e.g. `perl -e 1` 0.86 ms
  unpinned vs 0.72 pinned; C `dpkg --version` 1.70 vs 0.45).
* Module census: `scratchpad/timing/DumpINC.pm` loaded via `PERL5OPT=-MDumpINC`,
  dumps `%INC` at `END` (so runtime `require`s are included when the tool really
  runs; `exec` paths would be missed). "Module code" = non-blank, non-comment,
  non-POD lines of the loaded `Dpkg*.pm` files.
* Spawn counting: `strace -f -e trace=execve`, post-processed by
  `scratchpad/timing/execcount.py` (pairs `<unfinished>`/`resumed` lines, counts
  only successful execs; the host PATH has ~12 entries so raw `execve` lines
  over-count). Make-fragment experiments used logging PATH shims in
  `scratchpad/mk/bin/`.
* End-to-end build: a minimal dh-compat-13 native package (`hello`, 1 C binary,
  `Rules-Requires-Root: no`) in `scratchpad/dbp/hello-1.0`, built with the
  **in-tree** `dpkg-buildpackage` and in-tree Perl tools/modules
  (`PATH=build/scripts:$PATH PERL5LIB=scripts`), the system C `dpkg`/`dpkg-deb`/
  `dpkg-query` and the system debhelper; `DEB_BUILD_OPTIONS=noautodbgsym` (Ubuntu's
  debhelper renames dbgsym to `.ddeb`, which upstream `dpkg-genbuildinfo` then
  fails to find — observed once, rc=25, `scratchpad/dbp/build1.log`).

---

## A. Per-program profiles (`scripts/*.pl`)

### A.0 Summary table

LOC = `wc -l`. "logic" = non-blank/non-comment lines outside `sub usage`
(help text) — computed with a perl one-liner. "help opts" = `print_option(`
calls in `--help`; "man items" = `=item B<-` in `man/<prog>.pod`. "mods/code" =
Dpkg modules loaded and their code lines, measured on a real run where available
(R) otherwise at `--version` (V, compile-time `use` only).

| program | LOC | logic | help opts | man items | Dpkg mods / code | option parser | invoked by (verified unless noted) |
|---|---:|---:|---:|---:|---|---|---|
| dpkg-architecture | 498 | 315 | 19 | 19 | 7 / 902 (R) | `normalize_options` + exact match (`:236`) | mk (`architecture.mk`), dpkg-buildpackage (`-f`, `dpkg-buildpackage.pl:785`), debhelper `Dh_Lib.pm:1857` |
| dpkg-ar | 139 | 72 | 5 | 0 | 6 / 668 (V) | hand loop | **test-only**: `src/at/local.at:57` (not in `bin_SCRIPTS`, `scripts/Makefile.am:107-150`) |
| dpkg-buildapi | 83 | 29 | 3 | 3 | 34 / 5472 (R) | hand regexes (buggy, see I) | mk (`buildapi.mk:12`) |
| dpkg-buildflags | 250 | 146 | 10 | 12 | 18 / 3610 (R) | hand regexes | mk (`buildflags.mk:69`), humans, `debian/rules` of dpkg itself (`debian/rules:18-22`) |
| dpkg-buildpackage | 1214 | 737 | 66 | 77 | 52 / 7704 (R in build) | hand regexes + `Dpkg::Conf` injection (`:430-434`) | humans, sbuild/pbuilder/debuild (unverified, not installed here) |
| dpkg-buildtree | 89 | 36 | 4 | 2 | 9 / 826 (V) | hand loop | humans/`debian/rules` (no caller found in debhelper) |
| dpkg-checkbuilddeps | 270 | 149 | 10 | 10 | 33 / 5409 (R) | Getopt::Long | dpkg-buildpackage (`:885`) |
| dpkg-distaddfile | 100 | 42 | 3 | 3 | 8 / 656 (V) | hand regexes | `debian/rules` (byhand artifacts; no debhelper caller found) |
| dpkg-genbuildinfo | 638 | 419 | 12 | 13 | 49 / 7777 (R in build) | hand regexes | dpkg-buildpackage (`:967`) |
| dpkg-genchanges | 628 | 380 | 27 | 27 | 40 / 6557 (R in build) | hand regexes | dpkg-buildpackage (`:985`) |
| dpkg-gencontrol | 519 | 340 | 15 | 16 | 46 / 7180 (R in build) | hand regexes (attached values) | dh_gencontrol (`/usr/bin/dh_gencontrol:155,183`), per binary package |
| dpkg-gensymbols | 402 | 262 | 15 | 15 | 41 / 6904 (V) | hand regexes | dh_makeshlibs (`/usr/bin/dh_makeshlibs:446`) per library package |
| dpkg-mergechangelogs | 328 | 224 | 4 | 4 | 26 / 4369 (V) | Getopt::Long | git merge driver (`man/dpkg-mergechangelogs.pod:122-130`) |
| dpkg-name | 289 | 187 | 7 | 7 | 20 / 3637 (V) | hand regexes (buggy) | humans |
| dpkg-parsechangelog | 196 | 80 | 16 | 17 | 27 / 4453 (R) | `normalize_options` (`:118`) | mk (`pkg-info.mk:53-61`), many scripts (gen-release `build-aux/gen-release:36,43`) |
| dpkg-scanpackages | 351 | 241 | 9 | 9 | 24 / 4076 (V) | Getopt::Long | archive/local-repo tooling, humans |
| dpkg-scansources | 358 | 239 | 6 | 6 | 23 / 3824 (V) | Getopt::Long | archive/local-repo tooling, humans |
| dpkg-shlibdeps | 1070 | 753 | 19 | 18 | 43 / 7525 (R in build) | hand loop over all args (`:93-153`) | dh_shlibdeps (`/usr/bin/dh_shlibdeps:200`) per package |
| dpkg-source | 808 | 479 | 30 | 60 | 61 / 9248 (R in build) | hand regexes + `debian/source/{options,local-options}` injection (`:137-154`) | dpkg-buildpackage (3×: `:874,918,996`; plus `-x` at `:736`), humans, apt-get source (unverified) |
| dpkg-vendor | 123 | 54 | 6 | 6 | 12 / 2161 (R) | hand regexes | mk (`vendor.mk:50-54`), dpkg's own `debian/rules:39-42` |

Totals: 8353 LOC in 20 `.pl` files (19 installed + `dpkg-ar`). The logic in
the scripts themselves totals 5,184 lines; help text in `usage()` (and
`get_format_help()` in dpkg-source) is 1,515 lines; the rest is blanks/comments.

**Thin wrappers** (almost all logic in modules): dpkg-architecture
(Dpkg::Arch), dpkg-buildapi (Dpkg::BuildAPI + Dpkg::Control::Info), dpkg-buildflags
(Dpkg::BuildFlags + Dpkg::Vendor::*), dpkg-buildtree (Dpkg::BuildTree),
dpkg-distaddfile (Dpkg::Dist::Files), dpkg-parsechangelog (Dpkg::Changelog::Parse),
dpkg-vendor (Dpkg::Vendor), dpkg-ar (Dpkg::Archive::Ar), dpkg-source (driver
over Dpkg::Source::Package::* — ~5.4k module lines incl. POD).
**Heavy inline logic**: dpkg-shlibdeps (753 logic lines: library search, package
mapping, symbol resolution, dependency strength merging), dpkg-buildpackage (737:
orchestration, hooks, signing, checksum rewriting), dpkg-genbuildinfo (419: status
parsing, transitive build-dep closure, env cleansing, cross-exec test),
dpkg-genchanges (380), dpkg-gencontrol (340), dpkg-gensymbols (262), dpkg-scanpackages
(241), dpkg-scansources (239), dpkg-mergechangelogs (224), dpkg-name (187),
dpkg-checkbuilddeps (149, has its own "silly little status file parser",
`dpkg-checkbuilddeps.pl:193-229`; dpkg-genbuildinfo has a near-copy,
`dpkg-genbuildinfo.pl:93-146`, with a comment saying so).

### A.1 Profiles

Common to all: `use v5.36`, `Dpkg`, `Dpkg::Gettext` (text domain `dpkg-dev`),
`Dpkg::Getopt` (`print_option`, `print_version`), `Dpkg::ErrorHandling`.
"arch spawns" below = `gcc -dumpmachine` (host arch, `scripts/Dpkg/Arch.pm:168`,
via `qx($CC -dumpmachine)`) and/or `dpkg --print-architecture` (build arch,
`scripts/Dpkg/Arch.pm:138`); both are skipped when `DEB_HOST_ARCH`/`DEB_BUILD_ARCH`
are in the environment (`scripts/Dpkg/Arch.pm:151-154,234-237`), which
dpkg-buildpackage guarantees for its children (`dpkg-buildpackage.pl:785-792`).
Measured spawns are from strace with an empty environment unless noted.

**dpkg-architecture** — print/query/compare Debian arch tuples, export them to
env (`-c` execs a command with them, `:481-485`). Reads `$DATADIR/{cputable,ostable,tupletable,abitable}`
(`scripts/Dpkg/Arch.pm:272`). Env vars short-circuit computation unless `-f`
(`:304-311`). 33 variables (`:177-211`). Output formats: `VAR=value` list,
`--print-format shell|make` for `-s/-u` (`:458-476`, make format uses `undefine`).
Spawns: dpkg + gcc (no args), gcc only (`-qDEB_HOST_MULTIARCH`), dpkg only
(`-qDEB_BUILD_ARCH`). Logic: Dpkg::Arch (438 code lines). Thin wrapper.

**dpkg-ar** — create/list/extract ar archives with pure-Perl `Dpkg::Archive::Ar`.
Not installed; used by the autotest suite to craft/inspect `.deb`s
(`src/at/local.at:56-73`; `src/at/deb-format.at` uses the `DPKG_AR*` macros 49×,
`deb-split.at` 18×). Trivially replaceable.

**dpkg-buildapi** — prints the `dpkg-build-api` level from `debian/control`
Build-Depends or `DPKG_BUILD_API` (`scripts/Dpkg/BuildAPI.pm:73-74`). To print
"0" it loads 34 Dpkg modules (5,472 code lines) and spawns gcc + dpkg (arch needed
for dependency evaluation). 32.5 ms per call (D).

**dpkg-buildflags** — compute CFLAGS/LDFLAGS/… with vendor features (hardening,
LTO on Ubuntu, etc.). Reads `/etc/dpkg/buildflags.conf`,
`$XDG_CONFIG_HOME/dpkg/buildflags.conf` (`scripts/Dpkg/BuildFlags.pm:135-153`),
`DEB_*_{SET,STRIP,APPEND,PREPEND}` and `DEB_*_MAINT_*` env (`:172,202`),
`DEB_BUILD_OPTIONS`, `DEB_BUILD_MAINT_OPTIONS`, cwd/`DEB_BUILD_PATH` (for
`-ffile-prefix-map`, visible in output). Actions: `--get/--origin/--query-features/
--export=sh|make|cmdline|configure/--list/--status/--dump/--query`
(`dpkg-buildflags.pl:91-120`). `--status` lines are prefixed for log scraping
(`:211-213`). Spawns gcc + dpkg. Logic: Dpkg::BuildFlags (255 code) +
Dpkg::Vendor::Debian (472) + Dpkg::Vendor::Ubuntu (140). Note the Ubuntu-patched
system tool loads 32 Dpkg modules (incl. changelog parsing) vs 18 in-tree, and is
~40 ms vs 25 ms (D) — derivatives patch this logic.

**dpkg-buildpackage** — orchestrator. Reads `debian/changelog`, `debian/control`,
`/etc/dpkg/buildpackage.conf` + user conf (options injected before argv,
`:430-434`). Exports arch env (`:785-792`), `SOURCE_DATE_EPOCH` (`:777`),
`DEB_BUILD_OPTIONS` parallel (`:682-692`), `MAKEFLAGS` (`:687-696`),
`DEB_RULES_REQUIRES_ROOT`/`DEB_GAIN_ROOT_CMD` (`scripts/Dpkg/BuildDriver/DebianRules.pm:174-185`).
Runs: hooks (12 names, `:412-425`, via `system($cmd)`), dpkg-source, dpkg-checkbuilddeps,
`debian/rules` targets via the build driver (fakeroot by default when root is
needed, `DebianRules.pm:77-97`), dpkg-genbuildinfo, dpkg-genchanges, a check
command (`DEB_CHECK_COMMAND`, `:384,1007-1009`), OpenPGP signing via
Dpkg::OpenPGP backends (sq/sqop/rsop/gosop/gpg…, `debian/control:131-133`),
`getconf` (nproc, `scripts/Dpkg/SysInfo.pm:46-49`). After signing it rewrites
checksums inside `.buildinfo` and `.changes` itself (`:1020-1051`,
`update_files_field` regex surgery at `:1125-1134`). Heavy inline logic.

**dpkg-buildtree** — `clean` (removes `debian/files`, `debian/substvars`,
`debian/tmp`… `scripts/Dpkg/BuildTree.pm:83-96`) and `is-rootless`
(`BuildTree.pm:111-122`). Thin.

**dpkg-checkbuilddeps** — evaluates Build-Depends/-Conflicts(-Arch/-Indep) +
vendor builtin deps (`:138-154`) against `$admindir/status` parsed directly with
regexes (`:193-229`, paragraph mode). Spawns gcc + dpkg (host arch at `:101`).
50-55 ms (D). Logic split: inline status parser; Dpkg::Deps does evaluation.

**dpkg-distaddfile** — adds an entry to `debian/files` under an fcntl lock on
`debian/control` (`:83-100`). Thin.

**dpkg-genbuildinfo** — writes `../<src>_<ver>_<arch>.buildinfo` and registers
it in `debian/files` (locked, `:601-632`). Reads control, changelog (twice,
binNMU, `:464-469`), `debian/files`, all artifacts (checksums), `$admindir/status`
(own parser, `:95-146`), environment allow-list (`:327-351`), buildflags config
(`:337-347`). When cross-building it **compiles and executes** a test program with
`<triplet>-gcc` (`:265-315`). 100 ms (D).

**dpkg-genchanges** — writes `.changes` (stdout or `-O`). Reads control,
changelog (2 parses), `debian/files`, substvars, `../*.dsc` and every artifact
(checksums), `-C` file. Decides orig inclusion from version comparison
(`:366-400`), wraps `Binary` at 980 chars (`:571-575`), formats descriptions with
`%-.65s` on decoded UTF-8 (`:209-223`). No external commands besides arch. 43-45 ms.

**dpkg-gencontrol** — writes `<pkgdir>/DEBIAN/control` and updates
`debian/files` (locked, `:478-517`). Reads control, changelog (+previous entry for
binNMU, `:192-200`), `debian/substvars` or `-T` files, walks the package tree for
Installed-Size (`:415-445`). Simplifies dependency fields (`:321-360`). 45-47 ms.

**dpkg-gensymbols** — generates/compares `DEBIAN/symbols`. Reads changelog,
control, reference symbols from `debian/<pkg>.symbols.<arch>` etc. (`:222-229`),
scans lib dirs for ELF (`:242-264`), runs `objdump -w -f -p -T -R` per library
(`scripts/Dpkg/Shlibs/Objdump/Object.pm:99-102`, cross `<triplet>-objdump`
`:74-81`), `c++filt` for `(c++)` tags (`scripts/Dpkg/Shlibs/Cppfilt.pm:67-68`),
`diff -u` for the report (`:399`). Exit code = check level (`:323-351`,
`DPKG_GENSYMBOLS_CHECK_LEVEL` `:194-196`).

**dpkg-mergechangelogs** — 3-way merge of changelogs; uses
`Algorithm::Merge` if installed (Recommends `libalgorithm-merge-perl`,
`debian/control:135`) else a crude fallback (`:34-45`). Pure Perl, no spawns.

**dpkg-name** — renames `.deb` files to `<pkg>_<ver>_<arch>.deb`; reads fields
via `dpkg-deb -f` (`:119`), moves with `mv`/`ln -s` (`:233-248`).

**dpkg-parsechangelog** — prints changelog entries (deb822 or `-S field`);
custom formats via Perl plug-ins (I). Pure Perl (compressed input decompressed
via external gzip/xz when not `debian/changelog`, `scripts/Dpkg/Changelog/Parse.pm:151`).
27 ms; the 17,814-line / 456-entry repo changelog is not fully parsed (parser
`abort_early`, `scripts/Dpkg/Changelog/Debian.pm:161`).

**dpkg-scanpackages** — builds a Packages file from a directory of `.deb`s:
one `dpkg-deb --info <deb> control` per file (`:202-211`; that C call itself
spawns `tar` and `rm`), checksums in Perl (Dpkg::Checksums), override files
(possibly compressed). Measured: 200 copies of a tiny .deb → 549-567 ms, 200×
dpkg-deb + 200× tar + 200× rm.

**dpkg-scansources** — Sources file from `.dsc` files; pure Perl (except
decompressors for overrides); each `.dsc` processed inside `eval` (`:339-348`).

**dpkg-shlibdeps** — computes `shlibs:*` substvars. Inputs: the ELF files
(format sniffed in Perl, then `objdump`), `/etc/ld.so.conf{,.d}` and
`LD_LIBRARY_PATH` (`scripts/Dpkg/Shlibs.pm:106-143`), `debian/shlibs.local`,
`/etc/dpkg/shlibs.{override,default}` (`:71-72`), `/etc/dpkg/symbols/*`
(`:922-923`), `debian/*/DEBIAN/{symbols,shlibs}` (`:163-168`), installed packages'
symbols via `dpkg-query --control-path` (`scripts/Dpkg/Path.pm:287-290`, an
option `doc/README.feature-removal-schedule` lists as deprecated), file→package
via one batched `dpkg-query --search` whose text output it parses
(`:1039-1066`). Writes `debian/substvars` (`:609-646`). Measured 228 ms for one
trivial binary: ≈69 ms is `dpkg-query --search` (C, loads 2215 `.list` files),
≈10 ms `--control-path`, ≈41 ms parsing the 5,228-line libc6 symbols file in
Perl (65.2 ms vs 23.7 ms module-only), ≈35 ms startup, rest symbol lookup.

**dpkg-source** — build/extract/`--before-build`/`--after-build`/`--commit`/
`--print-format`. Reads `debian/source/{format,options,local-options}` (options
files are filtered then prepended to argv, `:137-154`), changelog, control,
`debian/tests/control` (Testsuite fields, `:557-608`). Spawns GNU tar
(`scripts/Dpkg/Source/Archive.pm:72-84,166-173`), compressors (xz/gzip/bzip2),
`diff -u` and GNU patch (`scripts/Dpkg/Source/Patch.pm:130,654,720`), OpenPGP
verifiers, `git`/`bzr` for 3.0 (git|bzr), `chmod -R`
(`scripts/Dpkg/Source/Functions.pm:86`), editor for `--commit`
(`scripts/Dpkg/Source/Package/V2.pm:823-828`). Measured: `-b` (native, tiny) 73 ms
(perl + tar + xz), `--before-build` 57 ms, `-x` (unsigned, tiny): perl + tar +
unxz + chmod + 2× rm. Sets `SOURCE_DATE_EPOCH` from changelog (`:258-259`).

**dpkg-vendor** — `--is/--derives-from/--query` against
`$CONFDIR/origins/*` or `DPKG_ORIGINS_DIR` (`scripts/Dpkg/Vendor.pm:67-68`). 17 ms.

---

## B. Tool call graph

### B.1 dpkg-buildpackage sequence (code, `scripts/dpkg-buildpackage.pl`)

1. load `buildpackage.conf` options (`:430-434`); parse argv (`:444-633`)
2. hook `preinit` (`:680`); set parallel/MAKEFLAGS (`:682-697`)
3. optional `dpkg-source --extract <dsc>` when given a `.dsc` (`:702-744`)
4. parse changelog + control (`:746-747`); `SOURCE_DATE_EPOCH` (`:777`)
5. `dpkg-architecture -f [arch opts]` → env (`:785-792`)
6. OpenPGP key handling (`:806-850`); optional `sanitize-environment` vendor hook (`:853-855`)
7. build driver from `Build-Driver` field (default `debian-rules`) (`:857-863`)
8. hook `init`; driver `pre_check` (chmod +x debian/rules) (`:869-871`)
9. `dpkg-source --before-build .` (`:873-875`)
10. `dpkg-checkbuilddeps [-A|-B|-I|--admindir]` (`:877-893`, exit 3 on failure)
11. (`-T/--rules-target` mode: run targets and exit, `:895-898`)
12. hook `preclean`; `debian/rules clean` (`:900-906`)
13. hook `source`; `dpkg-source -b .` (`:908-919`)
14. hook `build`; `debian/rules build[-arch|-indep]` only if root is needed for binary (`:925-936`)
15. hook `binary`; `[fakeroot] debian/rules binary[-arch|-indep]` (`:938-945`)
16. hook `buildinfo`; `dpkg-genbuildinfo` (`:962-967`)
17. hook `changes`; `dpkg-genchanges` (`:980-986`)
18. hook `postclean`; optional `debian/rules clean` (`:988-994`)
19. `dpkg-source --after-build .` (`:996`)
20. hook `check`; `$DEB_CHECK_COMMAND ... <changes>` (`:1000-1009`)
21. hook `sign`; sign .dsc → re-checksum .buildinfo → sign .buildinfo → re-checksum .changes → sign .changes (`:1016-1054`)
22. hook `done` (`:1067`)

### B.2 Observed in a real build (strace, minimal dh package)

Direct children of dpkg-buildpackage, in order (from `scratchpad/dbp/strace-noinc.txt`):
`dpkg-architecture -f` → `dpkg-source --before-build .` → `dpkg-checkbuilddeps` →
`debian/rules clean` → `dpkg-source -b .` → `debian/rules binary` →
[dh_shlibdeps → `dpkg-shlibdeps -Tdebian/hello.substvars …` → `dpkg-query --search -- …`,
`dpkg-query --control-path libc6:amd64 symbols`, `objdump`] →
[dh_gencontrol → `dpkg-gencontrol -phello -ldebian/changelog -Tdebian/hello.substvars -cdebian/control -Pdebian/hello`] →
[dh_builddeb → `dpkg-deb --root-owner-group --build debian/hello ..`] →
`dpkg-genbuildinfo -O../hello_1.0_amd64.buildinfo` → `dpkg-genchanges -O../hello_1.0_amd64.changes` →
`dpkg-source --after-build .`

Successful execs in the whole build: **133** without `include default.mk`,
**221** with it (40× dpkg-buildflags, 8× dpkg-buildapi, +40 `sh`), and with two
`override_dh_*` targets added: 80× dpkg-buildflags, 12× dpkg-buildapi, 4
`debian/rules` invocations. See C for the mechanism.

### B.3 Who calls the C binaries

| Perl tool / component | C binary call | where |
|---|---|---|
| everything needing build arch (dpkg-architecture, -buildflags, -checkbuilddeps, -buildapi, -gencontrol, -genbuildinfo, -gensymbols, -shlibdeps …) | `dpkg --print-architecture` | `scripts/Dpkg/Arch.pm:138` |
| dpkg-shlibdeps | `dpkg-query --search -- <files>` (output parsed) | `dpkg-shlibdeps.pl:1039-1066` |
| dpkg-shlibdeps (via Dpkg::Path) | `dpkg-query --control-path <pkg> symbols` | `scripts/Dpkg/Path.pm:287-290` |
| dpkg-scanpackages | `dpkg-deb --info <deb> control` (per file) | `dpkg-scanpackages.pl:202-211` |
| dpkg-name | `dpkg-deb -f -- <deb>` | `dpkg-name.pl:119` |
| debhelper (not dpkg) | `dpkg-deb --build` | `/usr/bin/dh_builddeb:138,152` |
| dselect methods | `dpkg --merge-avail/--clear-avail/-iB/-iGREOB/--predep-package/--print-architecture` | `dselect/methods/ftp/update.pl:50,239,249,59`, `ftp/install.pl:573-588`, `file/install.pl:53,127`, `media/install.pl:84,173` |

**Leaf tools** (spawn nothing but perl, measured): dpkg-parsechangelog,
dpkg-vendor, dpkg-distaddfile, dpkg-mergechangelogs, dpkg-buildtree,
dpkg-scansources (modulo decompressors), `dpkg-source --before/--after-build`.
**Leaf except arch detection** (gcc/dpkg): dpkg-buildflags, dpkg-buildapi,
dpkg-checkbuilddeps, dpkg-gencontrol, dpkg-genchanges, dpkg-genbuildinfo,
dpkg-gensymbols (no lib in test → no objdump). No C program in `src/`, `lib/`,
`utils/`, `dselect/` invokes perl (grep of `*.c/*.cc/*.h` for `perl` and the
dpkg-dev tool names returned nothing relevant).

---

## C. `scripts/mk/*.mk` (public packager interface)

| fragment | lines | provides | how values are obtained |
|---|---:|---|---|
| `default.mk` | 23 | includes all others; `buildtools.mk` only on build API ≥ 1 (`default.mk:13-21`, `:15`) | — |
| `architecture.mk` | 21 | 33 `DEB_{BUILD,HOST,TARGET}_*` vars, **exported**, `?=` (env wins) (`:15-19`) | one `dpkg-architecture -q<VAR>` per variable, lazily, cached in `DPKG_CACHE_<VAR>` (`:13`) |
| `buildapi.mk` | 16 | `DPKG_BUILD_API`, `dpkg_build_api_ge` (`:12-14`) | `DPKG_BUILD_API ?= $(shell dpkg-buildapi)` — **recursively expanded, not cached** |
| `buildflags.mk` | 78 | 20 flags (10 + `_FOR_BUILD`) (`:43-54`), exported only with `DPKG_EXPORT_BUILDFLAGS` (`:74-76`) | one `dpkg-buildflags --get <FLAG>` per flag, lazily cached (`:41,69`); forwards `DEB_BUILD_OPTIONS`, `DEB_*_MAINT_*` (`:56-67`) |
| `buildopts.mk` | 23 | `DEB_BUILD_OPTION_PARALLEL` | pure make (`:18-21`) |
| `buildtools.mk` | 87 | AS CC CXX … PKG_CONFIG QMAKE + `_FOR_BUILD` | pure make, needs `architecture.mk` (`:38,44-86`) |
| `pkg-info.mk` | 66 | `DEB_SOURCE`, `DEB_VERSION*`, `DEB_DISTRIBUTION`, `DEB_TIMESTAMP`, exports `SOURCE_DATE_EPOCH` (`:53-64`) | one `dpkg-parsechangelog -S<Field>` per field (+`echo|sed` for version parts), cached |
| `vendor.mk` | 62 | `DEB_VENDOR`, `DEB_PARENT_VENDOR`, `dpkg_vendor_derives_from{,_v0,_v1}` (`:50-59`) | `dpkg-vendor --query …`, cached; derives_from spawns per call |

GNU make requirements: `$(intcmp)` needs GNU make ≥ 4.4 (`buildapi.mk:14`;
`debian/control:123-124` "Version needed for GNU make intcmp function usage");
`undefine` (make ≥ 3.82) is emitted by `dpkg-architecture -u --print-format make`
(`dpkg-architecture.pl:472-475`); `flavor`, `value`, `eval`, `lastword` (≥ 3.81).
Caching trick: `dpkg_lazy_eval`/`dpkg_late_eval` = "if `DPKG_CACHE_x` undefined,
`$(eval DPKG_CACHE_x := $(shell …))`, return `$(value DPKG_CACHE_x)`" — one spawn
per variable per make process, never shared across processes.

### C.1 Measured spawns per make invocation (test pkg = `scripts/t/mk/debian`, no-op recipe)

| environment | target | spawns | wall |
|---|---|---|---|
| empty (e.g. `debian/rules clean` by hand) | `architecture.mk` only, `noop` | 66 = 33 dpkg-architecture + 22 gcc + 11 dpkg | 726 ms |
| empty | `default.mk`, `noop` | 69 (+2 dpkg-buildapi, +1 dpkg-parsechangelog for exported `SOURCE_DATE_EPOCH`) | 812 ms |
| empty | `default.mk`, uses 7 vars | 75 | 1172 ms |
| arch env exported (as dpkg-buildpackage does) | `default.mk`, `noop` | 3 | 116 ms |
| arch env + `SOURCE_DATE_EPOCH` (≈ inside dpkg-buildpackage) | `noop` / 2 vars / 7 vars | 2 / 3 / 8 | — |
| same + `DPKG_BUILD_API=0` | `noop` | 0 | — |
| same + `CFLAGS` in env | `noop` | +1 dpkg-buildflags per flag present in env | — |

Why all 33 arch vars are spawned for a no-op: they are `export`ed recursive
variables, and exporting requires expansion when make builds a child environment
(`architecture.mk:15`). Why dh builds pay 20× dpkg-buildflags per make process:
debhelper exports all 20 flags into the environment before invoking make
(env diff of the `make -Rrnpsf debian/rules debhelper-fail-me` child shows
`CFLAGS`…`OBJCXXFLAGS_FOR_BUILD`, `DH_INTERNAL_BUILDFLAGS`;
introspection code `/usr/share/perl5/Debian/Debhelper/SequencerUtil.pm:235`). A
variable inherited from the environment is exported, so the makefile's lazy
`CFLAGS = $(call dpkg_lazy_eval,…)` (`buildflags.mk:69`) becomes an exported
recursive variable and is expanded for every child (verified: `CFLAGS=-O2 make
-f rules-default noop` → +1 dpkg-buildflags). `dpkg-buildapi` runs twice per make
process because `DPKG_BUILD_API` is recursive (`buildapi.mk:12`) and is expanded in
`default.mk:15` and again in `vendor.mk:56`; nothing exports `DPKG_BUILD_API`
(grep of `scripts/` finds only the reader, `scripts/Dpkg/BuildAPI.pm:73`).

Cost in the minimal dh build (3 runs each, pinned to P-cores):
**1612/1611/1598 ms** without the include vs **2819/2845/2826 ms** with
`include default.mk`; **4077 ms** (1 run) with two `override_dh_*` targets.
≈1.2 s per such build is mk-fragment-driven Perl startup: 40 × ~23 ms
(dpkg-buildflags) + 8 × ~28 ms (dpkg-buildapi), consistent with D. Each
`override_*` target adds another `debian/rules` make process (+20 buildflags,
+2 buildapi).

Evidence both ways for a rewrite here: a native `dpkg-buildflags`/
`dpkg-buildapi` with ~1 ms startup would cut ≈1.1 s off this build; but most of
the waste is structural and fixable in make (export `DPKG_BUILD_API` from
dpkg-buildpackage or make it `:=`-cached; batch flag queries with one
`dpkg-buildflags --export=make` read; avoid re-spawning for env-provided values) —
(unverified: I did not try such a patch). How many archive packages include
these fragments is not measured (no archive access).

---

## D. Measured performance

### D.1 Startup (pinned to CPU 4, P-core; n=40, ms; `scratchpad/timing/bench-pcore.txt`)

| command (cwd = repo root; in-tree unless marked) | median | min | p90 | Dpkg mods / total `%INC` |
|---|---:|---:|---:|---|
| `/bin/true` (harness overhead) | 0.24 | 0.23 | 0.25 | — |
| `perl -e 1` | **0.72** | 0.71 | 0.76 | 0 / 0 |
| `perl -MDpkg::ErrorHandling -MDpkg::Gettext -MDpkg::Getopt -e1` | 13.34 | 13.23 | 13.41 | — |
| `perl -MDpkg::Control -e1` | 18.67 | 18.49 | 18.91 | — |
| `dpkg-architecture` | **17.83** | 17.51 | 18.13 | 7 / 32 |
| `dpkg-architecture -qDEB_HOST_MULTIARCH` | **16.88** | 16.70 | 17.48 | 7 / 32 |
| … with `DEB_HOST_ARCH` set | 16.85 | 16.67 | 17.01 | |
| … with `DEB_HOST_MULTIARCH` set | 15.66 | 15.42 | 15.90 | |
| `dpkg-buildflags` | 25.32 | 24.78 | 25.73 | 18 / 45 |
| `dpkg-buildflags --get CFLAGS` | **24.94** | 24.72 | 25.76 | 18 / 45 |
| `dpkg-parsechangelog` (repo changelog) | **27.17** | 26.84 | 27.91 | 27 / 58 |
| `dpkg-parsechangelog -SVersion` | 27.45 | 27.20 | 27.92 | |
| `dpkg-vendor --query Vendor` | **17.41** | 17.22 | 18.02 | 12 / 38 |
| `dpkg-checkbuilddeps` (repo control; deps satisfied, rc 0) | **55.48** | 54.58 | 56.45 | 33 / 69 |
| `dpkg-buildapi` (repo control) | **32.48** | 31.85 | 32.83 | 34 / 68 |
| `dpkg-source --version` | 39.86 | 39.32 | 40.28 | 45 / 84 (V) |
| `dpkg-buildpackage --version` | 38.09 | 37.72 | 38.64 | 39 / 80 (V) |
| `gcc -dumpmachine` | 0.42 | 0.41 | 0.44 | |
| `/usr/bin/dpkg --print-architecture` (system C) | 0.48 | 0.47 | 0.49 | |
| `build/src/dpkg --version` (in-tree C; real ELF, no libtool wrapper) | **0.45** | 0.43 | 0.48 | |
| `build/src/dpkg-query --version` | **0.34** | 0.32 | 0.35 | |
| `build/src/dpkg --print-architecture` | 0.44 | 0.43 | 0.45 | |
| `/usr/bin/dpkg-architecture -qDEB_HOST_MULTIARCH` (**system** 1.23.7ubuntu1) | 16.44 | 16.27 | 16.63 | 6 / 31 |
| `/usr/bin/dpkg-buildflags --get CFLAGS` (**system**) | 39.69 | 39.19 | 40.89 | 32 / 72 |

Ratio: the cheapest Perl tool (dpkg-architecture, ~17 ms) is ~37× a C binary's
startup (0.45 ms) and ~23× bare `perl -e 1`.

### D.2 Where Perl startup goes (pinned, n=30; `bench-mods.txt`, `bench-nls.txt`)

Single-module load (`perl -M<mod> -e1`): POSIX 7.21, Encode 8.83,
**Locale::gettext 12.12**, Storable 5.99, List::Util 2.76, Dpkg 1.40,
Dpkg::Gettext 12.40, Dpkg::ErrorHandling 13.01, Dpkg::Arch 14.25,
Dpkg::Version 14.09, Dpkg::Vendor 16.61, Dpkg::Deps 18.14, Dpkg::Control 18.69,
Dpkg::Changelog::Parse 19.00, Dpkg::Shlibs::SymbolFile 23.07,
Dpkg::Source::Package 32.08.

`Dpkg::Gettext` loads `Locale::gettext` (which uses POSIX and Encode) unless
`DPKG_NLS=0` (`scripts/Dpkg/Gettext.pm:123-130`). With `DPKG_NLS=0`:
dpkg-architecture -q 16.97 → **8.11**; dpkg-buildflags --get 24.98 → 17.95;
dpkg-vendor 17.25 → 10.07; dpkg-parsechangelog -S 27.13 → 20.51; dpkg-buildapi
31.96 → 24.61 (and dpkg-architecture's `%INC` drops from 32 to 15 files).
⇒ ~7-9 ms of every tool's startup is the gettext/POSIX/Encode stack — a
Perl-side optimisation (lazy loading) is available independently of any rewrite.

### D.3 Heavier operations (pinned, n=20; cwd = test package; `bench-heavy.txt`, `bench-chain.txt`)

| command | median ms (no arch env / arch env) |
|---|---|
| `dpkg-shlibdeps -O debian/hello/usr/bin/hello` | 227.7 / 228.9 |
| `dpkg-gencontrol -phello -Pdebian/hello -O` | 47.4 / 45.5 |
| `dpkg-genchanges -O/dev/null` | 45.1 / 43.2 |
| `dpkg-genbuildinfo -O` | 102.5 / 99.5 |
| `dpkg-source -b .` (native) | 72.7 |
| `dpkg-source --before-build .` / `--after-build .` | 57.3 / 57.6 |
| `dpkg-checkbuilddeps` (hello) | 50.7 |
| `dpkg-architecture -f` | 17.9 |
| `objdump -w -f -p -T -R hello` | 0.92 |
| `dpkg-query --search` (2 libs, system C) | 68.8 |
| `dpkg-query --control-path libc6:amd64 symbols` | 10.3 |
| parse libc6 symbols (5,228 lines) with Dpkg::Shlibs::SymbolFile | 65.2 (vs 23.7 load only) |

Sum of isolated medians of the dpkg-buildpackage chain in B.2 ≈ 0.71 s of the
1.6 s minimal build (estimate; in-build times not measured individually).

Unpinned reference (n=50, `bench-run2.txt`): dpkg-architecture 28.2,
-q 25.8, buildflags --get 31.0, parsechangelog 30.4, vendor 24.2,
checkbuilddeps 66.6, buildapi 39.1 ms.

---

## E. Perl everywhere else, classified

| item | class | what it does | replaceable by |
|---|---|---|---|
| `configure` → `DPKG_PROG_PERL` (`configure.ac:105`, `m4/dpkg-progs.m4:56-84`) | **build-time (tarball)** | hard `AC_MSG_ERROR` without perl ≥ 5.36, unconditionally | making it conditional on building dpkg-dev/docs (moderate autotools work) |
| `build-aux/subst.am` (`:7-25, :29-49`) | build-time | `perl -p -e` substitution of `src/*.sh` (→ `dpkg-maintscript-helper`, `dpkg-db-backup`, `dpkg-db-keeper` shipped in the **Essential** `dpkg`), `scripts/*.pl`, `dselect/methods/*`; `Dpkg.pm` at install (`scripts/Makefile.am:198-199`) | `sed` (trivial; the rules are simple `s{}{}`), verified effect: `build/src/dpkg-maintscript-helper` differs from source only in `version=` and `PKGDATADIR_DEFAULT=` |
| `utils/Makefile.am:51` polkit subst | build-time | `perl -p -e` on `update-alternatives.polkit.in` | sed (trivial) |
| `dselect/mkcurkeys.pl` (148 lines; `dselect/Makefile.am:39-40`) | build-time (needed to compile dselect) | parses `<curses.h>` KEY_* defines + `keyoverride` → `curkeys.h` | small Python/awk/C program, or checking in the generated table per ncurses version (S) |
| `man/Makefile.am:242-283` | build-time | `pod2man` (Perl/podlators) + `perl -E` for the section; generated `.1/.5/.7/.8` are **not distributed** (`man/Makefile.in:186` `DISTFILES` excludes `man_MANS`; `CLEANFILES` `:418`) | another POD renderer or switching doc source format (L: 63 pages, 19,300 lines of POD) |
| `man/Makefile.am:213-227` po4a | build-time (if NLS + po4a) | translated man pages (13 langs) | po4a has no non-Perl equivalent of comparable scope (unverified) |
| `scripts/Makefile.am:180-196` | install-time | `pod2man` → 92 `Dpkg::*.3perl` pages | n/a while modules stay Perl |
| autotools from git (`autogen` → `autoreconf`) | build-time (git) | `automake`, `aclocal`, `autoreconf`, `autom4te` are Perl scripts (`head -1 /usr/bin/automake` etc.) | only by leaving autotools |
| `build-aux/get-version`, `get-vcs-id`, `gen-release` | release/build | shell (not Perl); `gen-release` calls `dpkg-parsechangelog`, `make update-po`, `make dist-cpan` (`build-aux/gen-release:36-43,74,99`) | — |
| `build-aux/gen-changelog` (314 lines) | maintainer/release | renders git log into `debian/changelog` using in-tree `Dpkg::IPC`, `Dpkg::Index` (`:20-40,166,179`) | any language (S) |
| `build-aux/cpan.am`, `scripts/Build.PL.in`, `README.cpan.in` | release | CPAN tarball of libdpkg-perl (`dist-cpan`) | n/a (is Perl packaging) |
| `build-aux/lcov-inject` (`Makefile.am:144`) | coverage only | merges Devel::Cover data | — |
| `build-aux/test-runner` (TAP::Harness) via `build-aux/tap.am:21-34` | **test-time, all suites** | runs every TAP test incl. **37 C unit-test programs** of `lib/dpkg/t` (`lib/dpkg/Makefile.am:227-324`), so `make check` of the C library needs Perl | automake's `tap-driver.sh` (sh+awk) or a small native runner (S) (unverified) |
| `build-aux/get-make-jobs` (`tap.am:13`, `autotest.am:8`) | test-time | computes `TEST_PARALLEL` from `MAKEFLAGS` via Dpkg::SysInfo | shell (S) |
| `build-aux/run-script` | dev/test | sets `PERL5LIB`/`DPKG_DATADIR` for in-tree runs | — |
| `t/*.t` (14 files) | test-time (mostly author-only) | POD/strict/critic/syntax/taint/codespell/shellcheck/cppcheck/i18nspector drivers; `test_needs_author()` gates 10 of them | keep (they test Perl) or move to CI scripts |
| `scripts/t/*.t` (51 files, 7,944 lines) + `scripts/Test/Dpkg.pm` (430) | test-time | 46 module tests + `dpkg_source.t`, `dpkg_buildpackage.t`, `dpkg_buildtree.t`, `dpkg_mergechangelogs.t`, `mk.t` (program-level coverage is thin) | — |
| `lib/dpkg/t/t-{tarextract,treewalk,trigdeferred}.t` (454 lines) | test-time | Perl drivers for C helper programs | any language (S-M) |
| `utils/t/update_alternatives.t` (863 lines) | test-time | the update-alternatives (C) test suite is Perl | any language (M) |
| `src/at/*.at` (autotest, 58 `AT_SETUP`) | test-time | `$PERL` one-liners (`local.at:47,52,176`, `deb-split.at:18,45`, `deb-format.at:303`) and `dpkg-ar.pl` via `DPKG_AR*` (deb-format 49, deb-split 18, local 11, deb-content 2) | shell + `ar`/`dpkg-deb`, or a tiny native ar tool (S) |
| `scripts/t/Dpkg_Shlibs/spacesyms-{c-gen,o-map}.pl` | test data generation | create test objects | — |
| `debian/rules:18-22,39-42` | **packaging-time** | dpkg's own package build uses in-tree `dpkg-buildflags` and `dpkg-vendor` through `run-script`; build also uses debhelper (Perl) | n/a |
| `scripts/*.pl`, `scripts/mk/*.mk` | **run-time**, package `dpkg-dev` (Arch: all) | `debian/dpkg-dev.install:5-23`; Depends `${perl:Depends}`, `libdpkg-perl (=)`, bzip2, xz-utils, patch ≥2.7, make ≥4.4, binutils (`debian/control:114-125`) | see H |
| `scripts/Dpkg/**` | run-time, `libdpkg-perl` | `debian/libdpkg-perl.install`; also ships `dpkg-dev.mo` and `*.specs` | (other analyst) |
| `dselect/methods/{file,media}/install.pl`, `ftp/{setup,update,install}.pl`, `Dselect/Method{,/Config,/Ftp,/Media}.pm` (2,198 lines Perl); `media/setup.sh:186` runs `perl -ne` | **run-time**, package `dselect` | installed to `/usr/lib/dpkg/methods` and `/usr/share/perl5/Dselect` (`debian/dselect.install`); `dselect` only **Suggests** `perl, libdpkg-perl` (`debian/control:252-262`) although the methods `use Dpkg::Gettext/ErrorHandling/File` and `Net::FTP` | shell (file/media) or any language (ftp: FTP client, M); ftp state is Data::Dumper Perl code `eval`ed back (`Dselect/Method.pm:104,124`, `ftp/install.pl:67-75,639`) |
| Essential `dpkg` package | run-time | **contains no Perl**: `debian/dpkg.install` lists only C binaries and `/bin/sh` scripts (shebangs checked) | — |

`perl-base` is itself Essential on this system (`dpkg-query`: `perl-base
Essential=yes`); `dpkg-dev` depends on full `perl:any` (system control field).

---

## F. i18n

| domain (dir) | source | languages | msgids in .pot | plural / msgctxt | xgettext keywords |
|---|---|---:|---:|---|---|
| `dpkg` (`po/`) | C (lib, src, utils) + polkit XML (`po/POTFILES.in`, 116 entries) | 44 | 1,323 | 8 / 11 | `--keyword --keyword=_ --keyword=N_ --keyword=P_:1,2 --keyword=C_:1c,2` (`po/Makevars:18-20`) |
| `dpkg-dev` (`scripts/po/`) | Perl: 20 scripts + 91 modules (111 entries) | 11 | 916 | 5 / 3 | `--keyword --keyword=g_ --keyword=N_ --keyword=P_:1,2 --keyword=C_:1c,2` (`scripts/po/Makevars:11-13`) |
| `dselect` (`dselect/po/`) | C++ + 7 Perl method files listed | 31 | 263 | 0 / 2 | same as `dpkg` (`_`) |
| `dpkg-man` (`man/po/`, po4a) | 62 POD pages (all but `libdpkg.pod`) | 13 | 3,744 | — | po4a `[type:pod]` (`man/po/po4a.cfg`) |

* C marking: `_()`=gettext, `N_()`=noop, `P_()`=ngettext, `C_()`=pgettext
  (`lib/dpkg/i18n.h:40-43`); occurrences in C/C++: `_(` 1,792, `N_(` 213, `P_(` 9, `C_(` 23.
* Perl marking: `g_()`, `P_()`, `C_()` (context glued with `\004`), `N_()`
  (`scripts/Dpkg/Gettext.pm:121-189`); occurrences: `g_(` 1,192, `N_(` 51, `P_(` 6, `C_(` 6.
  `DPKG_NLS=0` disables gettext entirely (`Gettext.pm:124`).
* Extraction: standard gettext `Makevars`/`Makefile.in.in`; xgettext's Perl parser
  handles the `"…\n" . "…\n" . ''` concatenations used for help text. 462
  dpkg-dev entries carry `#, perl-format`, 848 dpkg entries `#, c-format`.
* dpkg-dev msgids by origin (parsed from `#:` references): 433 only in programs,
  464 only in modules, 19 shared.
* printf directives used in dpkg-dev msgids: `%s` 619, `%d` 39, `%u` 4, `%04o` 2,
  `%0x` 1, `%%` 2 — all C-compatible. Translations use positional reordering
  `%1$s` (e.g. `scripts/po/ca.po` 8×, `po/ko.po` 40×; 21 `.po` files overall).
* dselect: the 7 Perl method files are in `dselect/po/POTFILES.in` but the only
  marked string (`Dselect/Method/Config.pm:146`, `g_()`) is not extracted (keyword
  `g_` not configured) — `dselect.pot` has 0 references to `dselect/methods`; the
  methods are effectively untranslated.
* Completeness (`msgfmt --statistics`, % translated): dpkg-dev — pt, sv, uk 100;
  nl 98; ca, de, en 95; fr 83; pl, ru 59; es 54. dpkg — 100: nl, pt, ro, sv, uk;
  ≥90: ca, cs, de, en, fr, pt_BR, ru, th, zh_CN; many below 50 (44 total).
  dselect — 31 langs, mostly 76-100. man — sv 84, nl 84, pt 83, de 81, fr 66,
  it 32, es/ja/pl 29, zh_CN 3, ru 2, hu 1, pt_BR 1.
* **Effect of moving strings** (gettext semantics): runtime lookup is by exact
  msgid (+msgctxt) within the text domain; `#:` locations are irrelevant. A
  rewrite that emits byte-identical msgids under domain `dpkg-dev` keeps all
  existing `.mo` translations working. Any change in text — including the
  per-option help blocks with exact indentation (e.g. `"      --version\n
  Show the version.\n"`, `scripts/po/dpkg-dev.pot:101-104`) or a switch to a CLI
  framework that formats help itself — turns those entries fuzzy/obsolete on the
  next `msgmerge`. Re-extraction from Rust: the installed xgettext 0.23.2 lists
  no Rust language (`xgettext --help`), so extraction tooling would need
  attention; flags would change from `perl-format`. Runtime formatting must
  accept runtime printf-style strings with POSIX positional arguments. Merging
  dpkg-dev strings into the C `dpkg` domain is mechanically possible (msgcat)
  but changes translator workflows for 11 languages.

## G. Man pages and docs

* `man/*.pod`: 63 sources, 19,300 lines; man_MANS: 28 section-1 programs,
  27 section-5 formats, 4 section-7 (`deb-email`, `deb-version`, `dpkg-build-api`,
  `libdpkg`), plus `dselect.1`, `dselect.cfg.5`, `start-stop-daemon.8`,
  `update-alternatives.1` (conditional). Pages for the 19 installed Perl programs:
  all present (`dpkg-ar` has none). Largest: dpkg-source 1,234, dpkg-buildflags
  1,231, dpkg-buildpackage 1,064, dpkg-architecture 708, dpkg-shlibdeps 565,
  deb-src-symbols 470 lines.
* Build: `%VAR%` substitution with sed (`man/Makefile.am:242-260`) → `pod2man
  --utf8 --center='dpkg suite'` (`:235-272`) → `utf8toman.sed`; only if
  `pod2man` found (`BUILD_POD_DOC`, `m4/dpkg-progs.m4:116-121`). Translations via
  po4a into per-language dirs (`:188-215`), `.add` addenda for translator credits
  (9 files).
* Perl module docs are **not** in `man/`: each module's own POD becomes
  `Dpkg::*.3perl` at install (92 modules incl. `Dpkg.pm`,
  `scripts/Makefile.am:180-196`, errors on empty output); not translated.
* `doc/`: `Doxyfile.in` (2,807 lines; `INPUT = lib/dpkg src utils`,
  `OPTIMIZE_OUTPUT_FOR_C`, HTML only, `make doc` at `Makefile.am:120-121`) —
  C only; `README.api` (libdpkg.a "volatile", **libdpkg-perl "stable"**, custom
  changelog parsers as Dpkg::Changelog subclasses "stable since 1.18.8",
  `doc/README.api:1-35`); `README.feature-removal-schedule` (183 lines);
  `coding-style.txt` (POD, M4sh, C/C++, Perl styles, `:1,17,71,281`);
  `spec/` (`build-driver.txt` draft/experimental, `frontend-api.txt` stable,
  `protected-field.txt` draft, `rootless-builds.txt` stable, `triggers.txt` stable).

---

## H. Rewrite-candidate ranking (programs)

Frequency evidence: minimal dh build with `default.mk` = 40 dpkg-buildflags, 8
dpkg-buildapi, 3 dpkg-source, 1 each of the rest; +20/+2 per `override_*`
target; outside dpkg-buildpackage every make process with `architecture.mk`
spawns 33 dpkg-architecture. Per-call cost from D.

| program | LOC (logic) | perf benefit (latency × freq) | robustness benefit (hostile input) | dependency reduction | difficulty | compat risk | suggested priority |
|---|---|---|---|---|---|---|---|
| dpkg-buildflags | 250 (146) + BuildFlags/Vendor ~870 code | **High**: 23-25 ms × 20-80 per dh build | none | low (perl-base Essential) | M (vendor feature matrix, config files, origin tracking, derivative patches) | M-H: must equal `Dpkg::BuildFlags` used in-process by debhelper (`Dh_Lib.pm:2819-2854`); vendor modules are a Perl extension point | 1 (or fix mk first) |
| dpkg-buildapi | 83 (29) | **High**: ~28-32 ms × 2 per make process (8-12 per build) | none | — | S (deb822 + Build-Depends scan) | L | 1 (mk fix is cheaper) |
| dpkg-architecture | 498 (315) + Arch 438 | Med-High: 17 ms × 33 per make process outside dpkg-buildpackage; ×1 inside | none | — | S (table-driven; ERE-compatible table regexes) | L-M (output formats, env-override semantics, `-c`) | 2 |
| dpkg-vendor | 123 (54) | Med (vendor.mk, `derives_from` spawns per call) | none | — | S (origins file) | L, but vendor *modules* stay Perl | 2 |
| dpkg-parsechangelog | 196 (80) + changelog ~780 | Med (pkg-info.mk: 1 spawn per field per make process; widely scripted) | Low-Med (changelogs inside source pkgs) | — | M (Debian changelog grammar quirks; good module tests exist) | M: `-F <fmt>` plug-ins are Perl modules (stable API) → needs Perl fallback | 2-3 |
| dpkg-shlibdeps | 1070 (753) + Shlibs ~1,390 | Med: ~230 ms × #arch-dep packages; ~41 ms is Perl symbols parsing, ~69 ms is C dpkg-query | Low-Med (objdump text of built ELF; replacing it with native ELF parsing removes binutils text-format coupling) | binutils (objdump), c++filt | **L** | **H** (generates every package's Depends; Perl-regex patterns in symbols files) | 4 |
| dpkg-gensymbols | 402 (262) | Low-Med (per library pkg) | Low | binutils | L (shares Shlibs logic) | H (symbols file output/diff consumed by maintainers) | 4 |
| dpkg-gencontrol | 519 (340) | Low-Med: ~46 ms × #binary pkgs (+dbgsym) | Low | — | M-L (substvars, FieldsCore field tables ~1,180 code lines, deps simplification) | H (byte-exact `DEBIAN/control` for reproducibility) | 3 |
| dpkg-checkbuilddeps | 270 (149) | Low: ~51-55 ms × 1 | Low | — | M (dep evaluation, profiles, multi-arch, Provides) | M | 3 |
| dpkg-genbuildinfo | 638 (419) | Low: ~100 ms × 1 | Low | — | M | H (consumed by reproducible-builds tooling; field order/format) | 4 |
| dpkg-genchanges | 628 (380) | Low: ~43 ms × 1 | Low | — | M | H (consumed by archive/upload tools; Binary wrapping, Description formatting) | 4 |
| dpkg-buildpackage | 1214 (737) | Low: ~38 ms startup × 1 | Low | — | L (66 help opts/77 man items, hooks, signing backends, R³ driver) | H (CLI used by sbuild/pbuilder/debuild — unverified here; hook language) | 5 |
| dpkg-source | 808 (479) + Source::* (~2,100 code in Package/V2/Patch/Archive alone) | Low-Med: 57-73 ms × 3 per build | **Highest** (`-x` on downloaded `.dsc`) — but see below | patch, tar stay | **XL** (formats 1.0/2.0/3.0 quilt/native/git/bzr/custom, patch parse+generate, quilt, signatures, Perl-regex diff-ignore) | **XL** | 5 (or a narrowly-scoped hardened extractor) |
| dpkg-scanpackages | 351 (241) | Med for repo tools (2.75 ms/deb, dominated by C dpkg-deb+tar+rm spawns) | Med (untrusted .debs; parsing via C dpkg-deb) | — | S-M | M (Packages field order; `-a`/`-t` are regex fragments) | 3 |
| dpkg-scansources | 358 (239) | Low | Med (untrusted .dsc parsed in Perl) | — | S-M | M | 3 |
| dpkg-name | 289 (187) | Low | Med (see path traversal below; also fixable in Perl) | — | S | L-M (quirky option parsing) | 3 |
| dpkg-mergechangelogs | 328 (224) | Low | Low | Algorithm::Merge (optional) | S-M | L | 5 |
| dpkg-distaddfile | 100 (42) | Low | none | — | S | L (locking protocol on `debian/control`) | 4 |
| dpkg-buildtree | 89 (36) | Low | none | — | S | L | 4 |
| dpkg-ar | 139 (72) | n/a (test-only) | n/a | — | S | none (not installed) | any time |

### H.1 Tools that parse untrusted input

* **dpkg-source -x** (also `dpkg-buildpackage <file.dsc>`, `:702-744`): `.dsc`
  deb822 + OpenPGP armor parsing in Perl (Dpkg::Control::HashCore); checksum
  verification in Perl; patch parsing/validation in Perl
  (`scripts/Dpkg/Source/Patch.pm:402-609`: C-style filenames, insecure paths,
  symlinks, `.dpkg-orig`, multiple patching…); extraction delegated to GNU tar
  with `--no-same-owner --no-same-permissions` into a temp dir
  (`scripts/Dpkg/Source/Archive.pm:166-173`); application delegated to GNU patch
  (`Patch.pm:654,720`; configure insists on GNU patch for traversal resistance,
  `debian/changelog:5369-5374`); signature check delegated to gpgv/sqv/sopv.
  CVE history from `debian/changelog` on this path: CVE-2010-0396 (`:10189`),
  CVE-2010-1679 (`:9586`), CVE-2014-0471 (`:7899`), CVE-2014-3127 (`:7850`),
  CVE-2014-3864/3865 (`:7828`), CVE-2015-0840 (armor parsing, `:6962`),
  CVE-2017-8283 (`:5369`), CVE-2022-1664 (Dpkg::Source::Archive traversal,
  `:2992`) — all logic/path-traversal class. Memory-safety CVEs in the same log
  are on the C side (CVE-2015-0860 dpkg-deb off-by-one write `:6572`,
  CVE-2026-2219 zstd decompression `:504`). Evidence both ways: Perl already gives
  memory safety; the bug class seen here is one a rewrite can re-introduce;
  the heavy lifting (tar, patch, OpenPGP) is external either way.
* **dpkg-scanpackages** (`.deb` → `dpkg-deb --info`, fields then parsed by
  Dpkg::Control), **dpkg-scansources** (`.dsc` parsed directly in Perl, errors
  caught by `eval`, `:339-348`).
* **dpkg-name** (`.deb` → `dpkg-deb -f`): **verified path traversal** — the
  `Package` field is used unvalidated in the destination name; only spaces are
  stripped (`dpkg-name.pl:145-163`). A crafted `.deb` with `Package: ../../escaped`
  (built with tar+ar in the scratchpad; system `dpkg-deb -f` printed it without
  complaint) was moved by in-tree `dpkg-name` to `./../../escaped_1.0_all.deb`
  (`scratchpad/dpkgname/a/escaped_1.0_all.deb`). `Version`/`Architecture` are
  similarly interpolated (`:154`, and `-s` subdirs `:188-190`). Fixable in Perl
  with `pkg_name_is_invalid` as other tools do.
* **dpkg-shlibdeps / dpkg-gensymbols**: parse `objdump` text output of freshly
  built (semi-trusted) binaries and installed symbols files (trusted).
* **dpkg-parsechangelog / dpkg-mergechangelogs / dpkg-gencontrol / -genchanges /
  -checkbuilddeps**: parse `debian/changelog`/`debian/control` of the source being
  built (building already runs arbitrary `debian/rules`, so hostile-input
  hardening matters mainly for tools that only *inspect* sources).
* **dselect ftp method**: downloads `Packages` files and `.deb`s over FTP and
  feeds them to `dpkg --merge-avail`/`dpkg -iB`; grep finds no Release/OpenPGP
  verification, only MD5/size bookkeeping (`ftp/install.pl:143-168`) (absence
  of verification inferred from grep, not from a protocol test).

### H.2 Reasons to leave tools in Perl (evidence)

* `libdpkg-perl` is a documented **stable** API (`doc/README.api:14-28`) with 36
  reverse dependencies in this system's archive metadata (`apt-cache rdepends
  libdpkg-perl`: debhelper, lintian, devscripts, dgit, sbuild (libsbuild-perl),
  autopkgtest, dh-python, pkg-kde-tools, mmdebstrap, reprotest, …). debhelper
  loads Dpkg modules in-process (`Dh_Lib.pm:1287-1290,1898,2086,2206,2748,
  2819-2854,3312`) while also spawning the CLI tools; a native rewrite of a
  program does not remove the Perl implementation, it duplicates it and creates a
  divergence risk (e.g. flags computed by `dpkg-buildflags` vs
  `Dpkg::BuildFlags` inside dh).
* Extension points are Perl modules (I).
* Most logic sits in modules (A.0), which are shared across programs.
* Thin program-level tests (5 program `.t` files, 744 lines) vs 46 module test
  files: module tests would not cover a new implementation.
* Latent behaviour that callers may rely on (I.6) has to be decided case by case.
* `perl-base` is Essential anyway; dpkg-dev also needs make, binutils, patch,
  tar, xz — removing Perl from dpkg-dev saves little install footprint.

### H.3 Reasons for native rewrites (evidence)

* Measured 37× startup ratio vs C binaries, multiplied by high spawn counts from
  the mk fragments and debhelper (≈1.2 s of a 2.8 s minimal build).
* Some hot paths are pure computation in Perl: symbols-file parsing (41 ms for
  libc6), status-file parsing (dpkg-checkbuilddeps 51-55 ms, dpkg-genbuildinfo
  100 ms), with duplicated parsers (`dpkg-checkbuilddeps.pl:193` vs
  `dpkg-genbuildinfo.pl:93`).
* Text-format coupling to external tools (`objdump -w -f -p -T -R` output;
  `dpkg-query --search` output, including diversion lines,
  `dpkg-shlibdeps.pl:1047-1060`) could be replaced by direct ELF/database access.
* Several small tools (architecture, vendor, buildapi, distaddfile, buildtree,
  ar) are thin and table-driven: S-sized.

---

## I. What ties the programs to Perl (user-visible)

1. **Perl regex in CLI options / data formats**
   * `dpkg-source -i<regex>`, `--diff-ignore=<regex>`, `--extend-diff-ignore=<regex>`
     (`dpkg-source.pl:187-196`), applied as `m/$regex/o` (`Patch.pm:191`); default
     uses `(?:…)` groups (`scripts/Dpkg/Source/Package.pm:62-74`). These live in
     `debian/source/options` of source packages (read for build modes,
     `dpkg-source.pl:137-154`; not for `-x`). `--tar-ignore` is a tar glob, not Perl.
   * Symbols files `(regex)` tag: "They match by the perl regular expression
     specified in the symbol name field" (`man/deb-src-symbols.pod:350-353`, examples
     with `\d` at `:361-377`; compiled `qr/$regex/` at `Shlibs/Symbol.pm:193-197`).
   * `dpkg-scanpackages -a/-t` interpolated unescaped into a regex
     (`dpkg-scanpackages.pl:288-297`): verified `-a 'amd6.'` and `-a 'i386|amd64'`
     both match the 200 amd64 debs.
   * `dpkg-gensymbols -e<pattern>` uses Perl `glob()` (`dpkg-gensymbols.pl:154-163`).
   * Arch tables' regex column (`data/cputable`, `data/ostable`) only uses
     ERE-compatible syntax (checked all values).
2. **Vendor hooks**: `Dpkg::Vendor::<Name>` modules found by name from the
   origins `Vendor` field (`scripts/Dpkg/Vendor.pm:172-201`); hook ids include
   before-source-build, package-/archive-keyrings, builtin-build-depends/-conflicts,
   register-custom-fields, post-process-changelog-entry, extend-patch-header,
   update-buildflags, builtin-system-build-paths, build-tainted-by,
   sanitize-environment, backport-version-regex (returns a `qr//`,
   `Vendor/Debian.pm:106-107`), has-fuzzy-native-source
   (`scripts/Dpkg/Vendor/Default.pm:177-214`). In-tree vendors: Debian, Default,
   Devuan, PureOS, Ubuntu; derivatives can ship their own module.
3. **Changelog format plug-ins**: `-F <fmt>` or `changelog-format:` trailer →
   `require Dpkg::Changelog::<Fmt>` (`scripts/Dpkg/Changelog/Parse.pm:47-67,164-172`);
   declared stable (`doc/README.api:30-35`); the old program-based `-L` mechanism
   was dropped in favour of Perl modules (`doc/README.feature-removal-schedule`).
4. **Build drivers**: `Build-Driver:` field → `Dpkg::BuildDriver::<Name>`
   (`scripts/Dpkg/BuildDriver.pm:91-118`; spec draft `doc/spec/build-driver.txt`).
5. **dpkg-buildpackage hook mini-language**: `--hook-<name>=<cmd>` for 12 hooks
   (`:412-425,499-505`); `%%`, `%a` (enabled), `%p` source, `%v` version, `%s`
   version w/o epoch, `%u` upstream version; unknown `%x` warns and is kept
   (`:1095-1115`); env `DPKG_BUILDPACKAGE_HOOK_NAME` plus per-hook
   `…_SOURCE_OPTIONS/_BUILD_TARGET/_BINARY_TARGET/_BUILDINFO_OPTIONS/_CHANGES_OPTIONS/_CHECK_OPTIONS`.
   Executed with Perl single-string `system($cmd)` (`:1079-1084,1122`): via
   `/bin/sh -c` only when the string contains shell metacharacters, otherwise
   word-split and exec'd directly (Perl semantics, unverified edge-case differences).
   `-r/--root-command` and `-R/--rules-file` are split on spaces (`:492-494,621-623`).
6. **Bug-compatible quirks a rewrite must decide on** (all verified by running):
   * `dpkg-buildapi`: any arg containing `-c` is taken as `-c<file>`; any arg
     ending in `--help` prints help (unanchored/precedence regexes, `:64,70`).
   * `dpkg-name`: `^-v|--version$`-style alternations (`:256-274`), e.g. `-vfoo` prints the version.
   * `dpkg-buildtree clean is-rootless` → "two commands specified:  and clean"
     plus an uninitialized-value warning (`$1` instead of `$arg`, `:66`).
   * `dpkg-buildflags --export=sh` does not escape `"` (`s/"/\"/g` is a no-op,
     `:159`): `DEB_CFLAGS_APPEND='-DFOO="bar baz"'` produced
     `export CFLAGS="… -DFOO="bar baz""`.
   * `dpkg-mergechangelogs`: the backport regex is applied to `$a/$b` instead of
     `$av/$bv` (`:204-207`), so it has no effect; introduced by commit b5d87e21b
     which replaced `$av =~ s/~(bpo|deb)/+$1/`; no test covers it (no bpo data in
     `scripts/t/dpkg_mergechangelogs/`) (code reading + git history; not run).
   * `dpkg-scanpackages` sorts multiple versions with string `cmp`, not version
     comparison (`:321`).
7. **Output formats parsed in the wild**: `dpkg-architecture` `VAR=value`/`-s`
   shell/`make` formats (parsed by dpkg-buildpackage `:785-791` and debhelper
   `Dh_Lib.pm:1857-1863`); `dpkg-buildflags --status` lines designed for log
   extraction (`:211-213`); `--export` formats; `dpkg-parsechangelog` deb822 and
   `-S` continuation encoding (`:176-192`); `.dsc/.changes/.buildinfo/DEBIAN/control`
   field order and wrapping (FieldsCore tables; `Binary` wrap at 980 chars,
   `dpkg-genchanges.pl:573-575`, `dpkg-source.pl:466-468`); dpkg-gensymbols
   `diff -u` report; warning texts (`dpkg-shlibdeps` "should not be linked
   against", etc.) that lintian-like log checkers may grep (unverified).
8. **Config/state formats**: `buildpackage.conf`, `buildflags.conf`,
   `debian/source/{options,local-options}` (Dpkg::Conf: long options without
   leading `--`); `debian/files` + fcntl lock on `debian/control`
   (`dpkg-distaddfile.pl:83-97`, gencontrol/genbuildinfo); dselect ftp method
   state stored as Data::Dumper and `eval`ed (`Dselect/Method.pm:94-124`).
9. **Environment variables**: documented per man page `ENVIRONMENT` sections —
   dpkg-buildpackage 25, dpkg-buildflags 15 (plus the generated
   `DEB_<FLAG>_{SET,STRIP,APPEND,PREPEND}` and `_MAINT_` families), dpkg-source 10,
   dpkg-genbuildinfo 6, dpkg-checkbuilddeps 4, others 2-4. Names found in code
   include `CC`, `DEB_{BUILD,HOST}_ARCH…`, `DEB_BUILD_PATH`, `DEB_BUILD_PROFILES`,
   `DEB_CHECK_COMMAND`, `DEB_SIGN_KEYFILE/KEYID`, `DEB_VENDOR`, `DEB_RULES_REQUIRES_ROOT`,
   `DEB_GAIN_ROOT_CMD`, `DEBEMAIL`, `DPKG_{ADMINDIR,BUILD_API,COLORS,DATADIR,NLS,
   ORIGINS_DIR,PROGMAKE,PROGPATCH,PROGTAR,GENSYMBOLS_CHECK_LEVEL}`, `SOURCE_DATE_EPOCH`,
   `MAKEFLAGS`, `LD_LIBRARY_PATH` (deprecated), `GNUPGHOME`, `EDITOR/VISUAL`,
   `XDG_CONFIG_HOME`, git `GIT_*`. `Dpkg::BuildEnv` records which vars were read
   and `dpkg-buildflags --query/--status` print them (`dpkg-buildflags.pl:179-181,218-224`).
10. **Option-parsing conventions**: three styles coexist — Getopt::Long
    (checkbuilddeps, mergechangelogs, scanpackages, scansources), `normalize_options`
    splitting `-Xvalue`/`--opt=value` (`scripts/Dpkg/Getopt.pm`, used by
    architecture and parsechangelog), and hand-written regex loops with attached
    values such as `-Vname=value`, `-DField=value`, `-pPkg`, `-O[file]` (the rest).
    A replacement CLI parser must reproduce attached-value forms and `-O` vs
    `-Ofile` ambiguity.

---

## Could not verify / open items

* Whether release tarballs ship prebuilt man pages through some other path
  (only checked `DISTFILES`/`dist-hook` in `man/Makefile.{am,in}`: they do not).
* Callers outside this host: sbuild, pbuilder, debuild, gbp, dak, lintian log
  parsing, blhc parsing `dpkg-buildflags: status:` — not installed/not inspected.
* Number of archive packages that include `/usr/share/dpkg/*.mk`, use symbols
  `(regex)` patterns, or set `--extend-diff-ignore` (no archive access).
* Exploitability/impact of the dpkg-name traversal beyond the observed move.
* Absence of signature verification in the dselect ftp method (grep-based).
* The proposed mk-fragment fixes (export `DPKG_BUILD_API`, batching) were not
  implemented or measured.
* Rust-side tooling claims (gettext extraction for Rust, runtime positional
  printf crates, PCRE/fancy-regex coverage of Perl regex) are not verified here.
