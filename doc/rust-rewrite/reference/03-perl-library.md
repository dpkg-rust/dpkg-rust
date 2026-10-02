> **Reference notes.** Detailed working notes behind the chapters in the parent directory,
> produced by automated code analysis of dpkg 1.23.x (commit `27f661e21`) on 2026-10-02.
> Every non-trivial claim cites `path:line`; items marked "(unverified)" were inferred, not run.
> `<repo>` is the source tree, `<build>` an out-of-tree build of it, `<scratch>` a temporary
> directory that no longer exists. See [../README.md](../README.md) for how these notes were checked.

# 03 — The Perl module library (`scripts/Dpkg.pm`, `scripts/Dpkg/**`, `scripts/Test/`)

Analyst scope: the `Dpkg::*` Perl modules shipped as **libdpkg-perl** (and as the
`Dpkg` CPAN distribution), plus `Test::Dpkg` and the `scripts/t/Dpkg_*.t` unit tests.
Repository: `<repo>` (git, branch `develop`, clean tree,
upstream dpkg 1.23.12 development). System perl 5.40.1.

Conventions used below:
* `path:line` citations are relative to `scripts/` unless they start with `lib/`, `data/`,
  `debian/`, `m4/`, `man/` or `build-aux/` (which are relative to the repo root).
* "LOC" numbers come from a small Perl script (`scratchpad/inventory.pl`) that classifies
  each line of every `.pm` as POD (between `=xxx` and `=cut`), blank/comment, or code.
* Everything marked **(verified)** was reproduced by running code; **(unverified)** marks
  inferences I did not test.
* How to run modules in-tree: `DPKG_DATADIR=<repo>/data DPKG_ORIGINS_DIR=<repo>/scripts/t/origins perl -I scripts -MDpkg::X -e ...`
  (the same variables the test harness sets: `scripts/Makefile.am:219-226`;
  `build-aux/test-runner` sets `LC_ALL=C`, `DPKG_COLORS=never`, and `PATH` to the
  build tree, and passes `-I $srcroot/scripts`).

## Headline numbers (all measured)

| Metric | Value | How measured |
|---|---|---|
| `.pm` files in scope | 93 (92 `Dpkg*` + `Test::Dpkg`) | `find scripts/Dpkg scripts/Test scripts/Dpkg.pm -name '*.pm'` |
| Installed modules | 92 | `nobase_dist_perllib_DATA`, `scripts/Makefile.am:8-101`; `Test/Dpkg.pm` is `EXTRA_DIST` only (`:103-106`) |
| Total lines (92 Dpkg modules) | 28,639 (7,433 POD, 15,233 code, rest blank/comment) | `inventory.pl` |
| Public modules (`$VERSION >= 1.00`) | 43 modules, 15,676 lines (7,228 code, 5,749 POD) | `inventory.pl` |
| Private modules (`$VERSION < 1.00`) | 49 modules, 12,963 lines (8,005 code, 1,684 POD) | `inventory.pl`; 49 files carry "This is a private module" |
| Subroutine definitions | 849 (79 `_`-prefixed) | grep over non-POD lines |
| Non-`_` subs in public modules | 387 | same |
| Documented `=item` entries in METHODS/FUNCTIONS/VARIABLES/CONSTANTS sections of public modules | 564 (upper bound; includes option sub-items) | awk over POD |
| Non-core CPAN dependencies | 2, both optional: `Locale::gettext` (Dpkg::Gettext), `File::FcntlLock`/`::Pure` (Dpkg::Lock) | `Module::CoreList::is_core` over every `use`/`require` |
| Minimum perl | `use v5.36` in all 93 files | grep |
| Translatable call sites | `g_` 582, `N_` 49, `P_` 2, `C_` 2 | grep over non-POD lines |
| msgids in `scripts/po/dpkg-dev.pot` referenced from modules | 483 of 916 (19 shared with programs); 11 `.po` translations | awk over `.pot` |
| Unit tests (`t/Dpkg_*.t`) | 46 files, 7,200 lines, **12,694 assertions, all pass** (15 skipped: 11 OpenPGP-backend, 2+2 changelog) | ran `prove` on a scratch copy of `scripts/` |
| Unit-test fixture files | 99 (in 14 `t/Dpkg_*/` dirs) | `find` (145 files minus 46 `.t`) |

## A. Module inventory

Notes on the table:
* "Internal deps" = `Dpkg::*` modules named in `use`/`require`/`use parent` lines of the
  module (D:: = Dpkg::). Lazily `require`d modules (inside subs) are included.
* Core-module deps are omitted from the table for width. Across the library the core
  modules used are: Exporter, Carp, List::Util, Scalar::Util, POSIX, Fcntl, Errno, Cwd,
  File::{Spec,Basename,Find,Path,Temp,Copy,Compare,stat}, IO::{File,Dir}, Digest (with
  Digest::MD5/Digest::SHA), MIME::Base64, Time::Piece, Time::HiRes, Term::ANSIColor,
  Tie::Hash, Storable, Config. All are core in perl 5.40 (checked with
  `Module::CoreList::is_core`). The only non-core modules are both optional and loaded via
  string `eval`: `Locale::gettext` (`Dpkg/Gettext.pm:124-131`, falls back to identity
  functions) and `File::FcntlLock`/`File::FcntlLock::Pure` (`Dpkg/Lock.pm:56-74`, falls back
  to `flock`). (`debian/control` lists `libfile-fcntllock-perl` as "Used by Dpkg::File" —
  the comment is stale; the user is `Dpkg::Lock`.)
* "External commands" = programs the module itself executes via `spawn`, `system`, `qx`
  or a pipe-open (found by grepping `exec =>`, `system`, `qx`, `open(..., '-|')`).
* `$VERSION` is declared in the `package NAME VERSION;` line of each file.

| Module | Lines total / POD / code | $VERSION | API | Purpose | Internal Dpkg deps (`use`/`require`/`parent`) | Non-core CPAN deps | External commands |
|---|---|---|---|---|---|---|---|
| Dpkg | 305 / 255 / 25 | 2.00 | **public** | Core path/program variables ($PROGNAME, $DATADIR, $PROGTAR...); hierarchy entry point | - | - | - |
| Dpkg::Arch | 746 / 159 / 438 | 1.04 | **public** | Debian arch <-> GNU triplet <-> tuple mapping from data/ tables; wildcard matching | Dpkg, D::BuildEnv, D::ErrorHandling, D::Gettext | - | `dpkg --print-architecture`; `$CC -dumpmachine` |
| Dpkg::Archive::Ar | 451 / 104 / 233 | 0.01 | private | Native ar(1) archive reader/writer | D::ErrorHandling, D::Gettext | - | - |
| Dpkg::BuildAPI | 145 / 40 / 63 | 0.01 | private | Resolve dpkg-build-api level (DPKG_BUILD_API or Build-Depends) | D::BuildEnv, D::Deps, D::ErrorHandling, D::Gettext, D::Version | - | - |
| Dpkg::BuildDriver | 194 / 106 / 46 | 0.01 | private | Loads build driver named by Build-Driver field | Dpkg, D::ErrorHandling, D::Gettext | - | - |
| Dpkg::BuildDriver::DebianRules | 310 / 63 / 175 | 0.01 | private | Runs debian/rules targets; Rules-Requires-Root logic | Dpkg, D::BuildTypes, D::ErrorHandling, D::Gettext, D::Path | - | debian/rules <target>, fakeroot or gain-root cmd |
| Dpkg::BuildEnv | 114 / 55 / 29 | 0.01 | private | Tracked %ENV get/set/has (records accessed/modified vars) | - | - | - |
| Dpkg::BuildFlags | 608 / 249 / 255 | 1.06 | **public** | Build-flag store (CFLAGS...), config files and DEB_*_SET/APPEND env ops | Dpkg, D::BuildEnv, D::ErrorHandling, D::Gettext, D::Vendor | - | - |
| Dpkg::BuildInfo | 207 / 31 / 133 | 1.00 | **public** | Allow-list of env vars recorded in .buildinfo | - | - | - |
| Dpkg::BuildOptions | 256 / 117 / 91 | 1.02 | **public** | Parse/manipulate DEB_BUILD_OPTIONS (+ feature areas) | D::BuildEnv, D::ErrorHandling, D::Gettext | - | - |
| Dpkg::BuildProfiles | 219 / 59 / 105 | 1.01 | **public** | DEB_BUILD_PROFILES + <restriction formula> parse/evaluate | D::BuildEnv, D::ErrorHandling, D::Gettext | - | - |
| Dpkg::BuildTree | 134 / 52 / 44 | 0.01 | private | Clean/needs_root for a source tree | D::BuildDriver, D::Control::Info, D::Source::Functions | - | - |
| Dpkg::BuildTypes | 286 / 117 / 111 | 0.02 | private | Bitmask of build types (source/any/all/binary/full) | D::ErrorHandling, D::Gettext | - | - |
| Dpkg::Changelog | 801 / 320 / 373 | 2.00 | **public** | Abstract changelog: entries, ranges, dpkg/rfc822 output formats | D::Control, D::Control::Changelog, D::Control::Fields, D::ErrorHandling, D::Gettext, D::Index, D::Interface::Storable, D::Vendor, D::Version | - | - |
| Dpkg::Changelog::Debian | 271 / 51 / 174 | 1.00 | **public** | debian/changelog line-oriented state-machine parser | D::Changelog, D::Changelog::Entry::Debian, D::File, D::Gettext | - | - |
| Dpkg::Changelog::Entry | 321 / 131 / 134 | 1.01 | **public** | Abstract changelog entry (header/changes/trailer parts) | D::Control::Changelog, D::ErrorHandling, D::Gettext | - | - |
| Dpkg::Changelog::Entry::Debian | 461 / 139 / 231 | 2.00 | **public** | Debian entry: header/trailer regexes, Closes:, dates | D::Changelog::Entry, D::Control::Changelog, D::Control::Fields, D::Gettext, D::Version | - | - |
| Dpkg::Changelog::Parse | 242 / 118 / 84 | 2.02 | **public** | changelog_parse(): format detection + plug-in loading | Dpkg, D::Control::Changelog, D::ErrorHandling, D::Gettext | - | - |
| Dpkg::Checksums | 472 / 221 / 182 | 1.04 | **public** | md5/sha1/sha256 compute, parse/export Checksums-* fields | D::ErrorHandling, D::Gettext | - | - |
| Dpkg::Color | 89 / 20 / 40 | 0.01 | private | DPKG_COLORS handling via Term::ANSIColor | - | - | - |
| Dpkg::Compression | 425 / 163 / 169 | 3.00 | **public** | Compressor table (gzip,bzip2,lzma,xz) and cmdlines | D::ErrorHandling, D::Gettext | - | (builds cmdlines for gzip/gunzip/bzip2/bunzip2/xz/unxz) |
| Dpkg::Compression::FileHandle | 509 / 192 / 234 | 1.02 | **public** | Tied IO::File that (de)compresses via subprocess | D::Compression, D::Compression::Process, D::ErrorHandling, D::Gettext | - | (via Process) |
| Dpkg::Compression::Process | 234 / 113 / 74 | 1.00 | **public** | Spawn compressor/decompressor processes | D::Compression, D::ErrorHandling, D::Gettext, D::IPC | - | gzip, gunzip, bzip2, bunzip2, xz, unxz |
| Dpkg::Conf | 297 / 144 / 98 | 1.04 | **public** | Parse dpkg-* .conf option files | D::ErrorHandling, D::Gettext, D::Interface::Storable | - | - |
| Dpkg::Control | 356 / 232 / 90 | 1.05 | **public** | Typed deb822 stanza (type selects name/output order/PGP) | D::Control::Fields, D::Control::Hash, D::Control::Types, D::ErrorHandling, D::Gettext | - | - |
| Dpkg::Control::Changelog | 64 / 30 / 11 | 1.00 | **public** | Control subclass for parsed changelog output | D::Control | - | - |
| Dpkg::Control::Fields | 70 / 21 / 24 | 1.00 | **public** | FieldsCore + vendor-registered fields (runs hook at load) | D::Control::FieldsCore, D::Vendor | - | - |
| Dpkg::Control::FieldsCore | 1472 / 184 / 1180 | 1.05 | **public** | Field registry: 118 fields, allowed types, order, dep types | D::Control::Types, D::Email::Address, D::Email::AddressList, D::ErrorHandling, D::Gettext | - | - |
| Dpkg::Control::Hash | 48 / 19 / 7 | 1.00 | **public** | HashCore + forces vendor field registration | D::Control::Fields, D::Control::HashCore, D::ErrorHandling, D::Gettext | - | - |
| Dpkg::Control::HashCore | 512 / 166 / 240 | 1.02 | **public** | deb822 stanza parser/serializer, case-insensitive ordered hash | D::Control::FieldsCore, D::Control::HashCore::Tie, D::ErrorHandling, D::Gettext, D::Interface::Storable | - | - |
| Dpkg::Control::HashCore::Tie | 156 / 38 / 72 | 0.01 | private | Tie::ExtraHash impl: lc keys, insertion order | D::Control::FieldsCore | - | - |
| Dpkg::Control::Info | 235 / 106 / 86 | 1.01 | **public** | debian/control: source stanza + binary stanzas | D::Control, D::ErrorHandling, D::Gettext, D::Interface::Storable | - | - |
| Dpkg::Control::Tests | 87 / 40 / 20 | 1.00 | **public** | debian/tests/control index | D::Control, D::Control::Tests::Entry, D::Index | - | - |
| Dpkg::Control::Tests::Entry | 98 / 43 / 26 | 1.00 | **public** | debian/tests/control stanza w/ validation | D::Control, D::ErrorHandling, D::Gettext | - | - |
| Dpkg::Control::Types | 119 / 24 / 56 | 0.01 | private | CTRL_* bitmask constants | - | - | - |
| Dpkg::Deps | 497 / 197 / 224 | 1.07 | **public** | deps_parse/compare/iterate, implication truth table | D::Arch, D::BuildProfiles, D::Deps::AND, D::Deps::KnownFacts, D::Deps::OR, D::Deps::Simple, D::Deps::Union, D::ErrorHandling, D::Gettext, D::Version | - | - |
| Dpkg::Deps::AND | 177 / 54 / 74 | 1.00 | **public** | AND list (", ") | D::Deps::Multiple | - | - |
| Dpkg::Deps::KnownFacts | 215 / 65 / 100 | 2.00 | **public** | Installed/provided package facts for evaluation | D::Version | - | - |
| Dpkg::Deps::Multiple | 247 / 104 / 80 | 1.02 | **public** | Base for AND/OR/Union | D::ErrorHandling, D::Interface::Storable | - | - |
| Dpkg::Deps::OR | 170 / 54 / 69 | 1.00 | **public** | OR list (" | ") | D::Deps::Multiple | - | - |
| Dpkg::Deps::Simple | 679 / 215 / 305 | 1.02 | **public** | Single dep: name:arch (rel ver) [arches] <profiles> | D::Arch, D::BuildProfiles, D::ErrorHandling, D::Gettext, D::Interface::Storable, D::Version | - | - |
| Dpkg::Deps::Union | 115 / 43 / 34 | 1.00 | **public** | Unordered union (Conflicts/Breaks-style) | D::Deps::Multiple | - | - |
| Dpkg::Dist::Files | 218 / 21 / 131 | 0.01 | private | debian/files list (filename section priority attrs) | D::ErrorHandling, D::Gettext, D::Interface::Storable | - | - |
| Dpkg::Email::Address | 205 / 74 / 78 | 0.01 | private | Parse "Name <email>" | D::ErrorHandling, D::Gettext | - | - |
| Dpkg::Email::AddressList | 197 / 73 / 73 | 0.01 | private | Parse comma-separated address lists | D::Email::Address, D::ErrorHandling, D::Gettext | - | - |
| Dpkg::ErrorHandling | 292 / 20 / 204 | 0.02 | private | warning/error/syserr/info reporting, colored prefixes | Dpkg, D::Color, D::Gettext | - | - |
| Dpkg::Exit | 132 / 49 / 48 | 2.00 | **public** | Exit-handler stack wired to INT/HUP/QUIT and END | - | - | - |
| Dpkg::File | 104 / 20 / 53 | 0.01 | private | file_slurp/file_dump/file_touch | D::ErrorHandling, D::Gettext | - | - |
| Dpkg::Getopt | 165 / 21 / 94 | 0.02 | private | Option normalization and --help formatting | Dpkg, D::Color, D::ErrorHandling, D::Gettext | - | - |
| Dpkg::Gettext | 229 / 122 / 72 | 2.01 | **public** | g_/P_/C_/N_ wrappers over optional Locale::gettext | - | Locale::gettext (optional, string-eval) | - |
| Dpkg::IPC | 457 / 180 / 218 | 1.03 | **public** | spawn()/wait_child() fork+exec with pipes/strings/timeouts | D::ErrorHandling, D::Gettext | - | - |
| Dpkg::Index | 494 / 237 / 194 | 3.00 | **public** | Keyed, ordered collection of Dpkg::Control objects | D::Control, D::ErrorHandling, D::Gettext, D::Interface::Storable | - | - |
| Dpkg::Interface::Storable | 162 / 74 / 60 | 1.01 | **public** | Mixin: load/save/stringify via parse/output | D::Compression::FileHandle, D::ErrorHandling, D::Gettext | - | - |
| Dpkg::Lock | 86 / 20 / 30 | 0.01 | private | fcntl/flock write lock | D::ErrorHandling, D::Gettext | File::FcntlLock / ::Pure (optional, string-eval) | - |
| Dpkg::OpenPGP | 178 / 20 / 103 | 0.01 | private | OpenPGP facade; auto-selects backend | D::ErrorHandling, D::Gettext, D::IPC, D::OpenPGP::ErrorCodes, D::Path | - | - |
| Dpkg::OpenPGP::Backend | 246 / 21 / 151 | 0.01 | private | Base backend; native ASCII-armor/CRC24 | D::ErrorHandling, D::File, D::Gettext, D::OpenPGP::ErrorCodes, D::Path | - | - |
| Dpkg::OpenPGP::Backend::GnuPG | 371 / 21 / 257 | 0.01 | private | gpg/gpgv backend | D::ErrorHandling, D::File, D::Gettext, D::IPC, D::OpenPGP::Backend, D::OpenPGP::ErrorCodes, D::Path | - | gpgv-sq/gpgv, gpg-sq/gpg (gpg-agent probed) |
| Dpkg::OpenPGP::Backend::SOP | 148 / 22 / 74 | 0.01 | private | Stateless OpenPGP CLI backend | D::ErrorHandling, D::IPC, D::OpenPGP::Backend, D::OpenPGP::ErrorCodes | - | sqopv/rsopv/sopv, sqop/rsop/gosop/hop/pgpainless-cli |
| Dpkg::OpenPGP::Backend::Sequoia | 228 / 21 / 136 | 0.01 | private | sq/sqv backend | D::ErrorHandling, D::Gettext, D::IPC, D::OpenPGP::Backend, D::OpenPGP::ErrorCodes | - | sqv, sq |
| Dpkg::OpenPGP::ErrorCodes | 135 / 21 / 78 | 0.01 | private | OPENPGP_* return codes + strings | D::Gettext | - | - |
| Dpkg::OpenPGP::KeyHandle | 123 / 21 / 63 | 0.01 | private | Key spec (keyid/userid/keyfile/keystore) | - | - | - |
| Dpkg::Package | 95 / 20 / 39 | 0.01 | private | Package-name validation; global source name | D::ErrorHandling, D::Gettext | - | - |
| Dpkg::Path | 360 / 125 / 181 | 1.05 | **public** | Path helpers, traversal check, find_command, dpkg-query wrapper | D::Arch, D::ErrorHandling, D::Gettext, D::IPC | - | `dpkg-query --control-path` |
| Dpkg::Shlibs | 205 / 20 / 126 | 0.03 | private | Library search paths (ld.so.conf, multiarch), find_library | D::Arch, D::BuildAPI, D::ErrorHandling, D::Gettext, D::Path, D::Shlibs::Objdump | - | - |
| Dpkg::Shlibs::Cppfilt | 153 / 21 / 77 | 0.01 | private | Persistent c++filt coprocess for demangling | D::ErrorHandling, D::IPC | - | `c++filt --format=<type>` (long-lived coprocess) |
| Dpkg::Shlibs::Objdump | 310 / 21 / 204 | 0.01 | private | ELF header sniffing; object registry | D::ErrorHandling, D::Gettext, D::Shlibs::Objdump::Object | - | - |
| Dpkg::Shlibs::Objdump::Object | 414 / 21 / 268 | 0.01 | private | Parse objdump -w -f -p -T -R text output | D::Arch, D::ErrorHandling, D::Gettext, D::Path | - | objdump or <gnu-triplet>-objdump |
| Dpkg::Shlibs::Symbol | 547 / 21 / 359 | 0.01 | private | symbols-file entry: tags, patterns (c++/symver/regex) | D::Arch, D::ErrorHandling, D::Gettext, D::Shlibs::Cppfilt, D::Version | - | - |
| Dpkg::Shlibs::SymbolFile | 732 / 20 / 560 | 0.01 | private | symbols file model: parse/#include/merge/diff/output | D::Arch, D::Control::Fields, D::ErrorHandling, D::Gettext, D::Interface::Storable, D::Shlibs::Symbol, D::Version | - | - |
| Dpkg::Source::Archive | 280 / 21 / 193 | 0.01 | private | tar create/extract via GNU tar subprocess | Dpkg, D::Compression::FileHandle, D::ErrorHandling, D::Gettext, D::IPC, D::Source::Functions | - | GNU tar ($Dpkg::PROGTAR) |
| Dpkg::Source::BinaryFiles | 185 / 21 / 122 | 0.01 | private | debian/source/include-binaries handling | D::ErrorHandling, D::Gettext, D::Source::Functions | - | - |
| Dpkg::Source::Format | 201 / 90 / 64 | 1.00 | **public** | debian/source/format parse/emit | D::ErrorHandling, D::Gettext, D::Interface::Storable | - | - |
| Dpkg::Source::Functions | 146 / 21 / 76 | 0.01 | private | erasedir/fixperms/fs_time/is_binary | D::ErrorHandling, D::File, D::Gettext, D::IPC | - | `rm -rf`, `chmod -R` |
| Dpkg::Source::Package | 796 / 220 / 441 | 2.04 | **public** | Source package base: .dsc, checksums, signatures, reblessing to format class | D::Checksums, D::Compression, D::Control, D::ErrorHandling, D::Gettext, D::OpenPGP, D::OpenPGP::ErrorCodes, D::Path, D::Source::Format, D::Vendor, D::Version | - | - |
| Dpkg::Source::Package::V1 | 599 / 20 / 478 | 0.01 | private | Format 1.0 (tar + .diff.gz) | Dpkg, D::Compression, D::ErrorHandling, D::Exit, D::Gettext, D::Source::Archive, D::Source::Functions, D::Source::Package, D::Source::Package::V3::Native, D::Source::Patch, D::Vendor, D::Version | - | `cp -RPp` (+tar/patch/diff via helpers) |
| Dpkg::Source::Package::V2 | 847 / 20 / 696 | 0.01 | private | Format 2.0 (orig + debian.tar + patches) | D::Changelog::Parse, D::Compression, D::Control, D::ErrorHandling, D::Exit, D::File, D::Gettext, D::Path, D::Source::Archive, D::Source::BinaryFiles, D::Source::Functions, D::Source::Package, D::Source::Patch, D::Vendor, D::Version | - | `cp -RPp`, editor (sensible-editor/$VISUAL/$EDITOR/vi) |
| Dpkg::Source::Package::V3::Bzr | 234 / 20 / 142 | 0.01 | private | Format 3.0 (bzr) | D::Compression, D::ErrorHandling, D::Exit, D::Gettext, D::Path, D::Source::Archive, D::Source::Functions, D::Source::Package | - | bzr (status, branch, remove-tree, checkout) |
| Dpkg::Source::Package::V3::Custom | 96 / 21 / 45 | 0.01 | private | Format 3.0 (custom), build-only | D::ErrorHandling, D::Gettext, D::Source::Package | - | - |
| Dpkg::Source::Package::V3::Git | 334 / 20 / 239 | 0.02 | private | Format 3.0 (git) via git bundle | D::ErrorHandling, D::Exit, D::Gettext, D::IPC, D::Path, D::Source::Functions, D::Source::Package | - | git (clone, bundle, ls-files, config, submodule, remote), `cp -f` |
| Dpkg::Source::Package::V3::Native | 161 / 20 / 101 | 0.01 | private | Format 3.0 (native) | D::Compression, D::ErrorHandling, D::Exit, D::Gettext, D::Source::Archive, D::Source::Functions, D::Source::Package, D::Vendor, D::Version | - | - |
| Dpkg::Source::Package::V3::Quilt | 293 / 20 / 194 | 0.01 | private | Format 3.0 (quilt) | D::ErrorHandling, D::Exit, D::File, D::Gettext, D::Source::Functions, D::Source::Package::V2, D::Source::Patch, D::Source::Quilt | - | - |
| Dpkg::Source::Patch | 774 / 20 / 628 | 0.01 | private | Unified diff generation (diff -u) + validation + apply (patch) | Dpkg, D::Compression::FileHandle, D::ErrorHandling, D::Gettext, D::IPC, D::Source::Functions | - | `diff -u`, GNU patch ($Dpkg::PROGPATCH) |
| Dpkg::Source::Quilt | 435 / 20 / 322 | 0.02 | private | Minimal quilt: series, .pc db, push/pop | D::ErrorHandling, D::File, D::Gettext, D::Source::Functions, D::Source::Patch, D::Vendor | - | - |
| Dpkg::Substvars | 604 / 257 / 239 | 2.04 | **public** | ${var} substitution with attributes (optional/required/implicit) | Dpkg, D::Arch, D::ErrorHandling, D::Gettext, D::Interface::Storable, D::Vendor, D::Version | - | - |
| Dpkg::SysInfo | 68 / 21 / 16 | 0.01 | private | Number of CPUs via getconf | - | - | `getconf _NPROCESSORS_ONLN` / `_NPROC_ONLN` |
| Dpkg::Vendor | 246 / 111 / 89 | 1.03 | **public** | Vendor origins files, vendor class loading, run_vendor_hook | Dpkg, D::BuildEnv, D::Control::HashCore, D::ErrorHandling, D::Gettext | - | - |
| Dpkg::Vendor::Debian | 720 / 21 / 472 | 0.01 | private | Debian hooks incl. full build-flag feature logic | Dpkg, D::Arch, D::BuildOptions, D::Control::Types, D::ErrorHandling, D::Gettext, D::Vendor::Default, D::Vendor::Ubuntu | - | - |
| Dpkg::Vendor::Default | 252 / 163 / 53 | 0.01 | private | No-op hook implementations (template) | - | - | - |
| Dpkg::Vendor::Devuan | 67 / 21 / 24 | 0.01 | private | Devuan keyrings/patch header | D::Vendor::Debian | - | - |
| Dpkg::Vendor::PureOS | 87 / 21 / 37 | 0.01 | private | PureOS keyrings/checks | D::Control::Types, D::ErrorHandling, D::Gettext, D::Vendor::Debian | - | - |
| Dpkg::Vendor::Ubuntu | 235 / 34 / 140 | 0.01 | private | Ubuntu hooks: LP bugs field, flags tweaks | D::Arch, D::Control::Types, D::ErrorHandling, D::Gettext, D::Vendor::Debian | - | - |
| Dpkg::Version | 579 / 249 / 252 | 1.05 | **public** | Debian version object, comparison, validation | D::ErrorHandling, D::Gettext | - | - |
| Test::Dpkg | 430 / 21 / 328 | 0.00 | test-only | Test-suite helpers (not installed; CPAN dist only) | - | - | - |

### Group totals (from the same data)

| Group | Modules | of which public | Code LOC | Total lines |
|---|---|---|---|---|
| infra (Dpkg, ErrorHandling, Gettext, Color, Exit, IPC, File, Path, Lock, Getopt, SysInfo, Conf, Interface::Storable, Package) | 14 | 7 | 1,178 | 2,841 |
| version (Dpkg::Version) | 1 | 1 | 252 | 579 |
| arch (Dpkg::Arch) | 1 | 1 | 438 | 746 |
| control/deb822 (Control*, Index, Email::*) | 14 | 10 | 2,157 (≈960 of them the FieldsCore data tables) | 4,113 |
| deps (Deps, Deps::*) | 7 | 7 | 886 | 2,100 |
| changelog (Changelog*) | 5 | 5 | 996 | 2,096 |
| source packages (Source::Package*, Source::Format) | 9 | 2 | 2,400 | 3,561 |
| source patch/archive (Source::Patch, Quilt, Archive, Functions, BinaryFiles) | 5 | 0 | 1,341 | 1,820 |
| compression/checksums/ar | 5 | 4 | 892 | 2,091 |
| shlibs (Shlibs*) | 6 | 0 | 1,594 | 2,361 |
| build (Build*, Substvars, Dist::Files) | 12 | 5 | 1,422 | 3,295 |
| vendor (Vendor*) | 6 | 1 | 815 | 1,607 |
| openpgp (OpenPGP*) | 7 | 0 | 862 | 1,429 |
| **Total** | **92** | **43** | **15,233** | **28,639** |

Other inventory facts:
* `Dpkg.pm` holds install-time paths. In-tree values are placeholders
  (`Dpkg.pm:95-103`, e.g. `$PROGVERSION = '1.23.x'`, `$DATADIR = '../data'`) that are
  rewritten at install by `subst_perl_file` (`scripts/Makefile.am` `install-data-hook`;
  `build-aux/subst.am:42`) or, for the CPAN distribution, by `Build.PL`'s `subst()`
  (`scripts/Build.PL.in`). `$DATADIR` can be overridden by `DPKG_DATADIR`, `$PROGTAR`/
  `$PROGPATCH`/`$PROGMAKE` by `DPKG_PROG*` (`Dpkg.pm:96-98,105`).
* The library is also packaged as a CPAN distribution named `Dpkg`
  (`configure.ac:16`, `build-aux/cpan.am` target `dist-cpan`, which copies `scripts/Dpkg*`,
  `scripts/Test`, `data/` and `scripts/t/Dpkg*`). Whether it is actually uploaded to CPAN is
  (unverified).
* `Dpkg::Archive::Ar` is used only by `scripts/dpkg-ar.pl`, which is *not* installed
  (`EXTRA_DIST` only, `scripts/Makefile.am:130`), and has no unit test (grep).

## B. Layering

Computed with `scratchpad/graph.pl`: parses top-level `use Dpkg::...` and
`use parent qw(Dpkg::...)` lines (static graph) and indented `require Dpkg::...` lines
(lazy), runs Tarjan SCC, and assigns each module the longest-path level to a leaf.

### Static dependency levels (L0 = no internal deps)

| Level | Modules |
|---|---|
| L0 | Dpkg, Dpkg::BuildEnv, Dpkg::BuildInfo, Dpkg::Color, Dpkg::Control::Types, Dpkg::Exit, Dpkg::Gettext, Dpkg::OpenPGP::KeyHandle, Dpkg::SysInfo, Dpkg::Vendor::Default |
| L1 | Dpkg::ErrorHandling, Dpkg::OpenPGP::ErrorCodes |
| L2 | Arch, Archive::Ar, BuildDriver, BuildOptions, BuildProfiles, BuildTypes, Checksums, Compression, Control::FieldsCore, Email::Address, File, Getopt, IPC, Interface::Storable, Lock, Package, Vendor::Debian, Version |
| L3 | Compression::Process, Conf, Control::HashCore::Tie, Deps::KnownFacts, Deps::Multiple, Deps::Simple, Dist::Files, Email::AddressList, Path, Shlibs::Cppfilt, Source::Format, Source::Functions, Vendor::Devuan, Vendor::PureOS, Vendor::Ubuntu |
| L4 | BuildDriver::DebianRules, BuildTree, Compression::FileHandle, Control::HashCore, Deps::AND, Deps::OR, Deps::Union, OpenPGP, OpenPGP::Backend, Shlibs::Objdump::Object, Shlibs::Symbol, Source::BinaryFiles |
| L5 | Deps, OpenPGP::Backend::{GnuPG,SOP,Sequoia}, Shlibs::Objdump, Source::Archive, Source::Patch, Vendor |
| L6 | BuildAPI, BuildFlags, Control::Fields, Source::Quilt, Substvars |
| L7 | Control::Hash, Shlibs, Shlibs::SymbolFile |
| L8 | Control |
| L9 | Control::Changelog, Control::Info, Control::Tests::Entry, Index, Source::Package |
| L10 | Changelog, Changelog::Entry, Changelog::Parse, Control::Tests, Source::Package::V3::{Bzr,Custom,Git,Native} |
| L11 | Changelog::Entry::Debian, Source::Package::V1, Source::Package::V2 |
| L12 | Changelog::Debian, Source::Package::V3::Quilt |

Static fan-in (number of library modules that `use` it): Gettext 69, ErrorHandling 68,
Dpkg 13, Version 13, IPC 11, Interface::Storable 11, Source::Functions 11, Path 10,
Vendor 9, Arch 8, Control 8, Compression 7, File 7.

**Foundation**: `Dpkg::ErrorHandling` (which itself uses `Dpkg`, `Dpkg::Gettext` and `Dpkg::Color`) is
loaded by essentially everything; `ErrorHandling` turns every fatal error into a Perl
`die` with a pre-formatted, possibly colored, localized message
(`Dpkg/ErrorHandling.pm:176-237`).

### Cycles

* **Static graph: no cycles** (Tarjan SCC finds none).
* Including lazy `require`s, one cycle appears: `Dpkg::Vendor::Debian` ↔
  `Dpkg::Vendor::Ubuntu` (Ubuntu is a subclass of Debian, `Dpkg/Vendor/Ubuntu.pm:45`;
  Debian's `extend-patch-header` hook lazily `require`s Ubuntu to reuse
  `find_launchpad_closes`, `Dpkg/Vendor/Debian.pm:75-79`).
* A cycle is *deliberately avoided* by layering: `Dpkg::Control::HashCore` must not use
  `Dpkg::Control::Fields`, because `Fields` uses `Dpkg::Vendor`, which uses `HashCore` to
  read origin files (comment at `Dpkg/Control/HashCore.pm:54-56`; `Dpkg/Vendor.pm:65,106`).
  This is why there are two layers, `FieldsCore` (static registry) and `Fields`
  (registry + vendor additions) and `HashCore` vs `Hash`.
* Other runtime-only edges: `Dpkg::Interface::Storable` lazily requires
  `Dpkg::Compression::FileHandle` (`Dpkg/Interface/Storable.pm:89,122`) although
  `FileHandle` → `Compression::Process` → `IPC`; `FieldsCore` lazily requires
  `Email::Address(List)` (`Dpkg/Control/FieldsCore.pm:1216,1237`); `BuildTree` lazily
  requires `Control::Info` and `BuildDriver` (`Dpkg/BuildTree.pm:114-115`).

### Import-time side effects (matter for any replacement or binding)

* `use Dpkg::Control` (or anything that loads `Dpkg::Control::Fields`) **executes the vendor
  hook `register-custom-fields` at module load** (`Dpkg/Control/Fields.pm:44-60`), which reads
  `$Dpkg::CONFDIR/origins/default` (or `$DPKG_ORIGINS_DIR`) and loads a `Dpkg::Vendor::*`
  class. (verified: on this machine, where `/etc/dpkg/origins/default → ubuntu`,
  `perl -MDpkg::Control` loads `Dpkg/Vendor/Ubuntu.pm`; with the test origins it loads
  `Dpkg/Vendor/Debian.pm`.)
* `Dpkg::Gettext` decides at `BEGIN` time whether to use `Locale::gettext` and installs
  closures into the symbol table (`Dpkg/Gettext.pm:124-178`).
* `Dpkg::Source::Package::V3::Git` deletes `GIT_DIR`, `GIT_INDEX_FILE`, `GIT_OBJECT_DIRECTORY`, `GIT_ALTERNATE_OBJECT_DIRECTORIES` and `GIT_WORK_TREE` from `%ENV` at load
  (`Dpkg/Source/Package/V3/Git.pm:55-59`).
* `Dpkg::Exit` and `Dpkg::Shlibs::Cppfilt` install `END` blocks (`Dpkg/Exit.pm:107-110`,
  `Dpkg/Shlibs/Cppfilt.pm:138-143`).

### Modules loaded per entry point (measured with `%INC`)

`DPKG_DATADIR=data DPKG_ORIGINS_DIR=scripts/t/origins perl -Iscripts -e 'require X; ...count keys %INC'`

| Entry point | Dpkg::* modules loaded | All modules in `%INC` (incl. core) |
|---|---|---|
| Dpkg::Version | 5 | 30 |
| Dpkg::Checksums | 5 | 30 |
| Dpkg::Arch | 6 | 31 |
| Dpkg::Shlibs::Objdump | 10 | 45 |
| Dpkg::Vendor | 11 | 37 |
| Dpkg::BuildFlags | 12 | 38 |
| Dpkg::Substvars | 14 | 40 |
| Dpkg::Deps | 16 | 42 |
| Dpkg::Control | 16 | 42 |
| Dpkg::Control::Info / Dpkg::Index | 17 | 43 |
| Dpkg::Changelog::Parse | 18 | 44 |
| Dpkg::Shlibs::SymbolFile | 20 | 46 |
| Dpkg::Changelog::Debian | 24 | 55 |
| Dpkg::Source::Package | 26 | 65 |
| Dpkg::Source::Package::V3::Quilt | 39 | 91 |

Load cost is small (measured with `/usr/bin/time`): `perl -e1` 0.00 s/5.3 MB;
`-MDpkg::Version` 0.01 s/9.9 MB; `-MDpkg::Control` 0.02 s/10.7 MB;
`-MDpkg::Source::Package::V3::Quilt` 0.04 s/17.7 MB. Throughput samples:
`version_compare` ≈ 40,000 calls/s; `Dpkg::Control->parse` over this machine's
`/var/lib/dpkg/status` (2.28 MB, 2,215 stanzas) took 0.16 s CPU.

## C. Deep dives

### C.1 `Dpkg::Control*` — deb822 parsing and the field registry

**Object model (public: Hash, HashCore, Control, Fields, FieldsCore, Info, Changelog, Tests, Tests::Entry).**
Class chain: `Dpkg::Control` → `Dpkg::Control::Hash` → `Dpkg::Control::HashCore` →
`Dpkg::Interface::Storable` (`Dpkg/Control.pm:178`, `Dpkg/Control/Hash.pm:38`,
`Dpkg/Control/HashCore.pm:58`).

* A `HashCore` object is a **blessed scalar reference** to an options hash
  (`Dpkg/Control/HashCore.pm:120-131`) — not a hash ref — so that `use overload '%{}'`
  (`:60-62`) can make `$ctrl->{Field}` dereference into a **tied hash** stored in
  `$$self->{fields}` (`:133`) without infinite recursion. `eq` is overloaded as string
  comparison of the serialized stanza (`:62`), and `""` (stringification = `output()`) comes
  from `Dpkg::Interface::Storable` (`Dpkg/Interface/Storable.pm:39-41,139-145`).
* The tie class `Dpkg::Control::HashCore::Tie` (private, 0.01) extends `Tie::ExtraHash`
  (`Dpkg/Control/HashCore/Tie.pm:44-45`). Its state is `[ {lc(key) => value}, $parent_options ]`
  (`:47-52,79`). Semantics: keys are **case-insensitive** (`FETCH/STORE/EXISTS/DELETE` all
  `lc` the key, `:82-120`); **insertion order** is recorded in the parent's `in_order` array
  as the *canonical capitalisation* from `field_capitalize()` (`:95`); iteration
  (`FIRSTKEY`/`NEXTKEY`) walks `in_order` (`:122-144`). `NEXTKEY` rescans `in_order` from the
  start on every call, so iterating `keys %$ctrl` is O(n²) in the number of fields
  (inferred from code; not benchmarked). (verified) `version: 1` in input appears as key
  `Version` in `keys %$ctrl`.
* There is a reference cycle object ↔ tie; `DESTROY` breaks it by deleting `fields`
  (`Dpkg/Control/HashCore.pm:141-149`).
* Options (documented in POD, `:63-114`): `name`, `allow_pgp`, `allow_duplicate`,
  `keep_duplicate` (duplicate values become array refs, `:227-245`), `drop_empty`,
  `is_pgp_signed` (set by the parser).

**Parser** (`HashCore::parse`, `Dpkg/Control/HashCore.pm:197-305`) is line-oriented
(`while (<$fh>)`) and parses *one stanza per call* (returns true if a field was seen):
1. `chomp`, keep the raw line as `$armor`, strip trailing whitespace (`:211-213`).
2. Leading blank lines are skipped; lines whose first char is `#` are comments and skipped
   anywhere (`:215-218`).
3. `split /\s*:\s*/, $_, 2`; a field line is one whose name matches `^\S+?$`; a leading `-`
   is an error (`:221-226`). Duplicate fields are an error unless `allow_duplicate`
   (`:227-245`).
4. Continuation lines `^\s(\s*\S.*)$` are appended with `"\n"`; a continuation consisting only
   of dots loses one dot (deb822 " ." escaping, `:247-255`).
5. An empty line (after stripping trailing whitespace) ends the stanza; if a
   `-----BEGIN PGP SIGNED MESSAGE-----` header was seen, it then requires and skips the
   signature block, setting `is_pgp_signed` (`:256-283`). The signed-message header is only
   allowed when `allow_pgp` and before any field (`:284-293`). Signatures are **not verified**
   here (comment `:278-279`); verification is in `Dpkg::OpenPGP`.
6. Anything else → `parse_error` "line with unknown format" (`:294-297`). Errors use `$.` as
   the line number (`:179-184`).

**Serializer** (`output`, `:356-407`): order = `out_order` (set from the field registry by
type) with unknown fields appended **sorted alphabetically**, or `in_order` if no output
order is set (`:360-377`); multi-line values are escaped (`" ."` for empty or dots-only
lines); `drop_empty` skips whitespace-only values.
`apply_substvars` (`:432-492`) expands `${...}` via `Dpkg::Substvars`, synthesises fields
from *implicit* substvars, then normalises comma/line separated fields (drops empty
entries, duplicate commas) and turns `${}` into `$`.

**Control types** (`Dpkg/Control/Types.pm:65-109`): 16 bit-flag constants
(`CTRL_TMPL_SRC`=1<<0 … `CTRL_FILE_BUILDINFO`=1<<15) plus 6 backwards-compat aliases.
`Dpkg::Control::set_options(type => ...)` derives `allow_pgp` (only for `.dsc`, `.changes`,
`Release`), `drop_empty` (everything except `debian/control` templates), a human `name`, and
the output order (`Dpkg/Control.pm:265-309`). Quirk: the branch for
`CTRL_COPYRIGHT_LICENSE` tests `CTRL_COPYRIGHT_HEADER` a second time (`:277` and `:281`), so
the "license stanza of copyright file" name is unreachable.

**Field registry** (`Dpkg/Control/FieldsCore.pm`, public 1.05): `%FIELDS`
(`:84-650`) maps lower-case names to `{ name, allowed (bitmask of CTRL_*), separator,
dependency ('normal'/'union'), dep_order, default }`; `%FIELD_ORDER` (`:702-1042`) lists the
canonical output order per control type. (verified) 118 registered fields; 18 with a
`dependency` type; 48 with a `separator`; 2 with a default; 16 order lists (sizes 2–40).
About 960 of the module's 1,180 code lines are these data tables. Functions (`:1055-1440`):
`field_capitalize` (registry name, else `Ucfirst-Each-Dash-Part`), `field_is_official`,
`field_is_allowed_in`, `field_transfer_single/all` (implements the `X[SBC]*-` prefix export
rules, `:1125-1161`), `field_parse_maintainer/uploaders` (via `Dpkg::Email::*`),
`field_parse_binary_source`, `field_list_src_dep/pkg_dep` (sorted by `dep_order`),
`field_get_dep_type/sep_type/default_value`, `field_ordered_list`, and the **mutators**
`field_register`, `field_insert_after/before`, which mutate the global tables. The
`CTRL_FILE_STATUS` order list is commented as "Same as fieldinfos in «lib/dpkg/parse.c»"
(`:983`) — an explicit manual sync point with C.

`Dpkg::Control::Fields` re-exports FieldsCore and, **at load time**, applies the vendor's
`register-custom-fields` operations (`register`, `insert_before`, `insert_after`;
`Dpkg/Control/Fields.pm:44-60`). Ubuntu uses this to add `Launchpad-Bugs-Fixed`
(`Dpkg/Vendor/Ubuntu.pm:88-98`).

`Dpkg::Control::Info` parses `debian/control` (first stanza `CTRL_TMPL_SRC` must have
`Source`, following stanzas `CTRL_TMPL_PKG` must have `Package` and `Architecture`,
`Dpkg/Control/Info.pm:110-133`) and overloads `@{}` to `[source, packages...]` (`:40-41`).
`Dpkg::Control::Tests(::Entry)` is an `Index` of `CTRL_TESTS` stanzas requiring `Tests` or
`Test-Command` (`Dpkg/Control/Tests/Entry.pm:75-86`).

**`Dpkg::Index`** (public 3.00): ordered map key → `Dpkg::Control`, overloads `@{}` to the
key order (`Dpkg/Index.pm:39-41`). The key function is a **closure chosen by control type**
(`:159-222`; e.g. `Package_Version_Architecture` for Packages files when
`unique_tuple_key`) and can be supplied by callers (`get_key_func` option, documented at
`:97`). `get_keys(%criteria)` accepts plain strings, `Regexp` objects, or **code refs** per
field (`:319-342`); `sort(\&func)` takes a comparator (`:423-432`). For `CTRL_TESTS` the key
closure captures `$self` (`:182-184`), creating a reference cycle.

### C.2 `Dpkg::Deps*` — dependency parsing, restrictions, simplification

* **Parsing** (`deps_parse`, `Dpkg/Deps.pm:263-361`): normalise newlines, split on `,` then
  `|`, build `Dpkg::Deps::Simple` per alternative, wrap in `OR` (if >1) and the whole in
  `AND` (or `Union` with `union => 1`). Options: `use_arch`, `reduce_arch`, `host_arch`,
  `build_arch`, `use_profiles`, `reduce_profiles`, `build_profiles`, `reduce_restrictions`,
  `union`, `virtual`, `build_dep`, `tests_dep`. Parse failures return `undef` after a
  `warning` (`:316-319`) rather than dying.
* **Single dependency grammar** — one extended regex (`Dpkg/Deps/Simple.pm:176-202`):
  `name[:archqual] [(rel version)] [[arch list]] [<profile list>...]`. Package name
  `[a-zA-Z0-9][a-zA-Z0-9+.-]*` (with leading `@` allowed for `tests_dep`, `:170-174`). `:native`
  only with `build_dep` (`:207`). Version is *not* validated at parse (`[^\)\s]+`, `:186`) and
  becomes a `Dpkg::Version`. Relations `<`/`>` are accepted with a deprecation warning and mean `<=`/`>=`
  (`Dpkg/Version.pm:380-399`); an operator is mandatory (C accepts `pkg (1.0)` as `=`, G). Arch list goes through `debarch_list_parse` (dies on invalid
  names), profiles through `parse_build_profiles`.
* **Arch restrictions**: `arch_is_concerned` delegates to `Dpkg::Arch::debarch_is_concerned`
  (negated `!arch` semantics, `Dpkg/Arch.pm:655-681`); `reduce_arch` empties a dep that does
  not apply (`Dpkg/Deps/Simple.pm:488-515`).
* **Profile restrictions** (`Dpkg/BuildProfiles.pm`): a formula is a list of `<...>` lists;
  the dep applies if **any** list has **all** terms satisfied (term `!p` satisfied when `p` not
  active) (`:174-203`). Profile names `[?/;:=@%*~_A-Za-z0-9+.-]+` (`:51-59`).
* **Implication** (`implies`, `Dpkg/Deps/Simple.pm:395-457`; `AND.pm:74-98`; `OR.pm:74-99`;
  `Union` never implies, `Union.pm:74-77`). Three-valued result: 1 (p ⇒ q), 0 (p ⇒ ¬q), undef
  (unknown). Version reasoning is the hand-written truth table `deps_eval_implication`
  (`Dpkg/Deps.pm:84-164`). Arch qualifiers must be equal; restriction formulas must be a
  superset; arch lists are compared by `_arch_is_superset` (`Simple.pm:285-335`).
* **Simplification** (`simplify_deps`, `AND.pm:140-165`, `OR.pm:141-158`,
  `Union.pm:91-103`, `Simple.pm:594-599`): drop deps already satisfied by `KnownFacts`, drop
  deps implied by known deps or by other members, collapse OR alternatives;
  `Union` merges same-package entries through `merge_union` (`Simple.pm:621-656`).
* **`Dpkg::Deps::KnownFacts`** (public 2.00) holds installed packages
  (`add_installed_package(pkg, ver, arch, multiarch)`) and virtual provides; `evaluate_simple_dep`
  implements multi-arch matching rules: no qualifier matches host/all or `Multi-Arch: foreign`;
  `:any` requires `Multi-Arch: allowed`; `:native` matches build arch/all and refuses
  `foreign` (`Dpkg/Deps/KnownFacts.pm:113-143,158-191`). It calls
  `Dpkg::Arch::get_host_arch` without `use Dpkg::Arch` (relies on `Dpkg::Deps` having loaded it).
* `deps_compare` orders by package, then a fixed relation ranking, then version **string**
  (`Dpkg/Deps.pm:403-439`); `deps_iterate` walks the tree with a callback using `__SUB__`
  (`:375-392`).

**Verified anomalies** (present upstream, untested by `t/Dpkg_Deps.t`; a reimplementation
must decide whether to be bug-compatible):
1. `Dpkg/Deps.pm:127` compares `$v_p >= $v_p` (typo for `$v_q`), so for p = `>=`/`>>` and
   q = `<<` the result is always 0. (verified) `deps_parse("a (>= 5)")->implies(deps_parse("a (<< 10)"))`
   returns 0 ("p implies not q"), although `a 6` satisfies both. Line dates from 2009
   (`git blame`, commit 10badb3c2d).
2. `Dpkg/Deps/Simple.pm:287-288`: `my $p_arch_neg = defined $p and $p->[0] =~ /^!/;` —
   `and` binds looser than `=`, so `$p_arch_neg` is just `defined $p` (Deparse confirms
   `((my $p_arch_neg = defined($p)) and ...)`). Every non-empty arch list is therefore treated
   as negated. (verified) `a [amd64]` implies `a [amd64 i386]` → 1 and `a [amd64 i386]` implies
   `a [amd64]` → undef, the reverse of the documented intent in the comment at `:298-301`.
   Introduced 2018 (commit 738c8d5d54).
3. `Dpkg/Deps.pm:422-423`: `my $aundef = not defined $a or $a->is_empty();` — the
   `is_empty()` part is discarded (Deparse: `((my $aundef = (!defined($a))) or $a->is_empty)`).

### C.3 `Dpkg::Version`

* Object = hash `{epoch, version, revision, no_epoch, no_revision}` (`Dpkg/Version.pm:105-133`).
  Epoch: everything before the **first** `:` (`^([^:]*):(.+)$`); revision: after the **last**
  `-` (greedy `(.*)-(.*)$`). Missing parts become `0` with a `no_*` flag so `as_string`
  round-trips (`:301-311`).
* Overloads `<=>`, `cmp`, `""`, `bool` (bool = `is_valid`, after a semantic change documented
  at `:141-150`) with `fallback => 1` (`:71-76`); comparing with a plain string auto-constructs
  a `Dpkg::Version` (`:264-275`).
* Algorithm (`:414-488`): `version_split_digits` splits with look-around
  `(?<=\d)(?=\D)|(?<=\D)(?=\d)`; digit runs compare **numerically with Perl `<=>`**;
  non-digit runs compare char-by-char with weights `~` = −1, end of string = 0, digit = value+1
  (reached only when a digit run meets a non-digit run), letters = ASCII code, other = code+256
  (same ordering as C `order()`, `lib/dpkg/version.c:66-79`, where digits weigh 0).
* `version_check` (`:500-545`) returns `(ok, msg)` in list context or bool in scalar
  (`wantarray`). Allowed chars `[-+:.0-9a-zA-Z~]`; epoch must be `^\d*$`; upstream must start
  with a digit.
* **Verified divergences from C** (`lib/dpkg/version.c:66-155`, `lib/dpkg/parsehelp.c:243-323`):
  * Huge digit runs: Perl compares with numeric `<=>`, exact up to 2^64−1 (verified: `1.9007199254740993` > `...992` and `1.18446744073709551615` > `...614` both work) but digit runs ≥ 2^64 are converted to doubles; C compares digit-by-digit.
    `1.18446744073709551617` vs `1.18446744073709551616`: C (`build/src/dpkg --compare-versions`)
    says greater, Perl `version_compare` returns 0. Same for `1.99999999999999999999` vs
    `...98`.
  * Epoch range: Perl accepts `99999999999:1.0` as valid; C rejects "epoch in version is too big"
    (`parsehelp.c:281-282`).
  * C trims surrounding blanks and rejects embedded spaces with specific errors
    (`parsehelp.c:250-267`); Perl has no trimming and reports a generic invalid character.
  * C validates revision characters separately (`.+~` plus alnum, `parsehelp.c:316-321`);
    Perl validates the whole string against one class.
  The shipped test `t/Dpkg_Version.t` runs every comparison both in Perl and through
  `dpkg --compare-versions` (`t/Dpkg_Version.t:43-49,155-172`), i.e. the two implementations
  are already parity-tested on 43 fixture pairs (`__DATA__`, `:238+`), none of which hit these
  edge cases.

### C.4 `Dpkg::Arch` and the `data/` tables

* Tables read lazily from `$Dpkg::DATADIR` (`Dpkg/Arch.pm:262-341`). Sizes (non-comment rows): cputable 34, ostable 20, tupletable 41, abitable 2; (verified) `get_valid_arches()` yields 201 Debian architectures. 10 cputable rows and all 20 ostable rows use regex metacharacters (e.g. `linux[^-]*-musl`); the syntax used is the portable ERE subset, but the semantics are "whatever Perl's regex engine does":
  * `data/cputable`: `debcpu  gnucpu  regex  bits  endianness` (e.g.
    `i386 i686 (i[34567]86|pentium) 32 little`). The **third column is a Perl regex** applied as
    `/^$cputable_regex{$cpu}$/` when mapping a GNU triplet back to a Debian CPU
    (`:285-291,368-373`).
  * `data/ostable`: `abi-libc-os  gnu-os  regex` (`:297-303`), matched as
    `/^(.*-)?$ostable_regex{$os}$/` (`:376`).
  * `data/tupletable`: `abi-libc-os-cpu  debarch`, with a `<cpu>` placeholder expanded over
    every cputable CPU; first match wins (`:315-341`).
  * `data/abitable`: per-ABI bit width overrides (e.g. x32) (`:306-313`).
* Mappings: `debarch_to_debtuple` (4-tuple; list or hash-ref via `wantarray`, `:432-459`; a
  legacy `linux-<arch>` prefix is stripped), `debtuple_to_debarch`, `debtuple_to_gnutriplet`,
  `gnutriplet_to_debtuple` (regex scan in table order, `:356-384`), `gnutriplet_to_multiarch`
  (only rewrites `i[4567]86` → `i386`, `:392-402`), `debarch_to_multiarch`,
  `debarch_to_abiattrs` (`(bits, endian)`), `debarch_to_cpubits`, `get_valid_arches`.
* Wildcards (`:487-601`): `any`, `<os>-any`, `any-<cpu>`, `<abi>-<libc>-any-any` etc. are
  expanded to 4-tuples where `any` matches any component (`debarch_is`), `debarch_eq`
  compares tuples, `debarch_is_concerned` evaluates `[a b !c]` lists, `debarch_is_invalid`
  enforces `^!?[a-zA-Z0-9][a-zA-Z0-9-]*$` (`:620-630`), same rule as C
  `dpkg_arch_name_is_invalid` (`lib/dpkg/arch.c:57-81`).
* Host/build detection: `get_raw_build_arch` runs **`dpkg --print-architecture`** (the C
  program) (`:125-143`); `get_host_gnu_type` runs **`$CC -dumpmachine`** (default `gcc`)
  (`:157-185`); env overrides `DEB_BUILD_ARCH`/`DEB_HOST_ARCH` via `Dpkg::BuildEnv`.
* Coupling with C: the C build's compiled-in `ARCHITECTURE`, `ARCHITECTURE_CPU` and
  `ARCHITECTURE_OS` are computed at **configure time by running
  `scripts/dpkg-architecture.pl`** (i.e. `Dpkg::Arch`) (`m4/dpkg-arch.m4`, macro
  `_DPKG_ARCHITECTURE`). libdpkg itself does not read the tables at runtime (grep for
  `cputable` in C finds nothing). So the C build depends on Perl `Dpkg::Arch` at configure
  time, and `Dpkg::Arch` depends on C `dpkg` at run time.
* Test: `t/Dpkg_Arch.t` produces 6,967 assertions by iterating `get_valid_arches()` and
  wildcard tables (`:38-189`).


### C.5 `Dpkg::Changelog*`

* **Model.** `Dpkg::Changelog` (public 2.00) is an abstract container: `$self->{data}` is the
  entry list (newest first), overloaded as `@{}` (`Dpkg/Changelog.pm:49-50`), plus
  `parse_errors` and an `unparsed_tail` (text after an Emacs/vim modeline or an "ancient"
  delimiter, so that `output()` round-trips the file byte-for-byte, `:491-505`).
  `Dpkg::Changelog::Entry` (public 1.01) stores raw `header`, `changes` (lines),
  `trailer` and three `blank_after_*` arrays (`Dpkg/Changelog/Entry.pm:56-70`) and
  overloads `""`/`eq` (`:40-43`); accessor methods (`get_source`, `get_version`, …) return
  undef in the base class and are implemented by `Entry::Debian`.
* **Debian parser** (`Dpkg/Changelog/Debian.pm:134-255`) is a 4-state machine
  (`FIRST_HEADING`, `NEXT_OR_EOF`, `START_CHANGES`, `CHANGES_OR_TRAILER`, `:56-61`)
  over lines: header lines (`match_header`), `Local variables:`/`vim:` modelines (rest of file
  becomes the unparsed tail), RCS keywords / `# ` / `/* */` comments skipped, a large
  "ancient changelog" delimiter regex (`:63-118`) ends parsing, trailers (`match_trailer`),
  change lines (2+ leading spaces), blank lines. Errors are *collected*, not fatal
  (`parse_error` pushes `[file, line, msg, line_text]` and optionally warns,
  `Dpkg/Changelog.pm:144-156`). Parsing stops early when the requested range is satisfied
  (`abort_early`, `:449-476`).
* **Entry::Debian** (public 2.00): header regex `^(\w[-+0-9a-z.]*) \(([^() \t]+)\)((\s+[-+0-9a-z.]+)+)\;(.*?)\s*$`
  (`Dpkg/Changelog/Entry/Debian.pm:57-65`); `key=value` options validated
  (`Urgency`, `Binary-Only=yes`, `X[BCS]+-*`, others "unknown") (`:150-197`); trailer regex
  with exactly two spaces before an RFC-2822-ish date parsed by `Time::Piece->strptime(…,'%d %b %Y %T %z')`
  under `local $ENV{LC_ALL}='C'` (`:70-86,199-241`); `find_closes` scans
  `closes:\s*(?:bug)?\#?\s?\d+…` with `/pigx` and `${^MATCH}` (`:413-427`);
  `get_change_items` splits change text into bullet items (`:113-138`).
* **Ranges and output formats** (`Dpkg/Changelog.pm:246-440,526-703`): range options
  `since/until/from/to/count/offset/reverse/all` with fuzzy resolution of non-existent versions
  via `version_compare_relation` and many warnings (`_sanitize_range`). `format_range('dpkg')`
  collapses the range into one `Dpkg::Control::Changelog` (highest urgency by the `@URGENCIES`
  ordering, concatenated changes, union of `Closes`), `format_range('rfc822')` emits one stanza
  per entry; both run the vendor hook `post-process-changelog-entry` (`:578,606`). Scalar
  context returns a `Dpkg::Index` (`:692-702`).
* **Plug-in mechanism** (`Dpkg/Changelog/Parse.pm:47-68,138-200`): the format is taken from a
  `changelog-format: <name>` line in the **last 4 KiB** of the file (default `debian`); the
  module name `Dpkg::Changelog::<Ucfirst lc name>` is loaded by **string `eval "require ..."`**
  and must `isa('Dpkg::Changelog')`. The name is restricted to `[0-9a-z]+` (`:62`), so the
  string eval cannot inject code. This mechanism is declared a **stable API** in
  `doc/README.api` ("custom changelog parsers as Dpkg::Changelog derived modules … since dpkg
  1.18.8"). Third-party parsers on real systems: (unverified).

### C.6 `Dpkg::Source::*` — source package formats

**Base class** `Dpkg::Source::Package` (public 2.04):
* `new(filename => X.dsc)` loads the `.dsc` as `Dpkg::Control(CTRL_DSC)` (accepts OpenPGP
  armor), requires `Source`, `Version`, `Files`, imports checksums
  (`Dpkg/Source/Package.pm:228-323`).
* `upgrade_object_type` parses `Format` with `Dpkg::Source::Format`
  (`^(\d+)(?:\.(\d+))?(?:\s+\(([a-z0-9]+)\))?$`, `Dpkg/Source/Format.pm:109-121`), builds
  `Dpkg::Source::Package::V<major>[::<Ucfirst variant>]`, loads it with **string `eval
  require`**, calls `prerequisites()` if present and **re-blesses `$self` into that class**
  (`Dpkg/Source/Package.pm:325-349`). Concrete classes: `V1` (1.0), `V2` (2.0),
  `V3::Native`, `V3::Quilt`, `V3::Git`, `V3::Bzr`, `V3::Custom`. Any installed
  `Dpkg::Source::Package::V3::Foo` would be picked up for `3.0 (foo)` (inferred from code).
* Defaults: a big `diff_ignore` regex for VCS/editor droppings and a 36-entry `tar_ignore` glob list (counted with `get_default_tar_ignore_pattern()`) (`:60-118`), extended in `init_options` (`:265-300`).
* Verification: `check_checksums` (re-hash every file; error/warn on weak-only),
  `check_signature` (keyrings: `--signer-cert`, deprecated `~/.gnupg/trustedkeys.*`, and vendor
  hook `package-keyrings`, `:541-577`), `check_original_tarball_signature` against
  `debian/upstream/signing-key.asc` (`:498-517`); failures go through `report_verify`
  (= `error` or `warning`, `:256-260`).
* `extract` → `do_extract` (subclass), then **`check_directory_traversal`** over
  `debian/` (`Dpkg/Path.pm:220-250`: every path, following symlinks, must `realpath` inside
  the root) and writes `debian/source/format` (`:604-656`). `build` → `do_build`;
  `commit` → `do_commit`; `write_dsc` applies substvars and writes the `.dsc` (`:713-745`).

**Formats** (all private 0.0x):
* **1.0** (`Dpkg/Source/Package/V1.pm`, 478 code LOC): native tarball or orig + `.diff.gz`;
  gzip only (`:91-93`); `-s[akpursnAKPUR]` "source styles" (`:154-160,314-378`); handles
  unpacked `.orig` directories; uses `cp -RPp` (`:257`).
* **2.0** (`V2.pm`, 696 code LOC): orig(+`orig-<comp>`) tarballs, `debian.tar`, patches in
  `debian/patches` applied in order; at build time unpacks a pristine copy into a temp dir,
  applies patches, then **generates a new patch with `diff -u`** against the tree
  (`_generate_patch`, `:442-581`), checks binary files via `Dpkg::Source::BinaryFiles`,
  supports `--commit` (opens an editor, `:820-834`), patch header extended via vendor hook
  `extend-patch-header` (`:721`).
* **3.0 (quilt)** (`V3/Quilt.pm`): subclass of V2 using `Dpkg::Source::Quilt`; vendor-specific
  series file `debian/patches/<vendor>.series` with a `series` symlink (`Quilt.pm:286-294`,
  `V3/Quilt.pm:126-180`); `.pc/` database compatible with quilt (`.version` = 2,
  `applied-patches`, backup dirs) (`Dpkg/Source/Quilt.pm:65-103`). The class defines methods
  named `push`/`pop` (uses `CORE::push`).
* **3.0 (native)** (`V3/Native.pm`): one tarball; revisions in native versions are an error
  unless vendor hook `has-fuzzy-native-source` (Debian: true) downgrades to a warning
  (`:93-106`).
* **3.0 (git)** / **3.0 (bzr)**: shell out to `git` (bundle/clone/ls-files/submodule) and `bzr`;
  Git deletes five `GIT_*` variables at load (`V3/Git.pm:55-59`).
* **3.0 (custom)**: build-only, `--target-format` sets the `Format` field and files are given
  on the command line (`V3/Custom.pm:56-86`).

**`Dpkg::Source::Patch`** (private, 628 code LOC) is a subclass of
`Dpkg::Compression::FileHandle` (`:51`), i.e. the patch object *is* a (possibly compressed)
tied filehandle.
* *Generation*: `add_diff_file` runs `diff -u [-p] -L label -L label -- old new` under
  `LC_ALL=C TZ=UTC0` and copies its output, detecting binary output and "no newline" markers
  (`:97-180`); `add_diff_directory` walks both trees with `File::Find`, refuses to represent
  type changes, devices, sockets, symlink retargets, ignores deletions unless
  `include_removal`, and can reorder by an existing patch (`order_from`) (`:182-347`).
* *Validation* (`analyze`, `:449-617`) — the security-relevant part: skips leading
  comments as the patch header; requires `---`/`+++` pairs; **rejects C-style quoted
  filenames** (`_fetch_filename`, `:394-411`), names ending in `.dpkg-orig` (`:486-489`),
  paths containing `/../` (`:513-515`), `/dev/null`→`/dev/null` diffs, patching anything that is
  not a plain file (`:542-547`); warns on hard links (`:548-551`) and duplicate files
  (`fatal_dupes` makes it an error); checks hunk line counts against `@@` headers
  (`:569-605`); `_intuit_file_patched` replicates GNU patch's choice between old/new names
  (`:413-446`). It records directories to create, files patched and order.
  A loop at `:516-521` that used to check for symlinked path components no longer does
  anything: the `-l $path` test was removed on 2026-03-25 by d05856d7e ("Remove check for
  patching via a symlink … We already rely on the directory traversal checks performed by GNU
  patch"). The code therefore **depends on GNU patch's own traversal protection**;
  `$Dpkg::PROGPATCH` is documented as "GNU patch (or another implementation that is
  directory traversal resistant)" (`Dpkg.pm:67-70`). Historical hardening commits:
  00aa1a864 (CVE-2010-1679), 5348cbc98, a12eb5895, 7a6c03cb3.
* *Application* (`apply`, `:629-693`): `patch -t -F 0 -N -p1 -u -V never -b -z .dpkg-orig`
  with `LC_ALL=C PATCH_GET=0` and `POSIXLY_CORRECT` removed; on failure prints captured
  stdout/stderr; then resets mtimes of all patched files to one timestamp and removes backups.
  `check_apply` does a `--dry-run` (`:696-739`). Quilt push uses `-E -B .pc/<patch>/
  --reject-file=-` (`Dpkg/Source/Quilt.pm:185-215`) and on failure restores backups.

**`Dpkg::Source::Archive`** (private): also a `Compression::FileHandle` subclass. `create`
spawns `$Dpkg::PROGTAR -cf - --format=gnu --sort=name --mtime @<SOURCE_DATE_EPOCH|now>
--clamp-mtime --null --numeric-owner --owner=0 --group=0 -T -` with `TAR_OPTIONS` deleted, and
file names are fed NUL-separated on stdin (`Dpkg/Source/Archive.pm:53-120`); i.e. GNU tar
specific flags. `extract` unpacks into a temp dir next to the destination with
`--no-same-permissions --no-same-owner`, `fixperms` (`chmod -R` to umask-derived modes,
`Dpkg/Source/Functions.pm:68-88`), then either renames the single top-level directory into
place or, for `in_place`, moves entries one by one, warning when a destination symlink points
outside the root (`:134-270`). `erasedir` shells out to `rm -rf` (`Functions.pm:53-66`).

### C.7 `Dpkg::Shlibs::*` — shared library and symbols handling (all private)

* **Library search** (`Dpkg/Shlibs.pm`): paths = rpath, `-l` custom dirs, `LD_LIBRARY_PATH`
  (only with build API 0; API ≥1 makes in-tree use an error, `:107-132`), multiarch dirs when
  cross-building, `/lib /usr/lib`, recursive `/etc/ld.so.conf` (`include` globs, cycle guard,
  `:65-94`), then `/lib32 /usr/lib32 /lib64 /usr/lib64`. `find_library` filters candidates
  by ELF ABI string (`:166-195`).
* **ELF sniffing** (`Dpkg/Shlibs/Objdump.pm:227-289`): reads 64 header bytes and builds an ABI
  key `ELF:<bits>:<endian>:<mach>:<flags>` with per-machine flag masks (ARM EABI, MIPS ABI,
  RISC-V float ABI, LoongArch…, `:90-225`). Pure `unpack`, no external tool.
* **Symbol extraction** (`Objdump/Object.pm`): runs `objdump -w -f -p -T -R` (or
  `<gnu-triplet>-objdump` when cross-building, `:74-81,99-105`) under `LC_ALL=C`, and parses
  its text output section by section (`:108-198`): `DYNAMIC SYMBOL TABLE` (fixed-width flag
  column `.{7}`, version and visibility, `:228-287`), relocations (marks symbols with
  `R_*_COPY` relocations as undefined, `:289-315`), `Dynamic Section` (NEEDED/SONAME/RPATH/
  RUNPATH), `Program Header` (INTERP), `Version References`. **Correctness depends on the exact
  textual format of GNU binutils' objdump.**
* **Symbols file model** (`Shlibs/SymbolFile.pm`, 560 code LOC; `Shlibs/Symbol.pm`, 359): parses
  `deb-symbols(5)` files with `#include "file"` (with tag inheritance), `| alt deps`,
  `* Field: value`, `#MISSING:`/`#DEPRECATED:` markers (`SymbolFile.pm:214-285`); a built-in
  internal-symbol list (ARM, MIPS, PPC `_savegpr_*`…) (`:44-99`); `merge_symbols`,
  `get_new_symbols`, `get_lost_symbols`, `lookup_symbol`, `get_dependency`, and output in
  template or final mode with `#PACKAGE#`/`#CURVER#` substitution (`:298-367`).
* **Tags and patterns** (`Symbol.pm`): tags `(t1|t2=v)` with preserved order
  (`:75-99,240-270`); arch tags `arch`, `arch-bits`, `arch-endian` (`:314-328`); pattern
  types `c++` (alias via demangled name), `symver` (alias by version node), and `regex`
  (`:155-202`). **The `regex` tag compiles the symbols-file text as a Perl regex**
  (`qr/$regex/`, `Symbol.pm:197`), and `man/deb-src-symbols.pod:353` documents it as "the
  perl regular expression specified in the symbol name field" — the on-disk file format
  semantics are defined in terms of Perl regexes. Patterns produce cloned match symbols via
  `Storable::dclone` (`Symbol.pm:68-73,381-398`).
* **C++ demangling** (`Shlibs/Cppfilt.pm`): one long-lived `c++filt --format=<type>`
  coprocess per type, line-at-a-time request/response with a 1-entry cache; post-processes
  `>>` into `> >` except for `operator>>` (`:52-116`); terminated from an `END` block
  (`:123-143`). Symbols files therefore depend on the binutils demangler's output text.

### C.8 Build modules

* **`Dpkg::BuildEnv`** (private): wrapper over `%ENV` that records accessed and modified
  variable names (`Dpkg/BuildEnv.pm:35-102`) — used by `dpkg-buildflags`/`dpkg-genbuildinfo`.
* **`Dpkg::BuildOptions`** (public 1.02): parses `DEB_BUILD_OPTIONS`-style strings
  `^([a-z][a-z0-9_-]*)(?:=(\S*))?$`; `terse/noopt/nostrip/nocheck` drop values; `parallel`
  must be digits (`Dpkg/BuildOptions.pm:101-144`); `parse_features(area, \%features)` applies
  `+feat`/`-feat`/`+all` (`:183-203`); `export` writes back through BuildEnv.
* **`Dpkg::BuildProfiles`** (public 1.01): see C.2. Verified quirk: invalid profile names in
  `DEB_BUILD_PROFILES` or `set_build_profiles()` are *not* dropped but replaced by `1` (the
  return value of `warning()` inside the `map`, `Dpkg/BuildProfiles.pm:100-108,126-132`):
  `set_build_profiles("foo","b<ad")` → `get_build_profiles()` = `foo,1`.
* **`Dpkg::BuildFlags`** (public 1.06): a plain store of 20 flag variables
  (`ASFLAGS…LDFLAGS` and `*_FOR_BUILD`, `Dpkg/BuildFlags.pm:80-112`) with per-flag origin and
  "maintainer modified" bits, features (area→feature→bool), builtins and option values.
  Layering order (`load_config`, `:226-233`): vendor defaults (hook `update-buildflags`,
  `:120-127`) → `$CONFDIR/buildflags.conf` → `$XDG_CONFIG_HOME/dpkg/buildflags.conf` →
  `DEB_<FLAG>_{SET,STRIP,APPEND,PREPEND}` → `DEB_<FLAG>_MAINT_{SET,STRIP,APPEND,PREPEND}`
  (`:135-215`). The config file grammar is `(append|prepend|set|strip) FLAG VALUE`
  (`:433-465`). **The actual flag policy lives in `Dpkg::Vendor::Debian`** (C.9).
* **`Dpkg::BuildTypes`** (private 0.02): bitmask `DEFAULT=1, SOURCE=2, ARCH_DEP=4,
  ARCH_INDEP=8`, `BINARY`, `FULL`, mapping from `--build=` words and from `debian/rules` target
  names (`Dpkg/BuildTypes.pm:92-122`); global state.
* **`Dpkg::BuildInfo`** (public 1.00): only `get_build_env_allowed()`, a list of 87 environment variable names (counted) allowed in `.buildinfo` (`Dpkg/BuildInfo.pm:50-195`).
* **`Dpkg::BuildAPI`** (private): `dpkg-build-api` level from `DPKG_BUILD_API` or from an
  exact `dpkg-build-api (= N)` in Build-Depends*; max level 1 (`Dpkg/BuildAPI.pm:48-123`).
  Level ≥1 changes behaviour elsewhere (e.g. `Dpkg/Shlibs.pm:117-126`).
* **`Dpkg::BuildDriver`** (private): loads `Dpkg::BuildDriver::<Name>` where `Name` is derived
  from the source stanza's **`Build-Driver` field** (`join '', map { ucfirst lc } split /-/`),
  using **string `eval qq{ require $module }`** (`Dpkg/BuildDriver.pm:91-117`). Unlike the
  other plug-in loaders the name is not restricted to identifier characters. (verified) A
  `debian/control` whose `Build-Driver` value embeds a Perl statement makes
  `Dpkg::BuildDriver->new` execute `touch` at string-eval compile time (the test path was
  mangled by the case-folding, but the command ran). Building a package executes
  `debian/rules` anyway, so the practical security impact depends on which callers construct a
  BuildDriver on untrusted trees (callers are `dpkg-buildpackage`/`dpkg-buildtree`, outside my
  scope; impact (unverified)).
* **`Dpkg::BuildDriver::DebianRules`** (private): parses `Rules-Requires-Root` (`no`,
  `binary-targets`, `dpkg/target/<t>`, `dpkg/target-subcommand`, implementation keywords
  `x/y`), errors on uppercase/`yes`/duplicates, exports `DEB_RULES_REQUIRES_ROOT` and
  `DEB_GAIN_ROOT_CMD`, defaults the root command to `fakeroot`, and runs `debian/rules <target>`
  with `system` (`Dpkg/BuildDriver/DebianRules.pm:122-298`).
* **`Dpkg::BuildTree`** (private): `clean` removes `debian/files*`, `debian/substvars*`,
  `debian/tmp`; `needs_root` instantiates `Control::Info` + `BuildDriver`
  (`Dpkg/BuildTree.pm:77-122`).

### C.9 `Dpkg::Vendor*` — vendor identification and hooks

* Origins: `$CONFDIR/origins/<name>` (or `$DPKG_ORIGINS_DIR`, `Dpkg/Vendor.pm:67-68`), parsed
  with `Dpkg::Control::HashCore` (no compression); name variants tried: lower-case,
  as-is, Ucfirst-lc, Ucfirst, with non-alphanumerics mapped to `-` (`:124-136`); cached per
  normalised key (`:98-110`). Current vendor = `DEB_VENDOR` else `default` file
  (`:146-155`).
* Vendor class: `Dpkg::Vendor::<CamelCase name>` loaded by **string `eval require`**
  (`:172-202`); name parts restricted to `[A-Za-z0-9]+` so no injection; falls back to the
  origin's `Parent:` vendor recursively, then `Default`. Shipped classes: Default, Debian,
  Ubuntu (isa Debian), Devuan (isa Debian), PureOS (isa Debian). Out-of-tree vendor classes are
  possible by design ("If you use this file as template to create a new vendor class",
  `Dpkg/Vendor/Default.pm:44-46`); the hook API is explicitly "private … no guarantee to be
  stable" (`:28-36`). Whether any exist in the wild: (unverified).
* `run_vendor_hook($id, @params)` → `$obj->run_hook(...)` (`Dpkg/Vendor.pm:210-215`); each
  class dispatches with an `if/elsif` chain on the hook name and calls `SUPER::run_hook`.

**Every hook (15, from `Dpkg/Vendor/Default.pm:177-214`) and every call site** (grep of
`run_vendor_hook(` / `->run_hook(` across `scripts/`):

| Hook | Params / return | Called from | Non-default implementations |
|---|---|---|---|
| `before-source-build` | ($srcpkg) | `dpkg-source.pl:491` | Ubuntu (Maintainer checks, `Ubuntu.pm:50-78`), PureOS (`PureOS.pm:48-62`) |
| `package-keyrings` | → keyring paths | `Dpkg/Source/Package.pm:562` | Debian (4 keyrings, `Debian.pm:52-56`), Ubuntu, Devuan, PureOS |
| `archive-keyrings` | → keyring paths | **no caller in repo** | Debian, Ubuntu, Devuan, PureOS |
| `archive-keyrings-historic` | → keyring paths | **no caller in repo** | Debian, Ubuntu, Devuan, PureOS |
| `register-custom-fields` | → list of `[op, ...]` | `Dpkg/Control/Fields.pm:44` (module load) | Ubuntu (`Launchpad-Bugs-Fixed`) |
| `builtin-build-depends` | → dep strings | `dpkg-checkbuilddeps.pl:140`, `dpkg-genbuildinfo.pl:182` | Debian: `build-essential:native` |
| `builtin-build-conflicts` | → dep strings | `dpkg-checkbuilddeps.pl:148` | Debian: () |
| `post-process-changelog-entry` | ($fields) mutate | `Dpkg/Changelog.pm:578,606` | Ubuntu (LP bugs) |
| `extend-patch-header` | (\$text, $ch_info) mutate | `Dpkg/Source/Package/V2.pm:721` | Debian (`Bug-Debian:`, `Bug-Ubuntu:`), Devuan |
| `update-buildflags` | ($flags) mutate | `Dpkg/BuildFlags.pm:126` | Debian (all flag logic), Ubuntu (overrides) |
| `builtin-system-build-paths` | → paths | `dpkg-genbuildinfo.pl:554` | Debian: `/build/` |
| `build-tainted-by` | → reasons | `dpkg-genbuildinfo.pl:318` | Debian: scans `/usr/local/{etc,include,bin,sbin,lib}` (`Debian.pm:683-710`) |
| `sanitize-environment` | () mutate `%ENV`, umask | `dpkg-buildpackage.pl:854` | Debian: umask 022, `LC_ALL`→`LANG`, `LC_COLLATE/CTYPE=C.UTF-8` (`Debian.pm:87-105`) |
| `backport-version-regex` | → qr// | `dpkg-genchanges.pl:319`, `dpkg-mergechangelogs.pl:93` | Debian `qr/~(bpo|deb)/` |
| `has-fuzzy-native-source` | → bool | `Dpkg/Source/Package/V3/Native.pm:98`, `V1.pm:412` | Debian: 1 |

PureOS also answers an undocumented `keyrings` alias (`PureOS.pm:63-64`). The
`archive-keyrings*` hooks are presumably for external consumers (download methods) —
(unverified).

**Build-flag policy** (`Dpkg/Vendor/Debian.pm:115-681`, the largest piece of policy in the
library): feature areas `future`, `abi` (lfs, time64), `qa` (bug, bug-implicit-func, canary),
`reproducible` (timeless, fixfilepath, fixdebugpath), `optimize` (lto), `sanitize`
(address, thread, leak, undefined), `hardening` (pie, stackprotector, stackprotectorstrong,
stackclash, fortify, format, relro, bindnow, branch) with defaults (`:123-167`); builtins
depending on arch (PIE-by-default arch list, `:195-226`; time32 arch list, `:236-267`);
`DEB_BUILD_OPTIONS`/`DEB_BUILD_MAINT_OPTIONS` feature parsing (`:273-282`); dozens of
arch-specific exceptions (`:286-419`); then concrete flags appended per feature
(`-g -O2`, `-D_FILE_OFFSET_BITS=64`, `-D_TIME_BITS=64`, `-Werror=implicit-function-declaration`,
`-ffile-prefix-map=…`, `-flto=auto -ffat-lto-objects`, `-specs=$DATADIR/pie-compile.specs`,
`-fstack-protector-strong`, `-fstack-clash-protection`, `-D_FORTIFY_SOURCE=2`,
`-Wl,-z,relro`, `-mbranch-protection=standard`/`-fcf-protection`, …, `:438-681`). Ubuntu
overrides LTO on some arches, `-O3` on ppc64el, fortify level 3,
`-Wl,-Bsymbolic-functions`, and explicit negative flags (`Dpkg/Vendor/Ubuntu.pm:113-197`).
This is table/arch-driven logic; tests: `t/Dpkg_BuildFlags.t` (118 assertions) and
`t/Dpkg_BuildFlags_Ubuntu.t` (19).

### C.10 `Dpkg::OpenPGP*` (all private)

* Facade `Dpkg::OpenPGP->new(backend => auto|sop|sq|gpg, needs => {api => full|verify,
  keystore => 0|1}, cmd/cmdv => …)` (`Dpkg/OpenPGP.pm:53-86`). `auto` tries backends in the
  order **sop, sq, gpg** (`:42-51,100-118`), each loaded by string `eval require` from a fixed
  map (`:88-98`), and picks the first whose commands are found in `$PATH`
  (`find_command`) and that satisfies `needs`; otherwise a do-nothing base backend.
* Command discovery per backend (`DEFAULT_CMDV` = verify-only tool, `DEFAULT_CMD` = full tool):
  SOP `sqopv rsopv sopv` / `sqop rsop gosop hop pgpainless-cli`
  (`Backend/SOP.pm:51-57`); Sequoia `sqv` / `sq` (`Backend/Sequoia.pm:45-51`); GnuPG
  `gpgv-sq gpgv` / `gpg-sq gpg`, keystore probe `gpg-agent` (`Backend/GnuPG.pm:49-59`).
* Operations: `armor`, `dearmor`, `inline_verify`, `verify`, `inline_sign`; return
  `OPENPGP_*` codes (`Dpkg/OpenPGP/ErrorCodes.pm`). The base backend implements **ASCII armor
  natively in Perl** (base64 + CRC-24, `Backend.pm:119-216`), warns on concatenated armor
  blocks. GnuPG backend detects keybox/LibrePGP keyring formats by magic bytes
  (`GnuPG.pm:126-220`). SOP calls use `timeout => 10` (`SOP.pm:69-80`).
* `Dpkg::OpenPGP::KeyHandle`: `auto` → keyfile (if path exists) or keyid
  (`^(?:0x)?[[:xdigit:]]+$`) or userid (`KeyHandle.pm:53-86`).
* Consumers: `Dpkg::Source::Package` (verify) and `dpkg-buildpackage` (sign).

### C.11 Smaller modules

* **`Dpkg::Compression`** (public 3.00): table of 4 methods — gzip (`gzip -n`, level 9),
  bzip2, lzma (`xz --format=lzma`), xz (default, level 6) (`Dpkg/Compression.pm:60-89`).
  Command lines add `-<level>`/`--fast|--best`, xz `--quiet --no-warn --no-adjust -T<n>`, gzip
  `--rsyncable` on linux/gnu/solaris (`:312-384`). **No zstd**, while C libdpkg supports
  zstd for `.deb` (`lib/dpkg/compress.c:1332-1351`; commit 2c2f7066b "libdpkg: Add zstd support for
  .deb archives").
* **`Dpkg::Compression::FileHandle`** (public 1.02): `IO::File` subclass that **ties its own
  glob** to itself (`tie *$self, $class, $self`, `:144-168`) and implements
  `TIEHANDLE/READ/READLINE/WRITE/OPEN/CLOSE/EOF/SEEK/TELL/BINMODE/FILENO` (`:203-296`) by
  lazily spawning a (de)compressor; compression is guessed from the file extension
  (`use_compression`, `:389-399`); on close it reaps the child and tolerates SIGPIPE when
  reading (`:458-474`). State lives in the glob's hash slot (`*$self->{...}`). Users can
  write `open($fh, '<', $file)` / `<$fh>` / `print $fh` on it, so tied-handle semantics are
  part of its documented use.
* **`Dpkg::Compression::Process`** (public 1.00): one child at a time via `Dpkg::IPC::spawn`
  (`:130-222`).
* **`Dpkg::Checksums`** (public 1.04): md5/sha1/sha256 via core `Digest`
  (`Dpkg/Checksums.pm:54-70,182-215`); parses `Checksums-*`/`Files` field text with filename
  regex `[0-9a-zA-Z][-+:.,=0-9a-zA-Z_~]+` and size/sum conflict detection
  (`:238-267`); exports back to fields (`:396-435`). Only sha256 is "strong".
* **`Dpkg::Substvars`** (public 2.04): variables with attribute bits USED/AUTO/AGED/OPT/DEEP/
  REQ/IMPL (`Dpkg/Substvars.pm:44-52`); file syntax `name[?!$]=value` (`:222-246`);
  substitution loop `^(.*?)\$\{([-:0-9a-z]+)\}(.*)$`/si re-scanning the whole string, so values
  are expanded recursively, with a 50-expansion cap only for values containing `$`
  (`:380-419`); warns on undefined and unused, errors on unused *required* and on obsolete
  (`Source-Version`) (`:427-448`). Built-ins `Newline`, `Space`, `Tab`, `dpkg:Version`,
  `dpkg:Upstream-Version`, plus setters for `binary:Version`, `source:Version`,
  `source:Upstream-Version`, `Arch`, `vendor:Name`, `vendor:Id`, `source:Synopsis`,
  `source:Extended-Description`, and `F:<Field>` (`:72-95,261-356`).
* **`Dpkg::IPC`** (public 1.03): `spawn(exec, from_/to_/error_to_{file,handle,string,pipe},
  env, delete_env, sig, delete_sig, chdir, timeout, wait_child, no_check)` is a hand-rolled
  `fork`/`exec { $prog[0] } @prog` (no shell) with pipe plumbing (`Dpkg/IPC.pm:212-359`);
  `wait_child` implements `timeout` with `alarm` + `local $SIG{ALRM}` and kills with TERM
  (`:400-429`). `to_string` is read fully before `error_to_string`, and `from_string` is
  written fully before reading, so a child that fills the other pipe could deadlock
  (inferred from code; not tested). Non-shell exec means no quoting issues.
* **`Dpkg::Getopt`** (private): `normalize_options` splits `-Xvalue`/`--opt=value` into two
  argv elements until a delimiter (`Dpkg/Getopt.pm:54-72`); help/version printing with colors.
* **`Dpkg::Gettext`** (public 2.01): exports `g_`, `P_`, `C_` (context via `\004` glue),
  `N_`, `textdomain`, `gettext`, `ngettext`; text domain `dpkg-dev`; `DPKG_NLS=0` or missing
  `Locale::gettext` installs identity functions (`Dpkg/Gettext.pm:122-194`). The C library
  copies this behaviour ("We mimic the behavior of the Dpkg::Gettext perl module",
  `lib/dpkg/i18n.c:42`).
* **`Dpkg::ErrorHandling`** (private 0.02): `debug/hint/info/notice/warning/error/errormsg/
  syserr/printcmd/subprocerr/usageerr`, `report_options(quiet_warnings, show_hints,
  debug_level, info_fh)`; output format `"<progname>: <type>: <msg>\n"`; `error` and
  `syserr` **`die`** (callers may `eval`), `usageerr` exits 2 (`Dpkg/ErrorHandling.pm:125-282`).
  No `$SIG{__DIE__}`/`__WARN__` handlers anywhere in the library (grep).
* **`Dpkg::Exit`** (public 2.00): LIFO stack of closures; first push installs handlers for
  INT/HUP/QUIT (exit 127 after running handlers); also run from `END`
  (`Dpkg/Exit.pm:41-110`).
* **`Dpkg::Path`** (public 1.05): `get_pkg_root_dir`/`relative_to_pkg_root`
  (`DEBIAN/` lookup), `guess_pkg_root_dir`, `check_files_are_the_same` (dev/ino),
  `canonpath` (symlink-aware `..` folding), `resolve_symlink`, `check_directory_traversal`,
  `find_command` ($PATH scan), `get_control_path` (runs **`dpkg-query --control-path`**),
  `find_build_file` (`base.<arch>`, `base.<os>`, `base`) (`Dpkg/Path.pm:72-328`).
* **`Dpkg::File`** (private): `file_slurp`, `file_dump`, `file_touch`; minor bug: the error
  message in `file_slurp` interpolates `$fh` instead of `$file` (`Dpkg/File.pm:55`).
* **`Dpkg::Lock`** (private): `file_lock($fh, $name)` via `File::FcntlLock` (fcntl
  `F_SETLKW`) or `flock` fallback with an NFS warning on non-Linux (`Dpkg/Lock.pm:45-76`).
* **`Dpkg::Conf`** (public 1.04): option files from `$CONFDIR/<file>` and
  `$XDG_CONFIG_HOME|~/.config/dpkg/<file>`; lines `name[ =]value` become `--name=value`,
  quotes stripped, short options rejected unless `allow_short` (`Dpkg/Conf.pm:107-199`);
  `filter(remove=>sub, keep=>sub)`; overloads `@{}` (`:40-42`).
* **`Dpkg::Dist::Files`** (private): `debian/files` lines
  `filename section priority [k=v ...]`, filename parsed as `pkg_ver_arch.type` (`Dpkg/Dist/Files.pm:64-122`).
* **`Dpkg::Email::Address(List)`** (private): RFC-ish `Name <user@host>` regexes with `/a`
  flag (`Dpkg/Email/Address.pm:41-76`), single-label domains warn; list = comma-separated
  (`Dpkg/Email/AddressList.pm:42-55`).
* **`Dpkg::Package`** (private): `pkg_name_is_invalid` (`[-+.0-9a-z]`, must start
  alphanumeric, `Dpkg/Package.pm:48-62`), plus a process-global "source name" with conflict
  detection.
* **`Dpkg::SysInfo`** (private): `get_num_processors` via `getconf` (`Dpkg/SysInfo.pm:41-58`).
* **`Dpkg::Color`** (private): `DPKG_COLORS=auto|always|never`, `-t` on STDOUT/STDERR,
  lazy `Term::ANSIColor` (`Dpkg/Color.pm:42-79`).
* **`Dpkg::Archive::Ar`** (private, unused by installed tools): `!<arch>` reader/writer with
  `unpack 'A16A12A6A6A8A10a2'` headers, even-size check, 15-char name limit, 2-byte padding
  (`Dpkg/Archive/Ar.pm:42-414`); no GNU long-name table support (inferred from code). Quirk:
  `write_member` passes `$member->{offs}` as the *buffer offset* argument of
  `IO::Handle::write` (`:367`).
* **`Dpkg::Interface::Storable`** (public 1.01): mixin giving `load($file|'-', compression=>1)`
  (opens through `Compression::FileHandle`, so `.gz/.xz/...` inputs are transparently
  decompressed), `save`, and `""` overload = `output()` (`Dpkg/Interface/Storable.pm:39-145`).
  Note: this is unrelated to the core `Storable` module, which is used only by
  `Dpkg::Shlibs::Symbol::clone` (`dclone`).

## D. Perl-dynamic feature inventory

Counts come from `scratchpad/dyn.pl`, which scans non-POD, non-comment lines of the 92
`Dpkg*` modules with one regex per feature (raw hits; a few categories were then checked by
hand as noted). "Public API?" says whether a non-Perl replacement would have to reproduce the
behaviour for existing Perl consumers.

| Feature | Count / locations | Public API? |
|---|---|---|
| `tie` | 2 tie classes: `Dpkg::Control::HashCore::Tie` (tied **hash**, `Dpkg/Control/HashCore/Tie.pm:65-144`) and `Dpkg::Compression::FileHandle` (tied **glob/handle**, `Dpkg/Compression/FileHandle.pm:150,203-296`) | **Yes.** `HashCore` POD documents the hash-like, case-insensitive, order-preserving field access (`Dpkg/Control/HashCore.pm:24-43`); every `Dpkg::Control`/`Index`/`Changelog` consumer uses `$ctrl->{Field}`. `Compression::FileHandle` is used as a normal filehandle (`open`, `<$fh>`, `print`). `Dpkg::Source::Archive` and `Dpkg::Source::Patch` inherit from it. |
| `use overload` | 9 modules: Version (`<=> cmp "" bool`, `Dpkg/Version.pm:71-76`), HashCore (`%{} eq`, `:60-62`), Interface::Storable (`""`, `:39-41`), Control::Info (`@{}`), Changelog (`@{}`), Changelog::Entry (`"" eq`), Index (`@{}`), Conf (`@{}`), Email::Address (`""`) | **Yes** for the 8 public ones: POD documents `$v1 <=> $v2`, `$v1 < $v2`, boolean evaluation of versions, `"$ctrl"`, `@{$ctrl_tmpl}`, `@{$chlog}`, `@{$conf}`, `"$index"`, `"$obj"` (found by grepping `=item` lines). |
| `AUTOLOAD` | 0 | – |
| string `eval` | 7: `Dpkg/Gettext.pm:127` (`use Locale::gettext`), `Lock.pm:57`, `Vendor.pm:186`, `Changelog/Parse.pm:167`, `Source/Package.pm:334`, `OpenPGP.pm:92`, `BuildDriver.pm:98` — all are `require $module` of a computed name | Plug-in loading is part of the design (custom changelog parsers are stable API per `doc/README.api`; vendor classes are a documented extension point). `BuildDriver`'s name is not sanitised (see C.8, verified code execution from `Build-Driver`). |
| `require` of computed module name | the same 6 loaders (+ Gettext's literal one): vendor classes, changelog parsers, source formats (+ `bless $self, $module` re-blessing, `Dpkg/Source/Package.pm:348`), OpenPGP backends, build drivers, FcntlLock | Changelog: stable. Vendor: semi-public (hooks private). Source formats / backends / drivers: internal, but any module installed under the right name is picked up. |
| Symbol-table manipulation | 9 glob assignments, all in `Dpkg/Gettext.pm:133-174` (`*g_ = sub {...}` etc. chosen at BEGIN); `*$self->{...}` glob-hash state in FileHandle/Archive/Patch; no `no strict 'refs'` | Internal. |
| Block `eval` (exception catching) | 6: `Changelog/Entry/Debian.pm:217` (date parse), `Control/FieldsCore.pm:1217,1238`, `IPC.pm:411` (alarm), `Source/Patch.pm:623`, `Source/Quilt.pm:185` (cleanup then rethrow) | Errors are Perl exceptions (`die` strings from `error()`); callers rely on `eval { ... }` to catch them (e.g. Quilt). A binding must map failures to `die` with the same message text if consumers parse it (unverified whether any do). |
| Closures/code refs crossing the API | 75 `sub {` occurrences in 28 files (raw). Documented callback parameters: `Dpkg::Index` `get_key_func` option (`Dpkg/Index.pm:97`), `get_keys(%criteria)` with CODE/Regexp values (`:309-342`), `sort(\&sortfunc)` (`:415-432`); `deps_iterate($deps, $callback)` (`Dpkg/Deps.pm:363-392`); `Dpkg::Conf::filter(remove=>, keep=>)` (`Dpkg/Conf.pm:201-232`); `Dpkg::Substvars::filter` (`Dpkg/Substvars.pm:462-495`); `push_exit_handler($func)` (`Dpkg/Exit.pm:47-58`); `spawn(sig => {NAME => handler})` (`Dpkg/IPC.pm:272-276`). Private: `Source::Patch` `set_header(CODE)`, `handle_binary_func`, `diff_ignore_func`; `Dist::Files::filter` | **Yes** for the listed public ones. |
| `wantarray` | 31 in 12 files (e.g. `version_check` returns `(ok,msg)` vs bool; `debarch_to_debtuple` list vs hashref; `Changelog::get_range`, `format_range` list vs `Dpkg::Index`; `HashCore::output` skips string building in void context) | **Yes** where documented (context-dependent return types are part of several public signatures). |
| `DESTROY` | 2: `Dpkg/Control/HashCore.pm:146` (break tie cycle), `Dpkg/Archive/Ar.pm:433` | Internal. |
| `END` / `BEGIN` blocks | END: `Dpkg/Exit.pm:107`, `Dpkg/Shlibs/Cppfilt.pm:138`; BEGIN: `Dpkg/Gettext.pm:124`, `Dpkg/Changelog.pm:514` | Exit handlers are public behaviour. |
| `local` on globals | `local $_` ×12, `local $/` ×5, `local $?` (`Exit.pm:108`), `local $SIG{ALRM}` (`IPC.pm:412`), `local $ENV{LC_ALL}='C'` (`Shlibs/Objdump/Object.pm:100`, `Changelog/Entry/Debian.pm:216`) | Internal. |
| `%SIG` | `__DIE__`/`__WARN__`: **0**. Other: `Exit.pm:95-103` (INT/HUP/QUIT), `IPC.pm:274,278,412` | Exit's signal behaviour is observable. |
| `state` variables (process-global caches) | 11 in 8 files: build/host arch caches (`Arch.pm:127,196`), color mode, vendor info/object caches (`Vendor.pm:101,175`), ld.so.conf visited set, ELF format cache, objdump choice | Behavioural (results are cached for the life of the process; e.g. changing `DEB_VENDOR` after first use has no effect). |
| Other global mutable state | `Dpkg::Control::FieldsCore` `%FIELDS`/`%FIELD_ORDER` mutated by `field_register`/`field_insert_*`; `Dpkg::BuildTypes` current type; `Dpkg::Package` source name; `Dpkg::Compression` default method/level/threads; `Dpkg::BuildProfiles` cached profiles; `Dpkg::BuildAPI` level; `Dpkg::Shlibs` library paths | Several are public functions (`field_register`, `compression_set_default`, `set_build_profiles`). |
| Regex features | look-around: `Version.pm:487` (split on digit/non-digit boundaries), `Arch.pm:285-320` (`(?!\#)`), `Shlibs/Cppfilt.pm:108`; `/p` + `${^MATCH}`: `Changelog/Entry/Debian.pm:422`, `Package.pm:54-55`, `Vendor/Ubuntu.pm:216-217`; `/x` multi-line regexes throughout (dependency grammar, changelog header/trailer, Email, armor); `/a` (Email); **user-supplied regexes compiled at runtime**: symbols-file `(regex)` tag (`Shlibs/Symbol.pm:197`), `diff_ignore_regex` option (`Source/Patch.pm:191`, from `dpkg-source --extend-diff-ignore` / `debian/source/options`), arch table column 3 (`Arch.pm:369,376`), vendor `backport-version-regex`. `s///e`: 0; recursive regexes: 0 | **Yes** for symbols files: the format is documented as Perl regex (`man/deb-src-symbols.pod:353`). `--extend-diff-ignore=REGEX` can come from package-supplied `debian/source/options` (`dpkg-source.pl:135-141` loads `local-options` and `options` as option files; `:191-196` appends the regex), and ends up compiled by `Source/Patch.pm:191`, so package data is interpreted as Perl regexes. |
| Prototypes | 4 `:prototype($$)` (`version_compare`, `version_compare_string`, `version_compare_part`, `deps_compare`) so they can be used directly as `sort` comparators | Part of the exported signatures; the `($$)` prototype is what lets them be passed to `sort` as named comparators (Dpkg::Deps POD: `deps_compare` "is mainly used to implement the sort() method", `Dpkg/Deps.pm:397`). In-repo code does not use them that way (grep); external use unverified. |
| Signatures | 34 subs in 12 files use `sub f($x)` | Internal style. |
| `Storable` | `Storable::dclone` once (`Shlibs/Symbol.pm:70`) | Internal. |
| Class hierarchies | `use parent` in 39 files; no `base`/`fields`; plain blessed hashes except HashCore (blessed scalar ref) and FileHandle/Archive/Patch (blessed globs); `->isa` checks 14, `->can` 5 | Subclassing is part of the public API (e.g. custom `Dpkg::Changelog` subclasses must implement documented methods; `Dpkg::Index::new_item` override pattern used by `Control::Tests`). Object internals are partly documented: `Dpkg::Deps::Simple` documents its hash **fields** `package`, `relation`, `version`, `arches`, `archqual`, `restrictions` (`Dpkg/Deps/Simple.pm` POD), and `scripts/*.pl` read them directly 12 times (grep `->{package}` etc.). |
| Taint mode | none (no `-T`, `${^TAINT}`, untaint code) | – |
| printf-style gettext | `g_` 582, `N_` 49, `P_` 2, `C_` 2 call sites; messages are `sprintf` formats passed to `error/warning(...)`; 483 msgids in `scripts/po/dpkg-dev.pot` come from modules; 11 translations | Message text is user-visible output; tests run with `LC_ALL=C`. Not an API in the POD sense. |

Not found: `AUTOLOAD`, `no strict 'refs'`, `__DIE__`/`__WARN__` hooks, `s///e`, recursive
regexes, `fields`/`base`, taint handling, XS code (the library is pure Perl).

## E. Data files, config files and environment variables consumed

### Data files shipped with dpkg (`$Dpkg::DATADIR`, default `/usr/share/dpkg`, override `DPKG_DATADIR`)

| File | Reader | Purpose |
|---|---|---|
| `data/cputable` (34 rows) | `Dpkg/Arch.pm:282-293` | Debian CPU → GNU CPU, match regex, bits, endianness |
| `data/ostable` (20 rows) | `Dpkg/Arch.pm:295-304` | Debian abi-libc-os → GNU system, match regex |
| `data/tupletable` (41 rows, `<cpu>` templates) | `Dpkg/Arch.pm:315-341` | Debian tuple ↔ Debian arch name |
| `data/abitable` (2 rows) | `Dpkg/Arch.pm:306-313` | ABI bit width overrides (`abin32`, `x32`) |
| `data/{pie,no-pie}-{compile,link}.specs` | referenced as `-specs=$DATADIR/...` in flags (`Dpkg/Vendor/Debian.pm:583-589`) | gcc spec files to toggle PIE |

The same tables are used at **configure** time to compute the C build's native arch
(via `scripts/dpkg-architecture.pl`, `m4/dpkg-arch.m4`).

### System / user configuration files

| Path | Reader | Notes |
|---|---|---|
| `$CONFDIR/origins/<vendor>` (`/etc/dpkg/origins`, override `DPKG_ORIGINS_DIR`) | `Dpkg/Vendor.pm:67-136` | deb822 `Vendor`, `Vendor-URL`, `Bugs`, `Parent` |
| `$CONFDIR/buildflags.conf`, `$XDG_CONFIG_HOME/dpkg/buildflags.conf` (or `~/.config/dpkg/`) | `Dpkg/BuildFlags.pm:135-155,433-465` | `set/append/prepend/strip FLAG VALUE` |
| `$CONFDIR/<file>`, `$XDG_CONFIG_HOME/dpkg/<file>` | `Dpkg/Conf.pm:107-152` | generic option files; file names chosen by callers (e.g. `dpkg-source.conf`, `buildpackage.conf` — callers outside scope) |
| `/etc/ld.so.conf` (+ `include` globs) | `Dpkg/Shlibs.pm:65-94,144` | library search path |
| `/usr/share/keyrings/*.pgp|*.gpg` | vendor hooks `package-keyrings`/`archive-keyrings*` (`Dpkg/Vendor/Debian.pm:52-60`, Ubuntu/Devuan/PureOS) | signature verification |
| `$GNUPGHOME` or `~/.gnupg/trustedkeys.{gpg,kbx}` | `Dpkg/OpenPGP/Backend/GnuPG.pm:77-94` | deprecated implicit keyrings (warning at `Dpkg/Source/Package.pm:553-558`) |
| `/usr/local/{etc,include,bin,sbin,lib}` (scanned for files) | `Dpkg/Vendor/Debian.pm:683-710` | `Build-Tainted-By: usr-local-has-*` |
| `$CONFDIR/shlibs.default`, `shlibs.override` | **not read by modules**; handled in `dpkg-shlibdeps.pl:71-72` | (program, other analyst) |

### Package-tree files read by modules (inputs, not config)

`debian/control` (default of `Dpkg::Control::Info->new`, `Dpkg/Control/Info.pm:76-77`),
`debian/changelog` (default of `changelog_parse`, `Dpkg/Changelog/Parse.pm:147`),
`debian/source/format` (`Dpkg::Source::Format`), `debian/source/include-binaries`
(`Dpkg/Source/BinaryFiles.pm:54`), `debian/patches/series` or `debian/patches/<vendor>.series`
and `.pc/{.version,.quilt_patches,.quilt_series,applied-patches}` (`Dpkg/Source/Quilt.pm:65-103,286-318`),
`debian/upstream/signing-key.asc` (`Dpkg/Source/Package.pm:465-469`), `debian/files`
(`Dpkg::Dist::Files`), `debian/substvars` (`Dpkg::Substvars`), `*.symbols` / `debian/*.symbols`
with `#include` (`Dpkg::Shlibs::SymbolFile`), `.dsc` files, ELF objects (header bytes),
and `DEBIAN/` dirs for package-root detection (`Dpkg/Path.pm:72-82`).

### Environment variables

Read (grep of `$ENV{...}` and `Dpkg::BuildEnv::get/has` in non-POD code):

| Variable(s) | Module(s) |
|---|---|
| `DPKG_DATADIR`, `DPKG_PROGTAR`, `DPKG_PROGPATCH`, `DPKG_PROGMAKE` | `Dpkg.pm:96-98,105` |
| `DPKG_ORIGINS_DIR`, `DEB_VENDOR` | `Dpkg/Vendor.pm:68,148-149` |
| `DPKG_COLORS` | `Dpkg/Color.pm:44` |
| `DPKG_NLS` | `Dpkg/Gettext.pm:125` |
| `DPKG_BUILD_API` | `Dpkg/BuildAPI.pm:73-74` |
| `DEB_BUILD_ARCH`, `DEB_HOST_ARCH`, `CC` | `Dpkg/Arch.pm:154,162,181,236` |
| `DEB_BUILD_PROFILES` (read and **written**) | `Dpkg/BuildProfiles.pm:99-108,133` |
| `DEB_BUILD_OPTIONS`, `DEB_BUILD_MAINT_OPTIONS` (envvar configurable) | `Dpkg/BuildOptions.pm:71-74`, `Dpkg/Vendor/Debian.pm:276-277` |
| `DEB_<FLAG>_{SET,STRIP,APPEND,PREPEND}`, `DEB_<FLAG>_MAINT_{SET,STRIP,APPEND,PREPEND}` for 20 flags | `Dpkg/BuildFlags.pm:164-215` |
| `DEB_BUILD_PATH` | `Dpkg/Vendor/Debian.pm:338` |
| `SOURCE_DATE_EPOCH` | `Dpkg/Source/Archive.pm:67` (tar mtime clamp) |
| `XDG_CONFIG_HOME`, `HOME` | `Dpkg/Conf.pm:126-127`, `Dpkg/BuildFlags.pm:150-151`, GnuPG backend |
| `GNUPGHOME` | `Dpkg/OpenPGP/Backend/GnuPG.pm`, `Sequoia.pm` |
| `LD_LIBRARY_PATH` | `Dpkg/Shlibs.pm:107-132` |
| `DEBEMAIL` | `Dpkg/Vendor/Ubuntu.pm:61-62` |
| `VISUAL`, `EDITOR`, `PATH` | `Dpkg/Source/Package/V2.pm:822-828`, `Dpkg/Path.pm:266` |
| `DEB_GAIN_ROOT_CMD` | `Dpkg/BuildTree.pm:121` |
| `GIT_DIR`, `GIT_INDEX_FILE`, `GIT_OBJECT_DIRECTORY`, `GIT_ALTERNATE_OBJECT_DIRECTORIES`, `GIT_WORK_TREE` (all deleted at module load) | `Dpkg/Source/Package/V3/Git.pm:55-59` |
| `LC_ALL`, `LANG`, `LC_*` | Debian `sanitize-environment` hook (`Dpkg/Vendor/Debian.pm:87-105`) |

Written / exported: `DEB_RULES_REQUIRES_ROOT`, `DEB_GAIN_ROOT_CMD`
(`Dpkg/BuildDriver/DebianRules.pm:174-185`), `DEB_BUILD_PROFILES`
(`Dpkg/BuildProfiles.pm:133`), any variable via `Dpkg::BuildOptions::export`/`BuildEnv::set`,
`LANG`/`LC_COLLATE`/`LC_CTYPE` and `umask 022` (Debian `sanitize-environment`), and child-only
`LC_ALL=C`, `TZ=UTC0`, `PATCH_GET=0` (Patch), deleted `TAR_OPTIONS`, `POSIXLY_CORRECT`.
`Dpkg::BuildInfo::get_build_env_allowed` lists the 87 variables recorded in `.buildinfo`.

## F. Public API surface and consumers

### Contract

`doc/README.api`: "Among the perl modules provided by libdpkg-perl, you can safely rely on
those that have $VERSION set to 1.00 (or higher) … the API is defined by what's documented in
the corresponding manual pages and nothing more … In case of API-breaking changes, the major
number in $VERSION will be increased." A second stable item: "custom changelog parsers as
Dpkg::Changelog derived modules" (since 1.18.8). `Dpkg.pm`'s POD repeats the rule
(`Dpkg.pm` "MODULES" section). POD is turned into `man3` pages named `Dpkg::X.3perl` at
install (`scripts/Makefile.am`, `install-data-local`). The `libdpkg-perl` package description
enumerates 39 public modules by name (`debian/control`, Description of libdpkg-perl).

### Size of the documented surface (measured)

* 43 modules with `$VERSION >= 1.00` (list = Dpkg, Arch, BuildFlags, BuildInfo, BuildOptions,
  BuildProfiles, Changelog, Changelog::Debian, Changelog::Entry, Changelog::Entry::Debian,
  Changelog::Parse, Checksums, Compression, Compression::FileHandle, Compression::Process,
  Conf, Control, Control::Changelog, Control::Fields, Control::FieldsCore, Control::Hash,
  Control::HashCore, Control::Info, Control::Tests, Control::Tests::Entry, Deps, Deps::AND,
  Deps::KnownFacts, Deps::Multiple, Deps::OR, Deps::Simple, Deps::Union, Exit, Gettext, IPC,
  Index, Interface::Storable, Path, Source::Format, Source::Package, Substvars, Vendor,
  Version). (`Control::FieldsCore`, `Control::HashCore`, `Changelog::Debian`,
  `Changelog::Entry::Debian` have `$VERSION >= 1.00` but are not in the package
  description list; the README rule makes them public anyway.)
* 7,228 code lines and 5,749 POD lines in those modules; 387 non-underscore subs;
  ≈564 `=item` entries in their METHODS/FUNCTIONS/VARIABLES/CONSTANTS sections (upper
  bound, includes option items). Per-module counts range from 0 `=item` (mixins such as
  `Interface::Storable`, `Control::Hash`, documented in prose) to 48 (`Dpkg`), 43
  (`Changelog`), 32 (`Index`), 29 (`BuildFlags`), 27 (`Substvars`, `Version`).
* The public surface includes **behavioural** contracts beyond function signatures:
  overloaded operators (D), the tied-hash semantics of control objects, documented object
  hash fields (`Dpkg::Deps::Simple`), callback parameters, context-sensitive returns,
  plug-in loading of `Dpkg::Changelog::<Format>`, and deb822/changelog/symbols file formats.

### API evolution (from `=head1 CHANGES` sections)

Counted over all CHANGES sections: 102 "New …" entries, 43 "Mark the module as public",
19 "Deprecated …", 15 "Remove …", 6 "Obsolete …". Versions referenced span dpkg 1.15.6 (32
entries, the initial public marking) to 1.23.8. The biggest break was dpkg **1.20.0** (16
entries; many modules bumped to 2.00 removing deprecated APIs: `Dpkg` variables,
`Dpkg::Gettext::_g`, `Changelog::dpkg/rfc822`, `Changelog::Parse::changelog_parse_debian/
_plugin`, `Deps::KnownFacts::check_package`, `Substvars::no_warn`, `Exit::@handlers`,
`Changelog::Entry::Debian::check_header/check_trailer`). Later majors: `Dpkg::Index` 3.00
(1.21.2, CTRL_TESTS keys changed), `Dpkg::Compression` 3.00 (1.23.6, removed
`compression_get_property`). Pattern: deprecate with `warnings::warnif('deprecated', …)`
(e.g. `Dpkg/Version.pm:218-222`, `Dpkg/Arch.pm:642-644`, `Dpkg/IPC.pm:345-349`), remove a
few releases later with a major bump. Recent additions continue (e.g. Substvars 2.01–2.04 for
optional/required/implicit substvars, 1.21.8–1.23.0).

### Evidence of external consumers

In-repo (git history and packaging):
* `debian/control` libdpkg-perl `Breaks: dh-exec (<< 0.31~)` "Uses the Dpkg::BuildProfiles in
  an incorrect way, which broke when making the parser more strict".
* Commit e1792f228 "debian: Add Breaks dgit << 3.13~ to libdpkg-perl": "Older dgit versions
  assumed that Dpkg::Compression::Process was available, via implicit import from
  Dpkg::Source::Package" (i.e. consumers depend even on undocumented import side effects).
* Commit 833274b1e "debian: Add Breaks to libdpkg-perl against pkg-kde-tools": "That package is using private modules with no API guarantees, and broke due to recent changes in 1.19.0" (Closes: #878919) — consumers also use *private* modules.
* Commits c8dcfa3f2 (new public `Dpkg::Source::Format` "so that other projects can reuse it",
  referencing devscripts MR 63) and e0b3b307d (new `format` option, devscripts MR 61).
* Commit cbf13f86a "Dpkg::Vendor: add the module the supported Perl API": "Lintian would like to use it when dpkg-dev is absent".
* `git log -i --grep` hit counts (any context, not only API use): lintian 63, debhelper 32,
  autopkgtest 15, devscripts 8, sbuild 4, dgit 2.
* In-repo non-`scripts/` consumers: dselect access methods (`dselect/methods/**` use
  `Dpkg`, `Dpkg::Gettext`, `Dpkg::ErrorHandling`, `Dpkg::File`, …), and
  `utils/t/update_alternatives.t`.

Local-machine evidence (Ubuntu host, `libdpkg-perl 1.23.7ubuntu1` installed), output of
`apt-cache rdepends libdpkg-perl` (offline cache), 35 unique reverse dependencies:
libsbuild-perl, lintian, libc6-dev, reprotest, pkg-perl-tools, pkg-kde-tools, pkg-js-tools,
mmdebstrap, libconfig-model-lcdproc-perl, kgb-client, git-debrebase, git-debpush, extrepo,
dh-python, dh-nodejs, dh-make-perl, dh-linktree, dh-elpa, dgit-infrastructure, dgit, debsums,
debian-cd, blhc, aptitude, apt-cacher, dupload, gobject-introspection-bin, dpkg-dev, dselect,
dpkg-repack, dh-golang, debhelper, dh-exec, devscripts, autopkgtest. (This is the local
archive index, not a census of which modules each uses.)

Within dpkg itself, the 20 `scripts/*.pl` programs (19 installed + `dpkg-ar.pl`) import modules directly (counted with grep of
`use|require` in `scripts/*.pl`): Gettext/ErrorHandling/Getopt/Dpkg 20 each, Arch 9,
Control::Info 9, Version 8, Vendor 8, Control 7, Changelog::Parse 7, Package 6,
Control::Fields 6, Deps 5, Checksums 5, BuildProfiles 5; 14 modules are imported by exactly one
program (e.g. Archive::Ar ← dpkg-ar, BuildTree ← dpkg-buildtree, SysInfo/OpenPGP/Exit/
BuildDriver ← dpkg-buildpackage, Source::Format/Control::Tests ← dpkg-source).

## G. Duplication between C libdpkg and the Perl library

| Logic | C location | Perl location | Same behaviour? (evidence) |
|---|---|---|---|
| Version parse + compare | `lib/dpkg/parsehelp.c:243-323` (`parseversion`), `lib/dpkg/version.c:66-198` (`order`, `verrevcmp`, `dpkg_version_compare/relate`) | `Dpkg/Version.pm:105-133,264-275,414-545` | Equivalent ordering rules (Perl gives digits weight value+1, C 0, but both sort them below letters); **diverges** on (verified) digit runs ≥ 2^64 (Perl `<=>` falls back to doubles; C compares digit-wise), epoch > INT_MAX (C rejects), whitespace handling and error wording. `t/Dpkg_Version.t` cross-checks both on 43 pairs. |
| Relation operators | `enum dpkg_relation`, `lib/dpkg/fields.c:590-626` | `Dpkg/Version.pm:63-69,380-399`, `Dpkg/Deps/Simple.pm:185` | Both accept deprecated `<`/`>` as `<=`/`>=` with a warning. **Diverge** on a missing operator: C accepts `pkg (1.0)` as `=` with a warning (`fields.c:619-626`, code reading); Perl `deps_parse("a (1.0)")` fails with "cannot parse dependency" and returns undef (verified). |
| deb822 stanza parsing | `lib/dpkg/parse.c:632-756` (`parse_stanza`), field handlers `lib/dpkg/fields.c` | `Dpkg/Control/HashCore.pm:197-305` | **Different parsers.** C: character-level, requires final newline, MS-DOS EOF handling, blank-in-value is a (lax) error, no `#` comments, no OpenPGP armor, case handled by field table. Perl: line-level, `#` comments anywhere, OpenPGP armor skipping, case-insensitive tie, duplicate-field options, a whitespace-only line ends the stanza (verified: `" \n"` inside Description terminates the stanza in Perl). |
| Field registry / output order | `lib/dpkg/parse.c` `fieldinfos[]` (37 `FIELD(` entries) | `Dpkg/Control/FieldsCore.pm:84-1042` (118 fields, 16 order lists) | Manually synced: `CTRL_FILE_STATUS` order is commented "Same as fieldinfos in «lib/dpkg/parse.c»" (`FieldsCore.pm:983`), extra fields noted as "not tracked by lib/dpkg/parse.c" (`:1014`). |
| Dependency parsing | `lib/dpkg/fields.c:450-700` (`f_dependency`) | `Dpkg/Deps.pm:263-361`, `Dpkg/Deps/Simple.pm:166-221` | C handles binary-package syntax only (`name[:arch] (op ver)`, `|`), no `[arch]` lists or `<profiles>` (grep of fields.c finds no `[`/`<` restriction parsing). Perl handles source syntax too. Name validation differs (next row). |
| Package-name validation | `lib/dpkg/parsehelp.c:138-160` `pkg_name_is_invalid` | `Dpkg/Package.pm:48-62` `pkg_name_is_invalid`; dependency regex `Dpkg/Deps/Simple.pm:170-174` | **Diverge**: C allows ASCII alnum incl. upper case and `-+._` ("TODO: _ is deprecated"); `Dpkg::Package` allows only `[-+.0-9a-z]`; `Deps::Simple` allows upper case but no `_`. |
| Architecture name validation | `lib/dpkg/arch.c:57-81` | `Dpkg/Arch.pm:620-630` | Same rule (alnum first, then alnum or `-`). |
| Architecture tables / tuple mapping / wildcards | none at runtime; native arch compiled in from configure (`m4/dpkg-arch.m4`, which *runs* `scripts/dpkg-architecture.pl`) | `Dpkg/Arch.pm` (+ `data/*table`) | Single implementation (Perl), consumed by C at build time. C `dpkg` has its own arch list (`lib/dpkg/arch.c:84-381`) for multiarch database purposes (foreign architectures), not tables. |
| ar archives | `lib/dpkg/ar.c` (251 lines, `dpkg_ar_*`) | `Dpkg/Archive/Ar.pm` | Duplicate, but the Perl one is only used by the uninstalled `dpkg-ar.pl`. |
| Compression | `lib/dpkg/compress.c` (1,492 lines; in-process zlib/bzip2/liblzma/libzstd with command fallback; gzip, bzip2, xz, lzma, **zstd**) | `Dpkg/Compression*.pm` (external commands only; gzip, bzip2, lzma, xz; **no zstd**) | Different capability sets; Perl always shells out. |
| tar | `lib/dpkg/tarfn.c` (619 lines; in-process tar parser for `.deb` unpack) | `Dpkg/Source/Archive.pm` (spawns GNU tar) | Not duplicated logic as such: C parses tar, Perl drives `tar(1)`. |
| Checksums | MD5 via libmd in `lib/dpkg/buffer.c:29-95` (conffile/`md5sums` hashing) | `Dpkg/Checksums.pm` (MD5, SHA-1, SHA-256 via core `Digest`) | Overlap only on MD5; field formats are Perl-only. |
| Error/warning reporting | `lib/dpkg/report.c`, `lib/dpkg/ehandle.c` | `Dpkg/ErrorHandling.pm` | Same "prog: type: msg" shape and colors; separate code. |
| Colors | `lib/dpkg/color.c:52-74` (`DPKG_COLORS`) | `Dpkg/Color.pm` | Same env var and modes. |
| i18n | `lib/dpkg/i18n.c:37-50` | `Dpkg/Gettext.pm:124-178` | C explicitly copies Perl's `DPKG_NLS` handling ("We mimic the behavior of the Dpkg::Gettext perl module", `i18n.c:42`). |
| Subprocesses | `lib/dpkg/subproc.c`, `command.c` | `Dpkg/IPC.pm` | Separate implementations. |
| Exit/cleanup handlers | `lib/dpkg/ehandle.c` (`push_cleanup`, error contexts) | `Dpkg/Exit.pm` | Different models. |
| File locking | `lib/dpkg/file.c:228-340` (`fcntl`, `F_SETLKW` or non-blocking `F_SETLK` depending on flags) | `Dpkg/Lock.pm` (`File::FcntlLock` `F_SETLKW`, `flock` fallback) | C offers wait or try-lock and reports the holder via `F_GETLK`; Perl always waits (`F_SETLKW`) or uses `flock`. |
| Config/option files | `lib/dpkg/options.c:71-180` (`dpkg_options_load_file`, `dpkg.cfg.d`) | `Dpkg/Conf.pm:163-199` | Similar grammar (`name[ =]value`, `#` comments, quotes) but separate lexers: C errors on unbalanced quotes and unknown options; Perl strips one pair of matching quotes, rejects short options unless allowed, strips leading whitespace. |
| Path helpers | `lib/dpkg/path.c` | `Dpkg/Path.pm` | Little overlap (different functions). |
| Version / control from C programs inside Perl | – | `Dpkg::Arch::get_raw_build_arch` runs `dpkg --print-architecture`; `Dpkg::Path::get_control_path` runs `dpkg-query --control-path` | Perl depends on C at run time for these. |

**Sync evidence.** Explicit cross-references found by grep: `FieldsCore.pm:983,1014`
(Perl→C), `lib/dpkg/i18n.c:42` (C→Perl). No "keep in sync" TODOs elsewhere. The parity test
in `t/Dpkg_Version.t` is the only automated C/Perl cross-check in the module tests. The
version-number divergences above show the two implementations are not identical and nothing
tests the edge cases. `git log` shows features landing in one side only (e.g. zstd in libdpkg
only: 2c2f7066b, 24d57d5ee, 6610297a6; no corresponding `Dpkg::Compression` commit).

## H. Tests

**How they run.** `make check` in `scripts/` runs `build-aux/test-runner` (TAP::Harness,
`-I scripts`, `LC_ALL=C`, `DPKG_COLORS=never`, `PATH` with the build tree's `src/`,
`scripts/`, `utils/`) with `DPKG_DATADIR`, `DPKG_ORIGINS_DIR=t/origins`, `DPKG_PROG*`
(`scripts/Makefile.am:219-226`, `build-aux/tap.am`). `Test::Dpkg` provides
`test_get_data_path`/`test_get_temp_path` (writes to `./t.tmp/<test>`), `test_needs_*`
skips, and file lists for author tests (`Test/Dpkg.pm:83-404`). The same tests are copied
into the CPAN distribution and run there with `DPKG_TEST_MODE=cpan` and a fake
`DEB_BUILD_ARCH=amd64` (`scripts/Build.PL.in`, `ACTION_test`).

**Measured run** (copy of `scripts/` in the scratchpad, `prove -I. t/Dpkg_*.t`, perl 5.40):
46 files, **12,694 assertions, all pass**; `t/Dpkg_BuildProfiles.t` reports 2 TODO tests
(6, 19) that unexpectedly pass. Wall time ≈2 s with `-j4`.

| Test file | Assertions run (skipped) | Module(s) | Fixtures | Style |
|---|---|---|---|---|
| Dpkg_Arch.t | 6,967 | Arch | – (uses `data/` tables) | generated over `get_valid_arches()` + inline wildcard expectations; data-like |
| Dpkg_Version.t | 1,755 | Version (+ C `dpkg --compare-versions`) | `__DATA__` 43 triples | **data-driven, already cross-implementation** |
| Dpkg_Control_Fields.t | 2,629 | Control::Fields(Core) | inline expected field lists per type | data-like but via Perl functions |
| Dpkg_Shlibs.t | 152 | Shlibs, Objdump(::Object), Symbol, SymbolFile | 40 files: `objdump.*` text dumps, `*.symbols`, `ld.so.conf*`, C/C++ sources + linker maps | mixed: parses objdump text fixtures; compares `is_deeply` on Perl structures and serialized symbol files |
| Dpkg_Shlibs_Cppfilt.t | 154 | Shlibs::Cppfilt (needs `c++filt`) | inline mangled/demangled pairs | data-driven (oracle = binutils) |
| Dpkg_BuildFlags.t | 118 | BuildFlags, Vendor::Debian | – | env/option → flag strings, inline |
| Dpkg_BuildFlags_Ubuntu.t | 19 | Vendor::Ubuntu | – | inline |
| Dpkg_Changelog.t / Dpkg_Changelog_Ubuntu.t (the latter `do`es the former with `DEB_VENDOR=Ubuntu`) | 102 (2) each | Changelog, Changelog::Debian, Entry::Debian | 8 changelog files (`countme`, `date-format`, `fields`, `misplaced-tz`, `regressions`, `shadow`, `stop-modeline`, `unreleased`) | **round-trip** (`"$changes" eq file`), "no parse errors", plus many inline expectations through the Perl API |
| Dpkg_Deps.t | 84 | Deps* | – | string → string / implication results, inline |
| Dpkg_Substvars.t | 64 | Substvars | 4 substvars files | inline |
| Dpkg_Checksums.t | 59 | Checksums | 3 data files | known hashes inline; implementation-agnostic |
| Dpkg_Compression.t | 48 | Compression, FileHandle, Process | – | Perl API (round-trip compress/decompress) |
| Dpkg_BuildTypes.t | 39 | BuildTypes | – | Perl API |
| Dpkg_Email_Address.t | 34 | Email::* | – | inline good/bad addresses (data-like) |
| Dpkg_Path.t | 34 | Path | – (builds temp trees) | Perl API |
| Dpkg_BuildOptions.t | 28 | BuildOptions | – | inline |
| Dpkg_Dist_Files.t | 26 | Dist::Files | 3 files lists | inline |
| Dpkg_BuildProfiles.t | 24 | BuildProfiles | – | inline formula evaluations |
| Dpkg_Control.t | 24 | Control, Control::Info, HashCore | `control-1`, 8 `bogus-*.dsc` armor cases | **input file → expected dump** + accept/reject of malformed OpenPGP armor (agnostic) |
| Dpkg_OpenPGP_KeyHandle.t | 21 | OpenPGP::KeyHandle | – | Perl API |
| Dpkg_OpenPGP.t | 62 (11) | OpenPGP + backends | 7 key/sig files | needs backends; agnostic sign/verify results |
| Dpkg_BuildAPI.t | 17 | BuildAPI | 7 `ctrl-api-*` control files | file → API level/error (agnostic) |
| Dpkg_BuildEnv.t | 14 | BuildEnv | – | Perl API |
| Dpkg_Source_Format.t | 14 | Source::Format | – | inline |
| Dpkg_Package.t | 12 | Package | – | inline |
| Dpkg_BuildTree.t | 11 | BuildTree | – | Perl API |
| Dpkg_File.t | 10 | File | 3 | Perl API |
| Dpkg_Source_Patch.t | 10 | Source::Patch | 7 `.patch` files (`c-style`, `ghost-hunk`, `index-+++`, …) | **patch file → accepted/rejected/applied** (agnostic, security cases) |
| Dpkg_Conf.t | 9 | Conf | 1 | inline |
| Dpkg_IPC.t | 8 | IPC | – | Perl API |
| Dpkg_Exit.t | 7 | Exit | – | Perl API |
| Dpkg_Vendor.t | 7 | Vendor | `t/origins/*` (5 files) | inline |
| Dpkg_Source_Package.t | 6 | Source::Package | orig tar + `.asc`/`.sig` | signature handling |
| Dpkg_Control_Tests.t | 5 | Control::Tests | 3 | accept/reject (agnostic) |
| Dpkg_Getopt.t | 4 | Getopt | – | Perl API |
| Dpkg_Source_Archive.t | 4 | Source::Archive | – | builds tarballs in temp dir |
| Dpkg_Source_Quilt.t | 2 | Source::Quilt | `parse/debian/patches/series` | inline |
| Dpkg_BuildInfo.t | 2 | BuildInfo | – | trivial |
| Dpkg_ErrorHandling.t, Dpkg_Gettext.t, Dpkg_Index.t, Dpkg_Interface_Storable.t, Dpkg_Lock.t, Dpkg_Source_Functions.t, Dpkg_SysInfo.t | 1 each | — | – | **`use_ok` only** (no behavioural tests) |

Modules with no dedicated test file: Archive::Ar, BuildDriver(::DebianRules) (exercised via
`t/dpkg_buildtree.t` and `t/dpkg_buildpackage.t`), Source::Package::V1/V2/V3::* and
Source::BinaryFiles (exercised by the program-level `t/dpkg_source.t` with 4 `.dsc`
fixtures), Vendor::Debian/Devuan/PureOS (only through BuildFlags/Changelog tests), Color,
Deps::KnownFacts beyond what Dpkg_Deps.t covers.

Program-level tests in the same directory (outside my scope, but they exercise the modules
end-to-end and are file-based oracles): `t/dpkg_mergechangelogs.t` (16 fixture files:
`ch-a`+`ch-b`+`ch-old` → `ch-merged*`), `t/dpkg_source.t`, `t/dpkg_buildpackage.t`,
`t/dpkg_buildtree.t`, `t/mk.t`.

### Usefulness as an implementation-agnostic parity oracle

* **Directly reusable as oracles (input → expected output/accept-reject, no Perl API in the
  expectation):** `Dpkg_Version.t` `__DATA__` (43 version triples; already run against C);
  changelog round-trip and parse-error-free checks on 8 changelogs; `Dpkg_Control.t`
  `control-1` dump and 8 bogus-armor `.dsc` rejections; `Dpkg_Source_Patch.t` patch fixtures;
  `Dpkg_Control_Tests` fixtures; `Dpkg_BuildAPI` fixtures; `Dpkg_Checksums` data;
  `dpkg_mergechangelogs` fixtures; `Dpkg_Shlibs_Cppfilt.t` pairs (oracle is binutils).
* **Data-like but expressed through Perl calls (extractable with modest effort):** Arch
  (6,967 assertions; derived from the tables), Control_Fields (2,629; field lists per
  type), Deps (84 string → string cases), BuildFlags (env → flags), BuildProfiles, Email,
  Substvars, Dist_Files, Shlibs (objdump text → symbols files).
* **Tied to the Perl API** (would need rewriting for any other implementation, or would remain
  as tests of a Perl binding): Compression/FileHandle, IPC, Exit, Getopt, Path, File,
  BuildEnv, BuildTypes, BuildTree, OpenPGP::KeyHandle, Index/Interface_Storable (use_ok only),
  the overload/tie behaviour checks scattered through Control/Changelog/Version tests.
* **Coverage gaps relevant to parity** (found during this study): the implication bugs in C.2
  and the version edge cases in C.3 are not covered; `Dpkg::Index` (only exercised indirectly through `Control::Tests`/`Changelog` tests), `Dpkg::Lock` and `Dpkg::ErrorHandling` have no behavioural tests of their own.

## I. Replacement assessment per module group

Difficulty = effort to reimplement the *logic* in Rust with behavioural parity
(S < ~1 kLOC pure logic; M = moderate logic or some external coupling; L = large and/or
security-sensitive orchestration; XL = large + many external tools + public API). This is my
evidence-based estimate, not a measurement.

Cross-cutting facts that apply to every group:
* **Packaging**: `libdpkg-perl` is `Architecture: all`, `Multi-Arch: foreign`
  (`debian/control`), and the CPAN `Dpkg` dist is pure Perl (`scripts/Build.PL.in`; no XS
  anywhere, D). A Rust core exposed to Perl needs a compiled binding (XS, or the non-core
  `FFI::Platypus`), which would make the package architecture-dependent. The consequences for
  cross-building and Multi-Arch are (unverified) but are a real design constraint.
* **Behavioural API**: the stable API includes overloads, a tied hash, documented object
  fields, callbacks, context-sensitive returns and Perl exceptions with specific message text
  (D, F). A binding can only preserve these if a Perl layer remains on top of the core.
* **Performance is not a driver** (B: module load ≤0.04 s; 2,215 status stanzas parsed in
  0.16 s CPU; ~40k version comparisons/s).
* **Existing duplication with C** (G) means some logic already has a second implementation
  that could be unified, but the two currently disagree on edge cases.
* **Known upstream bugs** (C.2, C.3, C.8 BuildProfiles) force a bug-compat decision for any
  parity oracle.

| Group | Code LOC | Difficulty | What blocks / complicates replacement | What makes it tractable | Category |
|---|---|---|---|---|---|
| **Version** (`Dpkg::Version`) | 252 | **S** | Public (1.05) with overloaded `<=>`/`cmp`/`""`/`bool`, `($$)`-prototyped comparators, `wantarray` in `version_check`; ~13 internal modules and 8 programs use it; Perl and C already disagree on huge numbers/epochs (G). | Pure functions; C equivalent already exists (`lib/dpkg/version.c`); `t/Dpkg_Version.t` already compares Perl vs C on 43 fixtures and 1,755 assertions. | **(1)** shared core; overload layer stays Perl. |
| **Arch** (`Dpkg::Arch` + `data/` tables) | 438 | **S** | Public (1.04), 23 exportable functions, `wantarray` dual return; executes `dpkg --print-architecture` and `$CC -dumpmachine`; table regexes are evaluated by the host regex engine (all current ones are ERE-compatible, verified by listing them); C build runs it at configure time (`m4/dpkg-arch.m4`). | Table-driven and pure; 6,967 assertions; same arch-name validation as C; a single core could serve C/Rust programs and the configure step. | **(1)** |
| **Deps** (`Dpkg::Deps*`) | 886 | **M** | All 7 modules public; object model is mutable blessed hashes whose fields are documented and read directly by programs (12 sites in `scripts/*.pl`); in-place mutators (`reduce_arch`, `simplify_deps`); callback API (`deps_iterate`); `KnownFacts` populated by callers; verified implication bugs (C.2). | Pure logic, one regex grammar, explicit truth table; 84 assertions with string-in/string-out cases; parse/output round-trips are easy to diff. | **(1)** for parse/evaluate/implication; object layer must stay Perl or emulate hash fields. |
| **Control / deb822** (Control*, Index, Email) | 2,157 (≈960 data) | **M–L** | The public API *is* Perl data structures: tied case-insensitive ordered hash behind `%{}` overload, `""`/`eq`/`@{}` overloads, consumers mutate fields freely; streaming parse from any Perl filehandle (incl. tied decompressing handles); field registry mutated at load by vendor hooks and by public `field_register`; `Index` takes closures and Regexp criteria; Perl and C parsers differ in comment, whitespace, PGP and duplicate handling (G), so "reuse the C parser" changes behaviour. | Format well defined; parser is ~110 lines; registry is pure data that could be generated for all languages; good fixtures (`control-1`, 8 bogus-armor `.dsc`, 2,629 field-table assertions). | Parser/registry: **(1)**; object model: **(2)** only makes sense in Perl. |
| **Changelog** (Changelog*) | 996 | **M** | Custom-parser plug-ins (`Dpkg::Changelog::<Format>` subclasses) are a **stable** API (`doc/README.api`) → the base class and loader must stay Perl; output runs vendor hook `post-process-changelog-entry`; collected (non-fatal) errors with exact wording; `Time::Piece` date semantics; overloads. | Debian parser is a small line state machine; byte-exact round-trip checks on 8 fixture files; `dpkg-parsechangelog` output and `dpkg-mergechangelogs` fixtures give CLI-level oracles. | Debian-format parser **(1)**; plug-in framework **(2)**. |
| **Substvars, Dist::Files, BuildOptions, BuildProfiles, BuildTypes, BuildInfo, BuildAPI** | ≈870 | **S** | Substvars/BuildOptions/BuildProfiles/BuildInfo are public; Substvars `filter` takes closures; global state (`BuildTypes`, cached profiles); BuildProfiles quirk (invalid names → `1`, C.8). | String processing with small grammars; inline tests (64 + 28 + 24 + 39 + 17 assertions). | **(1)** or keep; private ones **(3)**. |
| **BuildFlags + Vendor** | 255 + 815 | **M** | Flag policy is Perl *code* in vendor subclasses (`Dpkg/Vendor/Debian.pm:115-681`, Ubuntu overrides), dispatched through `run_hook` string names; vendor classes are discovered by computed module name and out-of-tree classes are possible by design; 15 hooks with call sites in modules **and** programs; `BuildFlags` is public 1.06 (reverse-deps such as debhelper are on this machine; which APIs they use is unverified); env-var layering with tracking (`BuildEnv`). | Output is a deterministic function of (arch, vendor, env, options) and observable via `dpkg-buildflags`; 137 assertions; Debian/Ubuntu logic is mostly tables of arches and flags. | Policy engine **(1)** feasible if vendor extension is redesigned; hook mechanism as-is **(2)**. |
| **Source packages** (Source::Package*, Format) | 2,400 | **XL** | `Dpkg::Source::Package` is public (2.04) and used by external tools (devscripts MRs, dgit Breaks in git log); per-format classes chosen by re-blessing; options hash interface; orchestrates GNU tar, GNU patch, diff, git, bzr, cp, rm, chmod, editors, OpenPGP tools; security checks (`check_directory_traversal`, signature policy); vendor hooks (`package-keyrings`, `extend-patch-header`, `has-fuzzy-native-source`, `before-source-build`). | Format classes are private; behaviour is observable at the CLI (`dpkg-source -x/-b`) with program-level fixtures; Rust could replace subprocesses with in-process tar/diff/patch (feasibility of exact output parity not evaluated). | Mostly **(3)** (private, used by `dpkg-source`/`dpkg-buildpackage`), except the public `Source::Package`/`Source::Format` facade which external Perl code uses. |
| **Source patch/archive** (Patch, Quilt, Archive, Functions, BinaryFiles) | 1,341 | **L** | Security-sensitive patch validation (C.6) and reliance on GNU patch's traversal protection; GNU tar-specific flags for reproducible tarballs; quilt `.pc` on-disk compatibility; `Patch`/`Archive` are tied filehandle subclasses. | All private; well-delineated functions; 7 patch fixtures + program tests. | **(3)** (internal to dpkg-source). |
| **Shlibs** (Shlibs*) | 1,594 | **L** | Parses GNU objdump text and pipes through `c++filt`; symbols-file `(regex)` tag is documented as a **Perl regular expression** (`man/deb-src-symbols.pod:353`), so a non-Perl engine must emulate Perl regex semantics for package-supplied patterns; many tags/arch filters; tests compare internal Perl structures. | Entirely private (0.0x); only `dpkg-shlibdeps` and `dpkg-gensymbols` use it; ELF header parsing already native; 152 + 154 assertions and 40 fixture files (objdump dumps + expected symbols files). | **(3)**, with the regex-semantics caveat. |
| **OpenPGP** (OpenPGP*) | 862 | **M** | Backend selection by probing `$PATH` for 15 different command names; behaviour depends on external tool versions; armor helpers native. | All private; consumers are `Source::Package` and `dpkg-buildpackage`; Sequoia (`sq`/`sqv`) is itself a Rust project (library use not evaluated). | **(3)** |
| **Compression / Checksums / Ar** | 892 | **S–M** | `Compression`, `Compression::FileHandle`, `Compression::Process`, `Checksums` are public; FileHandle's tied-handle semantics; Perl lacks zstd while C has it. | C `compress.c` already has in-process codecs for all formats; checksums are standard; `Archive::Ar` is unused by installed tools. | Checksums/codec core **(1)**; FileHandle wrapper **(2)**; Archive::Ar **(3)**. |
| **Infra** (Dpkg, ErrorHandling, Gettext, Color, Exit, IPC, File, Path, Lock, Getopt, SysInfo, Conf, Interface::Storable, Package) | 1,178 | **S** each | Exist to serve Perl code (exceptions-as-`die`, `%SIG`/`END` handlers, filehandle conventions, gettext text domain); 7 are public and used by dselect methods and external tools. | Trivial logic; C libdpkg already has equivalents for reporting, colors, i18n, subprocesses, locking, option files (G). | **(2)** — only meaningful as Perl; disappear for Rust programs (replaced by Rust/C equivalents). |

### Summary of the three categories

1. **Self-contained enough for a shared Rust core with a thin Perl binding**: Version, Arch
   (+tables), Deps parsing/implication, deb822 stanza parser and field registry, Debian
   changelog parser, Checksums, compression codecs, Substvars, BuildOptions/BuildProfiles,
   and possibly the build-flag policy engine. Evidence for: pure or table-driven logic, small
   size, existing fixtures, and (for Version, compression, arch-name validation) an existing
   C twin. Evidence against: their *public* Perl faces depend on overloads/ties/callbacks, so
   the "thin" binding would still need a Perl object layer, and packaging changes from
   Arch:all pure Perl to compiled code.
2. **Only make sense as Perl**: the object/operator layers (tied control hashes, overloads,
   `Interface::Storable`, `Compression::FileHandle`), the changelog plug-in framework (stable
   API), vendor-class/hook dispatch as currently designed, and the infra modules
   (ErrorHandling, Gettext, Exit, IPC, Getopt, Color, File, Lock, Conf).
3. **Could disappear if the programs using them were rewritten** (all private, few
   consumers): Shlibs* (2 programs), Source::Patch/Quilt/Archive/Functions/BinaryFiles and the
   format classes (dpkg-source), OpenPGP* (Source::Package, dpkg-buildpackage),
   BuildDriver*/BuildTree/BuildTypes/BuildAPI (dpkg-buildpackage/dpkg-buildtree),
   Dist::Files, Email::*, Package, SysInfo, Archive::Ar (no installed user). Caveat: external
   projects demonstrably use private modules too (pkg-kde-tools Breaks, dgit Breaks; F), so
   "private" does not mean "unused outside dpkg".

## Appendix 1 — Verified anomalies found during this study

| # | Location | Finding | How verified |
|---|---|---|---|
| 1 | `Dpkg/Deps.pm:127` | `$v_p >= $v_p` typo: `a (>= 5)` "implies NOT" `a (<< 10)` (returns 0 instead of undef) | perl one-liner via `deps_parse(...)->implies(...)` |
| 2 | `Dpkg/Deps/Simple.pm:287-288` | `and`/`=` precedence makes every arch list "negated"; implication on positive arch lists reversed | B::Deparse + one-liner |
| 3 | `Dpkg/Deps.pm:422-423` | `is_empty()` result discarded in `deps_compare` | B::Deparse |
| 4 | `Dpkg/Version.pm:466` vs `lib/dpkg/version.c:106-118` | Perl numeric `<=>` is exact only up to 2^64−1 (native unsigned integers); digit runs ≥ 2^64 become doubles (e.g. `1.18446744073709551616` == `1.18446744073709551615` in Perl); C compares digit-wise | `dpkg --compare-versions` (build tree) vs `version_compare` |
| 5 | `Dpkg/Version.pm:537` vs `lib/dpkg/parsehelp.c:281` | Perl accepts epoch `99999999999`; C rejects as too big | same |
| 6 | `Dpkg/BuildProfiles.pm:100-108,126-132` | Invalid profile names become the string `1` instead of being dropped | one-liner |
| 7 | `Dpkg/BuildDriver.pm:95-100` | `Build-Driver` field text reaches a string `eval`; compile-time code embedded in the field runs | harmless `touch` payload in a scratch `debian/control` |
| 8 | `Dpkg/Control.pm:277,281` | Duplicate `CTRL_COPYRIGHT_HEADER` test; license-stanza name unreachable | code reading |
| 9 | `Dpkg/File.pm:55` | Error message interpolates `$fh` instead of `$file` | code reading |
| 10 | `Dpkg/Source/Patch.pm:516-521` | Dead loop left after symlink check removal (d05856d7e) | code reading + `git log -S` |
| 11 | `debian/control` | "libfile-fcntllock-perl # Used by Dpkg::File" — actually used by `Dpkg::Lock` | grep |
| 12 | HashCore vs C parser | whitespace-only line terminates a Perl stanza (drops the rest of a multi-line field into the next parse) whereas C treats it as a blank line inside the value | one-liner with `printf 'Description: x\n line1\n \n line2\n'` |
| 13 | `Dpkg/Deps/Simple.pm:185` vs `lib/dpkg/fields.c:619-626` | Operator-less version restriction `pkg (1.0)`: C accepts as `=` (warning), Perl rejects the whole dependency field (`deps_parse` returns undef) | one-liner (Perl); code reading (C) |

## Appendix 2 — Things I could not verify

* Whether the `Dpkg` CPAN distribution is actually published/used from CPAN.
* Which specific `Dpkg::*` APIs the 35 local reverse dependencies use (only the
  dependency relation and git-log anecdotes were checked).
* Existence of out-of-tree `Dpkg::Vendor::*` classes, custom `Dpkg::Changelog::*` parsers,
  or extra `Dpkg::Source::Package::V3::*` formats in the wild.
* Callers of the `archive-keyrings`/`archive-keyrings-historic` hooks (none in this repo).
* Practical security impact of finding 7 (depends on whether any caller builds a
  `Dpkg::BuildDriver` for an untrusted tree without otherwise executing `debian/rules`).
* Potential deadlock in `Dpkg::IPC::spawn` when both `to_string` and `error_to_string`
  (or `from_string` with large output) are used — inferred from code only.
* O(n²) cost of iterating tied control hashes — inferred from `NEXTKEY`, not benchmarked.
* Consequences of turning `libdpkg-perl` from `Architecture: all` into a compiled package
  (Multi-Arch / cross-build implications).
* Whether Rust regex/ELF/tar/diff/OpenPGP crates could reproduce byte-exact behaviour; none
  were evaluated.

## Appendix 3 — Reproduction aids (in the scratchpad, not in the repo)

* `inventory.pl` → `inventory.tsv` (per-module LOC/POD/version/deps); `table.pl` → Section A table.
* `graph.pl` / `graph_lazy.pl` (dependency levels, SCCs, fan-in).
* `dyn.pl` (dynamic-feature counts).
* `code.sh FILE...` (prints non-POD lines with line numbers).
* `run/scripts` — a copy of `scripts/` used to run `prove -I. t/Dpkg_*.t` without touching the
  repo or build tree (env: `LC_ALL=C DPKG_COLORS=never DPKG_DATADIR=<repo>/data
  DPKG_ORIGINS_DIR=t/origins srcdir=. builddir=. PATH=<build>/src:<build>/scripts:...`).
