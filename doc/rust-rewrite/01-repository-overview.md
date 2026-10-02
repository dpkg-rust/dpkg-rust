# 01 — How the repository works

dpkg is Debian's package management foundation. This repository contains three things that
share a name and little else:

1. **The installer**: C programs that install, configure and remove `.deb` packages and
   keep the package database (`dpkg`, `dpkg-deb`, `dpkg-query`, …).
2. **The build toolchain**: Perl programs and modules that turn a source tree into `.deb`,
   `.dsc`, `.changes` and `.buildinfo` files (`dpkg-buildpackage`, `dpkg-source`, …).
3. **Two standalone utilities** that live here for historical reasons
   (`update-alternatives`, `start-stop-daemon`) and a legacy text front-end (`dselect`).

This fork (`dpkg-rust`) tracks upstream dpkg 1.23.x and currently differs from it only by
a GitHub Actions workflow.

## 1. The repository at a glance

| Directory | Language | Lines | What it is | Read more |
|---|---|---|---|---|
| `lib/dpkg/` | C | 25,122 | libdpkg: database, parsers, error handling, archive and compression code | [02](02-libdpkg.md) |
| `lib/compat/` | C | 3,690 | Replacements for libc functions missing on some systems | [02](02-libdpkg.md#11-portability) |
| `src/` | C, shell | 19,553 C + 1,499 shell | `dpkg`, `dpkg-deb`, `dpkg-split`, `dpkg-query`, `dpkg-divert`, `dpkg-statoverride`, `dpkg-trigger`, `dpkg-realpath`; `dpkg-maintscript-helper` | [03](03-dpkg-programs.md) |
| `utils/` | C | 6,517 | `update-alternatives`, `start-stop-daemon` | [03](03-dpkg-programs.md#7-the-smaller-programs) |
| `dselect/` | C++, Perl, shell | 7,134 C++ + about 3,400 scripts | Curses front-end and its download methods | — |
| `scripts/Dpkg/`, `scripts/Dpkg.pm` | Perl | 28,639 | The `Dpkg::*` module library (`libdpkg-perl`) | [04](04-perl-toolchain.md) |
| `scripts/*.pl` | Perl | 8,353 | The `dpkg-dev` programs | [04](04-perl-toolchain.md#3-the-programs) |
| `scripts/mk/` | make | 376 | Make fragments for `debian/rules` | [04](04-perl-toolchain.md#32-the-make-fragments) |
| `scripts/t/`, `lib/dpkg/t/`, `src/at/`, `utils/t/`, `t/`, `tests/` | Perl, C, m4, make | — | Six test suites | [05](05-build-test-release.md#2-test-suites) |
| `data/` | text | — | Architecture tables, compiler spec files | [05](05-build-test-release.md#5-architecture-coverage) |
| `man/` | POD | 19,300 | 63 manual pages and their translations | [04](04-perl-toolchain.md#6-translations-and-manual-pages) |
| `po/`, `scripts/po/`, `dselect/po/` | gettext | — | Message catalogs (44, 11 and 31 languages) | [04](04-perl-toolchain.md#6-translations-and-manual-pages) |
| `doc/` | text | — | Coding style, API policy, specifications (triggers, front-end locking, rootless builds, build drivers, protected field) | — |
| `debian/` | — | — | dpkg's own Debian packaging | [05](05-build-test-release.md#4-debian-packaging) |
| `m4/`, `build-aux/`, `configure.ac`, `Makefile.am` | m4, shell, Perl | — | Build system | [05](05-build-test-release.md#1-build-system) |

In round numbers: 61,000 lines of C and C++, 50,000 lines of Perl, 2,000 lines of shell.

## 2. Two worlds

```
                 INSTALL SIDE (C)                               BUILD SIDE (Perl)
     needed on every Debian system                    needed where packages are built

  apt / dselect / the administrator                    developer / buildd / sbuild
                 |                                                |
                 v                                                v
   +-----------------------------+              +----------------------------------+
   | dpkg                        |              | dpkg-buildpackage                |
   |  unpack, configure, remove, |              |  dpkg-source      (.dsc, tarballs)|
   |  triggers, selections       |              |  dpkg-checkbuilddeps             |
   +--+-----------+----------+---+              |  debian/rules --> debhelper -->  |
      |           |          |                  |     dpkg-shlibdeps, dpkg-gensymbols,
      v           v          v                  |     dpkg-gencontrol, dpkg-deb -b |
  dpkg-deb   dpkg-split   maintainer            |  dpkg-genbuildinfo, dpkg-genchanges
  (.deb I/O)              scripts               +----------------+-----------------+
      |                      |                                   |
      |                      +--> dpkg-divert, dpkg-trigger,     v
      |                           dpkg-statoverride,     Dpkg::* modules (libdpkg-perl)
      |                           dpkg-maintscript-helper,       ^
      |                           update-alternatives            |
      v                                                  debhelper, lintian, devscripts,
   +-----------------------------+                       sbuild, ... (external users)
   | libdpkg (static)            |
   |  database, parsers, errors  |
   +-------------+---------------+
                 v
        /var/lib/dpkg  (the database)
```

The two sides meet in only a few places:

- The Perl tools run `dpkg --print-architecture`, `dpkg-query --search`,
  `dpkg-query --control-path` and `dpkg-deb --info`.
- A package build ends with the C `dpkg-deb --build`.
- `configure` runs the Perl `dpkg-architecture` to decide which architecture is compiled
  into the C programs.
- Both sides implement version comparison, control-file parsing and dependency syntax,
  separately (see [04](04-perl-toolchain.md#25-the-same-logic-twice)).

## 3. Installing a package, end to end

What happens for `dpkg -i foo.deb` (details in [03](03-dpkg-programs.md#4-how-dpkg-installs-a-package)):

1. `dpkg` reads its configuration files, parses the command line, and takes the database
   locks.
2. It loads `status` and replays any journal entries left in `updates/`.
3. It runs `dpkg-deb --control` to extract the package's control files into a staging
   directory and parses `control`.
4. It checks Conflicts, Breaks and Pre-Depends, and runs the old package's `prerm` and the
   new package's `preinst`.
5. It runs `dpkg-deb --fsys-tarfile` and reads the tar stream from a pipe. Each file is
   written next to its destination as `<path>.dpkg-new`; the existing file is kept as
   `<path>.dpkg-tmp`.
6. When the whole archive has been read, the new files are fsynced and renamed into place.
7. It runs the old `postrm`, removes files the new version no longer ships, writes the new
   file list and control files into `info/`, and records the state **unpacked**.
8. In the configure step it resolves conffiles (keep, replace or ask), runs
   `postinst configure`, processes triggers, and records **installed**.

Every state change is journalled, and an error at any point before step 7 rolls the files
back and runs the "abort" maintainer scripts.

## 4. The package database

Everything dpkg knows is in plain files under `/var/lib/dpkg` (the "admin directory"):

```
/var/lib/dpkg/
  status              one stanza per package: state, version, dependencies, conffile hashes
  status-old          previous version of status (hard link)
  updates/NNNN        journal of status changes not yet merged into status
  available           package descriptions known from archive indexes (mainly for dselect)
  info/
    format            database layout version
    <pkg>.list        files owned by the package
    <pkg>.md5sums     checksums of those files
    <pkg>.conffiles   configuration files
    <pkg>.{preinst,postinst,prerm,postrm}   maintainer scripts
    <pkg>.triggers, .symbols, .shlibs, ...  other control files
  diversions          files redirected to another path
  statoverride        ownership and mode overrides
  triggers/           trigger interests (File, <name>), deferred activations (Unincorp), Lock
  alternatives/       state of update-alternatives
  arch                foreign architectures enabled on this system
  parts/              partial packages collected by dpkg-split
  lock, lock-frontend
```

The formats, the journal protocol and the locking rules are described in
[02](02-libdpkg.md#5-the-on-disk-database). There is no binary format and no index: the
database is loaded into memory at start-up.

## 5. Building a package, end to end

What happens for `dpkg-buildpackage` in a source tree (details in
[04](04-perl-toolchain.md#31-what-a-build-runs)):

1. Read `debian/changelog` and `debian/control`; export the architecture variables.
2. `dpkg-source --before-build` applies patches; `dpkg-checkbuilddeps` verifies the build
   dependencies against the installed packages.
3. `debian/rules clean`, then `dpkg-source -b` produces the `.dsc` and tarballs.
4. `debian/rules build` and `binary` compile the software and assemble each binary
   package. Along the way `dpkg-shlibdeps` computes library dependencies,
   `dpkg-gencontrol` writes `DEBIAN/control`, and `dpkg-deb --build` creates the `.deb`.
5. `dpkg-genbuildinfo` and `dpkg-genchanges` describe the build and the upload.
6. The `.dsc`, `.buildinfo` and `.changes` files are signed.

## 6. Concepts worth knowing before reading the code

| Term | Meaning |
|---|---|
| **deb822** | The `Field: value` stanza format used by `control`, `status`, `.dsc`, `.changes`, changelog output and most other metadata |
| **`.deb`** | An `ar` archive with three members: `debian-binary` (format version), `control.tar.*` (metadata and scripts), `data.tar.*` (the files) |
| **Maintainer scripts** | `preinst`, `postinst`, `prerm`, `postrm`: programs shipped in the package that dpkg runs at defined points, with arguments such as `configure`, `upgrade`, `abort-upgrade` |
| **Conffile** | A configuration file whose local changes dpkg preserves across upgrades |
| **Diversion** | An instruction to install a path under another name, so one package can replace another's file |
| **Statoverride** | An administrator's override of a file's owner and mode |
| **Trigger** | A way for one package to ask to be notified (through its `postinst`) when other packages change certain paths |
| **Multi-Arch** | Co-installation of the same package for several architectures; `Multi-Arch: same` packages share files by reference count |
| **Selection** | What the administrator wants for a package: install, hold, deinstall, purge |
| **Admin directory / instdir / root** | Where the database is, and where files are installed; `--root` sets both |
| **Force options** | `--force-<thing>`: 29 switches that turn specific errors into warnings |
| **Vendor** | The distribution (Debian, Ubuntu, …); selects build-flag policy and hooks on the Perl side |
| **Substvars** | `${name}` variables substituted into control fields at build time |
| **Symbols file** | The list of symbols a shared library exports, with the version each appeared in |

## 7. Interfaces other software depends on

| Interface | Stability | Used by |
|---|---|---|
| Command lines, exit codes and output of every program | de facto stable | apt, debhelper, every maintainer script, countless scripts |
| The database files under `/var/lib/dpkg` | de facto stable; read directly by other tools | apt, debsums, bash completion, backup scripts |
| Lock protocol (`doc/spec/frontend-api.txt`) | specified | apt and other front-ends |
| `--status-fd` messages and `dpkg.log` | documented in `dpkg(1)` | front-ends, log analysers |
| Maintainer-script calling convention | specified by Debian Policy | every package |
| `.deb`, `.dsc`, `.changes`, `.buildinfo` formats | documented in section 5 manual pages | the whole Debian archive infrastructure |
| `Dpkg::*` modules with `$VERSION >= 1.00` | **stable** (`doc/README.api`) | debhelper, lintian, devscripts, sbuild, dgit, … |
| `/usr/share/dpkg/*.mk` | public | packages' `debian/rules` |
| `libdpkg.a` and its headers | **volatile** (`doc/README.api`); users must define `LIBDPKG_VOLATILE_API` | the programs in this tree; external users were not surveyed |

## 8. Building and testing

```
./autogen
mkdir build-tree && cd build-tree && ../configure
make -j"$(nproc)"
make -j"$(nproc)" check                       # unit, autotest, Perl and lint suites

# The functional suite is separate and needs no root.
# It runs inside the source tree's tests/ directory, so work on a copy when building out of tree:
cp -a ../tests /tmp/dpkg-functests
make -C /tmp/dpkg-functests test DPKG_BUILDTREE="$PWD" DPKG_DATADIR="$PWD/../src"
```

On the analysis machine `make check` passed in 12 seconds (test programs already built) and
81 functional scenarios passed in 23 seconds. See
[05](05-build-test-release.md#2-test-suites) for what each suite covers.

## 9. Where to start reading

| If you want to understand … | Start at |
|---|---|
| how an installation proceeds | `src/main/archives.c` (`archivefiles`), `src/main/unpack.c` (`process_archive`) |
| how a file is unpacked | `src/main/archives.c` (`tarobject`, `tar_deferred_extract`) |
| conffile handling | `src/main/configure.c` |
| dependency checks and ordering | `src/main/packages.c`, `src/main/depcon.c` |
| error handling | `lib/dpkg/ehandle.c`, `src/main/cleanup.c` |
| the status file and its journal | `lib/dpkg/parse.c`, `lib/dpkg/dump.c`, `lib/dpkg/dbmodify.c` |
| the `.deb` format | `man/deb.pod`, `src/deb/extract.c`, `src/deb/build.c` |
| triggers | `doc/spec/triggers.txt`, `lib/dpkg/triglib.c`, `src/main/trigproc.c` |
| version ordering | `lib/dpkg/version.c`, `scripts/Dpkg/Version.pm`, `man/deb-version.pod` |
| a package build | `scripts/dpkg-buildpackage.pl` |
| source package formats | `scripts/Dpkg/Source/Package/`, `man/dpkg-source.pod` |
| build flags | `scripts/Dpkg/Vendor/Debian.pm`, `man/dpkg-buildflags.pod` |
