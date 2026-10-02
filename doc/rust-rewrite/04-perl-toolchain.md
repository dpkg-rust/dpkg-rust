# 04 — The Perl side: dpkg-dev and libdpkg-perl

Half of the repository is Perl. It implements the *build* side of Debian packaging: the
tools a package build runs (`dpkg-buildpackage`, `dpkg-source`, `dpkg-gencontrol`,
`dpkg-shlibdeps`, …), the make fragments packagers include, and a module library that
other Debian tools use as an API. None of it is needed to install packages.

Line-level detail and citations: [reference/03-perl-library.md](reference/03-perl-library.md)
(modules) and [reference/04-perl-programs.md](reference/04-perl-programs.md) (programs,
timings, i18n, man pages).

## 1. What there is

| Part | Location | Size | Shipped as |
|---|---|---|---|
| Module library | `scripts/Dpkg.pm`, `scripts/Dpkg/**` | 92 modules, 28,639 lines (15,233 code, 7,433 POD) | `libdpkg-perl` (Architecture: all); also a CPAN distribution named `Dpkg` |
| Programs | `scripts/*.pl` | 20 files, 8,353 lines (19 installed + test-only `dpkg-ar`) | `dpkg-dev` (Architecture: all) |
| Make fragments | `scripts/mk/*.mk` | 8 files | `dpkg-dev`, under `/usr/share/dpkg/` |
| Tests | `scripts/t/` | 51 test files, 12,745 assertions, 210 fixture files | not shipped |
| dselect access methods | `dselect/methods/` | 2,198 lines Perl plus shell | `dselect` |
| Build and test helpers | `build-aux/`, `t/`, `utils/t/`, `lib/dpkg/t/*.t`, `dselect/mkcurkeys.pl` | — | not shipped |

All modules start with `use v5.36`. The library is pure Perl with no XS code. Its only
non-core dependencies are optional: `Locale::gettext` (translations) and `File::FcntlLock`.

## 2. The module library

### 2.1 Public and private modules

`doc/README.api` makes a promise: modules with `$VERSION` 1.00 or higher are a **stable
public API**, defined by their POD documentation, with the major version bumped on breaking
changes. Custom changelog parsers written as `Dpkg::Changelog` subclasses are a second
stable interface.

| | Modules | Code lines |
|---|---|---|
| Public (`$VERSION >= 1.00`) | 43 | 7,228 |
| Private (`$VERSION < 1.00`) | 49 | 8,005 |

The public surface is about 387 functions and methods. It is used in practice: on the
analysis machine `apt-cache rdepends libdpkg-perl` lists 35 packages, including debhelper,
lintian, devscripts, sbuild, dgit, autopkgtest, dh-python and mmdebstrap. The git history
shows `Breaks` being added because external tools depended on undocumented behaviour and
even on private modules (dgit, pkg-kde-tools, dh-exec).

### 2.2 Module groups

| Group | Modules | Public | Code lines | What it does |
|---|---|---|---|---|
| Infrastructure | `Dpkg`, `ErrorHandling`, `Gettext`, `IPC`, `Exit`, `Path`, `File`, `Lock`, `Getopt`, `Conf`, `Color`, `SysInfo`, `Package`, `Interface::Storable` | 7 of 14 | 1,178 | Error reporting, translations, subprocesses, option files |
| Version | `Version` | yes | 252 | Version objects and comparison |
| Architecture | `Arch` | yes | 438 | Debian architecture ↔ GNU triplet ↔ multiarch mapping from the tables in `data/` |
| Control files | `Control*`, `Index`, `Email::*` | 10 of 14 | 2,157 | deb822 parsing and writing; the registry of 118 known fields |
| Dependencies | `Deps`, `Deps::*` | all 7 | 886 | Parse, evaluate, simplify dependency fields |
| Changelog | `Changelog*` | all 5 | 996 | `debian/changelog` parser, ranges, output formats, plug-ins |
| Source packages | `Source::Package*`, `Source::Format` | 2 of 9 | 2,400 | Source formats 1.0, 2.0, 3.0 (quilt, native, git, bzr, custom) |
| Patches and archives | `Source::Patch`, `Quilt`, `Archive`, `Functions`, `BinaryFiles` | none | 1,341 | Patch generation, validation and application; tarball handling |
| Shared libraries | `Shlibs*` | none | 1,594 | ELF inspection through `objdump`; symbols files |
| Build | `BuildFlags`, `BuildOptions`, `BuildProfiles`, `BuildEnv`, `BuildTypes`, `BuildInfo`, `BuildAPI`, `BuildDriver*`, `BuildTree`, `Substvars`, `Dist::Files` | 5 of 12 | 1,422 | Compiler flags, build options and profiles, substitution variables |
| Vendor | `Vendor`, `Vendor::*` | 1 of 6 | 815 | Vendor detection and the hook mechanism; Debian and Ubuntu build-flag policy |
| OpenPGP | `OpenPGP*` | none | 862 | Signing and verification through sop, Sequoia or GnuPG command-line tools |
| Compression, checksums, ar | `Compression*`, `Checksums`, `Archive::Ar` | 4 of 5 | 892 | Compressor commands, MD5/SHA-1/SHA-256, `ar` archives |

The static dependency graph has no cycles and is 13 levels deep. `Dpkg::ErrorHandling` and
`Dpkg::Gettext` are used by almost every module.

### 2.3 How the important modules work

**Control files.** A `Dpkg::Control` object looks like a hash (`$ctrl->{Version}`), but it
is a blessed scalar with an overloaded `%{}` operator returning a *tied* hash that is
case-insensitive and remembers insertion order. Parsing is line-based: `#` comment lines
are skipped, an OpenPGP clear-sign wrapper is recognised and skipped (not verified),
duplicates are errors. Output order comes from per-file-type lists in
`Dpkg::Control::FieldsCore`, a 1,180-line module that is almost entirely data.

**Dependencies.** One regular expression parses
`name[:arch] (relation version) [architectures] <profiles>`; alternatives and conjunctions
become `OR`/`AND`/`Union` objects. `implies()` returns true, false or unknown from a
hand-written truth table over version relations, and `simplify_deps()` uses it to drop
redundant dependencies.

**Versions.** Same ordering rules as the C code, implemented separately.

**Architectures.** `data/cputable`, `ostable`, `tupletable` and `abitable` drive every
conversion. The tables contain regular expressions. The build architecture comes from
running `dpkg --print-architecture`; the host GNU type from `$CC -dumpmachine`.

**Changelogs.** A four-state line parser. Output can be regenerated byte-for-byte from the
parsed form. The parser class is chosen from a `changelog-format:` line in the file's tail
and loaded as `Dpkg::Changelog::<Format>`.

**Source packages.** `Dpkg::Source::Package` reads a `.dsc`, picks the class for its
`Format` field, and re-blesses itself into it. The format classes orchestrate GNU `tar`,
GNU `patch`, `diff`, compressors, `git` or `bzr`, and OpenPGP tools. `Dpkg::Source::Patch`
validates patches before applying them: it rejects C-quoted file names, paths containing
`/../`, patches to anything that is not a plain file, and inconsistent hunk counts. It
relies on GNU patch for the remaining directory-traversal protection, which is why
`configure` refuses to build without GNU patch.

**Shared libraries.** Reads 64 bytes of each ELF header natively, then parses the text
output of `objdump -w -f -p -T -R` and pipes C++ names through a long-running `c++filt`.
Symbols files support tags and three pattern kinds: `c++`, `symver` and `regex`.

**Build flags.** `Dpkg::BuildFlags` is a store with layered sources (vendor defaults, system
and user configuration files, `DEB_*_SET/STRIP/APPEND/PREPEND` environment variables, then
the maintainer variants). The actual policy — which hardening, LTO, time64 and
reproducibility flags apply on which architecture — is Perl code in
`Dpkg::Vendor::Debian` (lines 115-681), with overrides in `Dpkg::Vendor::Ubuntu`.

**Vendors.** `/etc/dpkg/origins/default` names the vendor. The class `Dpkg::Vendor::<Name>`
is loaded by computed module name, falling back to the parent vendor. Fifteen hooks are
dispatched through `run_vendor_hook()` from both modules and programs: keyrings, built-in
build dependencies, custom fields, changelog post-processing, patch headers, build flags,
environment sanitising, and others.

### 2.4 Perl-specific behaviour in the public API

The stable API is not just function signatures. Consumers depend on:

- **Tied objects**: the case-insensitive ordered hash behind every control object, and
  `Dpkg::Compression::FileHandle`, a file handle that transparently spawns a compressor.
- **Operator overloading** in nine modules: `<=>` and `cmp` on versions, stringification of
  control objects and changelog entries, array dereference of indexes and changelogs.
- **Callbacks**: key functions and filters for `Dpkg::Index`, `deps_iterate()`, substvar
  filters, exit handlers.
- **Context-sensitive returns** (`wantarray`) in a dozen modules.
- **Documented object fields**: `Dpkg::Deps::Simple` documents its hash keys, and the
  programs read them directly.
- **Exceptions**: errors are `die` with a formatted, translated message.
- **Plug-ins loaded by module name**: vendors, changelog formats, source formats, OpenPGP
  back-ends, build drivers.

And two *file formats* are defined in terms of Perl regular expressions: the `(regex)` tag
of symbols files ("the perl regular expression specified in the symbol name field",
`man/deb-src-symbols.pod`) and `dpkg-source --extend-diff-ignore`, which packages set in
`debian/source/options`.

### 2.5 The same logic, twice

Several things are implemented once in C and once in Perl:

| Logic | C | Perl | Agree? |
|---|---|---|---|
| Version parsing and comparison | `lib/dpkg/version.c`, `parsehelp.c` | `Dpkg::Version` | Not on edge cases: numbers of 2^64 or more, epochs above `INT_MAX`, whitespace |
| deb822 parsing | `lib/dpkg/parse.c` | `Dpkg::Control::HashCore` | Different parsers: comments, OpenPGP armor, whitespace-only lines, duplicates |
| Field registry and output order | `fieldinfos[]` | `Dpkg::Control::FieldsCore` | Kept in sync by hand |
| Dependency syntax | `lib/dpkg/fields.c` | `Dpkg::Deps` | C accepts `pkg (1.0)`; Perl rejects the whole field. Perl also handles architecture and profile restrictions |
| Package-name validation | `pkg_name_is_invalid()` | `Dpkg::Package`, `Dpkg::Deps::Simple` | Three different rules |
| Compression | `lib/dpkg/compress.c` (in-process, includes zstd) | `Dpkg::Compression` (external commands, no zstd) | Different capabilities |
| `ar` archives | `lib/dpkg/ar.c` | `Dpkg::Archive::Ar` | Perl version is test-only |
| Reporting, colours, i18n switches, option files, locking | libdpkg | infrastructure modules | Same conventions, separate code |

The two sides also depend on each other. `configure` runs the in-tree
`scripts/dpkg-architecture.pl` to compute the architecture compiled into the C programs,
and `Dpkg::Arch` runs the C `dpkg --print-architecture` at run time. The only automated
cross-check is `scripts/t/Dpkg_Version.t`, which runs 43 version pairs through both
implementations.

## 3. The programs

| Program | Lines | Role | Where the logic lives |
|---|---|---|---|
| `dpkg-buildpackage` | 1,214 | Orchestrates a package build: clean, source, build, binary, buildinfo, changes, sign; 12 hook points | in the script |
| `dpkg-source` | 808 | Build and extract source packages | `Dpkg::Source::*` |
| `dpkg-shlibdeps` | 1,070 | Compute shared-library dependencies for binaries | in the script, plus `Dpkg::Shlibs::*` |
| `dpkg-gensymbols` | 402 | Generate and compare symbols files | `Dpkg::Shlibs::*` |
| `dpkg-gencontrol` | 519 | Write `DEBIAN/control` for a binary package | in the script, plus `Substvars`, `Deps`, `Control` |
| `dpkg-genchanges` | 628 | Write the `.changes` upload file | in the script |
| `dpkg-genbuildinfo` | 638 | Write the `.buildinfo` file | in the script |
| `dpkg-checkbuilddeps` | 270 | Check build dependencies against the status file | `Dpkg::Deps` plus its own status parser |
| `dpkg-architecture` | 498 | Print and compare architecture variables | `Dpkg::Arch` |
| `dpkg-buildflags` | 250 | Print compiler and linker flags | `Dpkg::BuildFlags`, `Dpkg::Vendor::*` |
| `dpkg-parsechangelog` | 196 | Print changelog entries | `Dpkg::Changelog::*` |
| `dpkg-vendor` | 123 | Query vendor information | `Dpkg::Vendor` |
| `dpkg-buildapi` | 83 | Print the build API level | `Dpkg::BuildAPI` |
| `dpkg-scanpackages`, `dpkg-scansources` | 351, 358 | Build `Packages` and `Sources` indexes | in the scripts |
| `dpkg-name` | 289 | Rename `.deb` files to their canonical name | in the script |
| `dpkg-mergechangelogs` | 328 | Three-way merge of changelogs (git merge driver) | in the script |
| `dpkg-distaddfile`, `dpkg-buildtree` | 100, 89 | Edit `debian/files`; clean a build tree | modules |
| `dpkg-ar` | 139 | Test helper for crafting `.deb` files; not installed | `Dpkg::Archive::Ar` |

### 3.1 What a build runs

`dpkg-buildpackage` runs, in order: `dpkg-architecture -f` (to export the architecture
variables), `dpkg-source --before-build`, `dpkg-checkbuilddeps`, `debian/rules clean`,
`dpkg-source -b`, `debian/rules build` and `binary` (through fakeroot when root is
required), `dpkg-genbuildinfo`, `dpkg-genchanges`, `dpkg-source --after-build`, an
optional check command, then signing. Inside `debian/rules`, debhelper calls
`dpkg-shlibdeps`, `dpkg-gensymbols` and `dpkg-gencontrol` per binary package and the C
`dpkg-deb --build` to create each `.deb`.

Perl tools call the C tools in four places: `dpkg --print-architecture` (everything that
needs the build architecture), `dpkg-query --search` and `--control-path`
(`dpkg-shlibdeps`), `dpkg-deb --info` (`dpkg-scanpackages`) and `dpkg-deb -f`
(`dpkg-name`). No C program calls Perl.

### 3.2 The make fragments

`/usr/share/dpkg/*.mk` are a public interface for `debian/rules`:

| Fragment | Provides | How |
|---|---|---|
| `architecture.mk` | 33 `DEB_{BUILD,HOST,TARGET}_*` variables | one `dpkg-architecture -q` per variable, cached per make process |
| `buildflags.mk` | 20 flag variables | one `dpkg-buildflags --get` per flag, cached per make process |
| `buildapi.mk` | `DPKG_BUILD_API` | `dpkg-buildapi`, **not cached** |
| `pkg-info.mk` | `DEB_SOURCE`, `DEB_VERSION*`, `SOURCE_DATE_EPOCH` | one `dpkg-parsechangelog -S` per field |
| `vendor.mk` | `DEB_VENDOR`, `dpkg_vendor_derives_from` | `dpkg-vendor` |
| `buildopts.mk`, `buildtools.mk` | parallel jobs, tool names | pure make |
| `default.mk` | includes the others | — |

## 4. Measured performance

All numbers are medians on one pinned core of the analysis machine; method and raw data are
in the reference notes.

| Command | Time |
|---|---|
| C `dpkg --version` | 0.45 ms |
| `perl -e 1` | 0.72 ms |
| `dpkg-architecture -qDEB_HOST_MULTIARCH` | 16.9 ms |
| `dpkg-vendor --query Vendor` | 17.4 ms |
| `dpkg-buildflags --get CFLAGS` | 24.9 ms |
| `dpkg-parsechangelog` | 27.2 ms |
| `dpkg-buildapi` | 32.5 ms |
| `dpkg-checkbuilddeps` | 55.5 ms |
| `dpkg-genbuildinfo` | about 100 ms |
| `dpkg-shlibdeps` on one small binary | 228 ms |

Observations:

- The cheapest Perl tool starts about 37 times slower than a C binary. Roughly 7–9 ms of
  every start is `Locale::gettext` pulling in POSIX and Encode; with `DPKG_NLS=0`,
  `dpkg-architecture -q` drops from 16.9 ms to 8.1 ms.
- Throughput is fine: about 40,000 version comparisons per second, and the 2.3 MB status
  file of the analysis machine parses in 0.16 s.
- **The make fragments multiply the start-up cost.** A minimal debhelper build took 1.61 s
  without `include /usr/share/dpkg/default.mk` and 2.83 s with it, because the include adds
  40 `dpkg-buildflags` and 8 `dpkg-buildapi` runs. Each `override_dh_*` target adds another
  make process and another 22 runs. Outside `dpkg-buildpackage`, a no-op target spawns
  `dpkg-architecture` 33 times (about 0.7 s).
- Three causes were confirmed by experiment: debhelper exports all 20 flag variables, which
  forces make to recompute each in every make process; `DPKG_BUILD_API` is never cached;
  and exported lazily-evaluated variables are expanded for every child.
- Of the 228 ms in `dpkg-shlibdeps`, about 69 ms is the C `dpkg-query --search` loading
  every file list and about 41 ms is Perl parsing libc6's 5,228-line symbols file.

## 5. Tests

`scripts/t/` has 46 module test files and 5 program test files (`dpkg_source.t`,
`dpkg_buildpackage.t`, `dpkg_buildtree.t`, `dpkg_mergechangelogs.t`, `mk.t`). All pass.
The biggest are generated from data: `Dpkg_Arch.t` (6,967 assertions),
`Dpkg_Control_Fields.t` (2,629) and `Dpkg_Version.t` (1,755).

For a reimplementation the tests fall in three classes:

- **Reusable as they are** (input file → expected output or accept/reject): version
  triples, changelog round trips on 8 files, the control-file dump and 8 malformed
  signature cases, 7 patch fixtures, checksum data, changelog-merge fixtures.
- **Data expressed through Perl calls** (extractable with some work): architecture tables,
  field lists, dependency strings, build flags, profiles, substvars, objdump dumps and
  their expected symbols files.
- **Tied to the Perl API**: compression handles, IPC, exit handlers, option parsing, and
  all the overload and tie behaviour.

Program-level coverage is thin, and seven module test files only check that the module
loads.

## 6. Translations and manual pages

| Text domain | Source | Languages | Messages |
|---|---|---|---|
| `dpkg` | C programs and library | 44 | 1,323 |
| `dpkg-dev` | Perl programs and modules | 11 | 916 |
| `dselect` | dselect | 31 | 263 |
| `dpkg-man` (po4a) | 62 manual pages | 13 | 3,744 |

gettext looks messages up by their exact text, so a reimplementation that emits the same
strings under the same domain keeps the existing translations. Messages use C-style
`%s`/`%d` directives, and translators use positional forms such as `%1$s` to reorder
arguments. The per-option `--help` text is translated as preformatted blocks, so a command
line framework that formats help itself would orphan those strings.

Manual pages are 63 POD files (19,300 lines) in `man/`, rendered with `pod2man` and
translated with po4a. Module documentation is the POD inside each `.pm` file.

## 7. Where Perl is needed

| When | What needs Perl |
|---|---|
| Installing and removing packages | **Nothing.** The Essential `dpkg` package contains only C binaries and shell scripts |
| Building packages | Everything in `dpkg-dev` and `libdpkg-perl`; also debhelper, which is itself Perl |
| Building dpkg | `configure` (hard requirement, perl ≥ 5.36; it runs `dpkg-architecture.pl` and `dpkg-parsechangelog.pl`), the substitution step that generates the shipped shell scripts, `pod2man`, po4a, `dselect/mkcurkeys.pl`, autotools themselves |
| Testing dpkg | The TAP runner for every suite, including the C unit tests; `scripts/t/`, `utils/t/`, `t/` |
| Using dselect | The access methods |

The possibilities for reducing or replacing this Perl are analysed in
[07-perl-replacement-analysis.md](07-perl-replacement-analysis.md).
