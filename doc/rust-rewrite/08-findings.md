# 08 — Defects, divergences and quirks found during the study

Reading 110,000 lines closely turns up things. This page lists what the analysis found in
upstream dpkg 1.23.x, for two reasons:

- some items deserve a report or a patch upstream;
- a reimplementation has to decide, item by item, whether to reproduce the current
  behaviour or to fix it, and a differential test harness needs to know which differences
  are intentional.

"Verified" means the behaviour was reproduced by running code. "Read" means it follows
from the source but was not executed. Details and reproduction notes are in the
[reference notes](reference/).

> **Before publishing this page:** the items in section 1 may have security relevance.
> They should be reported to the dpkg maintainers (see `README` for the contact) and given
> time to be assessed before this document is pushed to a public repository.

## 1. Items with possible security relevance

| # | Where | Finding | Status | Notes |
|---|---|---|---|---|
| S1 | `scripts/Dpkg/BuildDriver.pm:91-103` | The value of the `Build-Driver` field of `debian/control` is turned into a module name and passed to a string `eval` without restricting its characters. The other plug-in loaders (vendor, changelog format, source format) do restrict them. | Verified: code embedded in the field ran when a build driver object was created. | Callers are `dpkg-buildpackage` and `dpkg-buildtree`. Building a package runs `debian/rules` anyway, so the practical impact depends on whether any caller inspects an untrusted tree without building it. Not assessed. |
| S2 | `scripts/dpkg-name.pl:141-163` | The `Package`, `Version` and `Architecture` fields of a `.deb` are interpolated into the destination file name with only spaces removed. | Verified: a `.deb` whose `Package` field contains path separators was moved outside the current directory. | `Dpkg::Package::pkg_name_is_invalid()` exists and is used by other tools. |
| S3 | `src/main/archives.c:759-764`, `lib/dpkg/fsys-hash.c` | `dpkg` rejects newlines in tar member names but not `..` components. | Verified: with `--root=DIR`, a member named `./../x` was written outside `DIR` and recorded in the file list. `dpkg-deb -x` refuses the same archive because GNU tar does. | Upstream's documented position is that installing untrusted packages is never safe. The inconsistency between `dpkg` and `dpkg-deb` is still worth raising, especially for `--root` and `--instdir` users. |
| S4 | `src/split/join.c:103-158` | `dpkg-split --join` checks that the parts agree with each other but never compares the MD5 of the reassembled file with the checksum recorded in the part headers. | Verified: a part with a flipped data byte joined with exit status 0. | |
| S5 | `lib/dpkg/compress.c` (xz `:711`, zstd `:1123`, bzip2) | The in-process xz, zstd and bzip2 decoders stop silently after the first stream or frame. gzip and all the command-line tools decode concatenated streams. | Verified with two concatenated streams. | The same bytes are interpreted differently depending on how dpkg was built and on which tool reads them. A reimplementation using a library's default multi-stream decoder would change behaviour. |
| S6 | `lib/dpkg/meminfo.c:83-108` | Reading a `meminfo` file of 4,096 bytes or more writes one byte past a stack buffer. | Verified with AddressSanitizer on a 5,000-byte input. | The real `/proc/meminfo` on the test machine is 1,615 bytes. |
| S7 | `scripts/dpkg-buildflags.pl:159` | `--export=sh` intends to escape double quotes, but the substitution replaces `"` with `"`. | Verified: a flag value containing double quotes produced a broken `export` line. | The output is meant to be passed to `eval`. Flag values come from the environment and from the maintainer. |
| S8 | `dselect/methods/ftp/` | The FTP access method downloads `Packages` files and packages and hands them to `dpkg` with only MD5 and size checks. | Read (no signature verification code was found by searching). | Legacy component. |

## 2. Correctness bugs

### 2.1 Perl

| # | Where | Finding | Status |
|---|---|---|---|
| P1 | `scripts/Dpkg/Deps.pm:127` | `$v_p >= $v_p` (a typo for `$v_q`) makes `a (>= 5)` "imply the negation of" `a (<< 10)`: `implies()` returns 0 instead of undef. Present since 2009. | Verified |
| P2 | `scripts/Dpkg/Deps/Simple.pm:287-288` | `my $x = defined $p and $p->[0] =~ /^!/;` assigns only `defined $p`, because `and` binds more loosely than `=`. Every architecture list is treated as negated, which reverses implication between positive lists. Present since 2018. | Verified |
| P3 | `scripts/Dpkg/Deps.pm:422-423` | Same precedence slip: the `is_empty()` test in `deps_compare` is discarded. | Read (confirmed with `B::Deparse`) |
| P4 | `scripts/Dpkg/BuildProfiles.pm:100-132` | An invalid profile name is replaced by the string `1` (the return value of `warning()`) instead of being dropped. | Verified |
| P5 | `scripts/dpkg-mergechangelogs.pl:204-207` | The backport-version regex is applied to the wrong variables, so backport handling has no effect. | Read, with git history |
| P6 | `scripts/dpkg-buildapi.pl:64,70`, `scripts/dpkg-name.pl:256-274` | Unanchored option regexes: any argument containing `-c` is taken as `-c<file>`; `-vfoo` prints the version. | Verified |
| P7 | `scripts/dpkg-buildtree.pl:66` | The "two commands specified" error prints an empty command name and an uninitialised-value warning. | Verified |
| P8 | `scripts/dpkg-scanpackages.pl:321` | Multiple versions of a package are sorted as strings, not as versions. | Read |
| P9 | `scripts/Dpkg/Control.pm:277,281` | The same constant is tested twice; the name for copyright licence stanzas is unreachable. | Read |
| P10 | `scripts/Dpkg/File.pm:55` | An error message interpolates the file handle instead of the file name. | Read |
| P11 | `scripts/Dpkg/Source/Patch.pm:516-521` | A loop left over from a removed symlink check does nothing. | Read |
| P12 | `dselect/po/POTFILES.in` | Lists the Perl access methods, but their strings use `g_()`, which is not among the configured keywords, so they are never extracted. | Read |

### 2.2 C

| # | Where | Finding | Status |
|---|---|---|---|
| C1 | `src/split/queue.c:330-333` | `dpkg-split --listq` prints its header once per package. | Verified |
| C2 | `src/main/depcon.c:548` | `*canfixbytrigaw = …` is not guarded against NULL, although the same assignment at `:446` is, and some callers pass NULL. | Read; reachability not established |
| C3 | `src/main/archives.c`, `src/main/configure.c:522` | A conffile's `.dpkg-new` is never fsynced, and its rename during configuration is not followed by one. No parent directory of an installed file is fsynced. | Verified with `strace` |
| C4 | `lib/dpkg/trigdeferred.c:262-281` | `triggers/Unincorp` is renamed into place without an fsync of the new file, unlike every other atomic write in the library. | Read |
| C5 | `lib/dpkg/ehandle.c:203` | Calling `ohshit()` with no error context dereferences NULL; the intended "error outside error context" report is unreachable. | Verified |
| C6 | `lib/dpkg/string.c:172` | `str_quote_meta()` allocates no room for the terminating NUL. Only tests call it. | Verified with AddressSanitizer |
| C7 | `lib/dpkg/subproc.h:46,48` | `SUBPROC_RETERROR` and `SUBPROC_RETSIGNO` have the same bit value. | Read |
| C8 | `lib/dpkg/pkg-show.c:383-390` | The package sort comparator returns −1 for both orderings in one case. | Read; practical effect not established |
| C9 | `lib/dpkg/log.c:105` | The status-fd list is allocated in the database arena and would dangle after a database reset. Unreachable while `--command-fd` stays disabled. | Read |
| C10 | `src/main/main.c:870-991` | `commandfd()` is compiled but its option entry is commented out. | Verified |
| C11 | `src/main/main.c:622-625` | A failing pre-invoke hook reports the raw wait status ("exit code 256"). | Verified |
| C12 | `lib/dpkg/color.c:36-51` | Colour mode `auto` tests whether stdout is a terminal, even for messages written to stderr. | Read |

### 2.3 Build and packaging

| # | Where | Finding | Status |
|---|---|---|---|
| B1 | `lib/dpkg/libdpkg.map` | 77 exported symbols are missing from the version script, three of them used by `src/`. Matters only for the shared-library build that author testing enables. | Verified with `nm` |
| B2 | `m4/dpkg-compiler.m4:272-274` | `--enable-compiler-analyzer` prepends variables that are never set, so `-fanalyzer` appears not to be applied. | Read |
| B3 | `data/tupletable` | `freebsd-riscv` refers to a CPU `riscv` that `cputable` does not define. | Read |
| B4 | `debian/control` | The comment on `libfile-fcntllock-perl` says it is used by `Dpkg::File`; the user is `Dpkg::Lock`. | Read |
| B5 | `.github/workflows/build.yml` (this fork) | The failure step prints `test-suite.log`, a file this project does not produce. The functional suite is not run. | Read |
| B6 | `scripts/mk/buildapi.mk:12` | `DPKG_BUILD_API` is a recursively expanded variable holding a `$(shell …)` call, so `dpkg-buildapi` runs every time it is expanded. | Verified by counting process spawns |

## 3. Where the C and Perl implementations disagree

| Topic | C | Perl | Status |
|---|---|---|---|
| Digit runs of 2^64 or more in versions | compared digit by digit | converted to floating point; adjacent values compare equal | Verified |
| Epoch larger than `INT_MAX` | rejected | accepted | Verified |
| Epoch syntax | parsed by `strtol`: `+1:1.0` and `-0:1.0` are valid | must be digits | Verified (C) |
| Bytes ≥ 0x80 in versions | order depends on whether `char` is signed on the platform | by code point | Verified on x86-64 only |
| `pkg (1.0)` without an operator | accepted as `=` with a warning | the whole field fails to parse | Verified (Perl), read (C) |
| Whitespace-only line inside a multi-line field | part of the value in lax mode | ends the stanza | Verified |
| `#` comment lines in control data | not supported | skipped anywhere | Read |
| OpenPGP clear-sign wrapper | not supported | recognised and skipped | Read |
| Package name characters | upper case and `_` accepted | `Dpkg::Package`: lower case only; `Dpkg::Deps::Simple`: upper case allowed, `_` not | Read |
| zstd | supported | not supported by `Dpkg::Compression` | Read |

Only version comparison has an automated cross-check, and its 43 test pairs include none of
these cases.

## 4. Behaviour that looks like a bug but is a contract

These are deliberate or long-standing and must be reproduced by a replacement:

- `Priority: weird` is accepted as a custom priority, while `Priority: optionalx` is an
  error (prefix match followed by a trailing-junk check).
- Package names are stored in lower case; upper case is valid input.
- The status file is written in a fixed field order regardless of input order.
- Repairs for databases written by very old dpkg versions are applied on every load.
- `dpkg` never replaces a directory with a symlink or the reverse, and follows existing
  symlinked directories when unpacking.
- The error message is printed when the error context unwinds, so "error processing
  archive" appears before the output of the abort scripts.
- If an undo step fails, the remaining abort scripts for that package are skipped.
- A `status: … conffile-prompt …` line is emitted even when a force option answers the
  question.
- Options must precede operands; there is no GNU-style permutation.
- A dpkg configuration file can set an *action*, because actions and options share one
  table.
- `--compare-versions` warns about an invalid version and still compares.
- Tar archives using PAX headers are rejected.
- User and group names in `statoverride` are resolved from `/etc/passwd` and `/etc/group`
  directly, not through NSS.
- The journal file `updates/tmp.i` is padded to 4 KiB before use.
- `dpkg-divert` and `dpkg-statoverride` do not take the database lock.

Longer lists, with source locations, are in section L.1 of
[reference/01-libdpkg.md](reference/01-libdpkg.md) and section J.2 of
[reference/02-src-programs.md](reference/02-src-programs.md).
