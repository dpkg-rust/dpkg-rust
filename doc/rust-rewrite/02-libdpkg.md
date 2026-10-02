# 02 — libdpkg, the C library

libdpkg (`lib/dpkg/`) is the support library shared by every C program in `src/` and by
`dselect`. It is where the package database, the file formats, the error model and most of
the parsing live. `lib/compat/` is a small portability shim next to it.

Line-level detail and citations for everything on this page are in
[reference/01-libdpkg.md](reference/01-libdpkg.md).

## 1. Shape and status

| Fact | Value |
|---|---|
| Size | 25,122 lines in 134 files (78 `.c`, 56 `.h`); about 15.9k lines of code without comments and blanks |
| Unit tests | `lib/dpkg/t/`: 45 files, 6,660 lines, 3,989 assertions, all passing |
| Compat shim | `lib/compat/`: 3,690 lines, of which 1,909 are imported GNU getopt/obstack; on glibc the built library is empty |
| Linkage | Static only. `configure.ac:48` uses `LT_INIT([disable-shared])` and configure refuses `--enable-shared` outside author testing |
| Consumers | All eight `src/` programs and `dselect`. `utils/` (update-alternatives, start-stop-daemon) does **not** link it |
| API status | "volatile" per `doc/README.api`; external users must define `LIBDPKG_VOLATILE_API`. Still shipped as `libdpkg-dev` (`libdpkg.a`, 52 headers, `libdpkg.pc`) |
| External libraries | libmd (MD5, mandatory unless libc has it), zlib or zlib-ng, liblzma, libzstd, libbz2 (each optional, with a command-line fallback) |

The library is single-threaded, installs no signal handlers, and contains no generated
code: every parser is written by hand.

## 2. Subsystems

| # | Subsystem | Main files | Lines | What it does |
|---|---|---|---|---|
| 1 | Error handling and program infrastructure | `ehandle.c`, `error.c`, `mustlib.c`, `report.c`, `program.c`, `debug.c`, `i18n.c`, `color.c` | 2,832 | `setjmp`/`longjmp` error contexts, the cleanup stack, `ohshit()`, allocators that never return NULL, warnings, gettext setup |
| 2 | Strings and containers | `varbuf.c`, `strvec.c`, `string.c`, `c-ctype.c`, `strhash.c`, `nfmalloc.c`, `namevalue.c` | 2,331 | Growable buffer, string vector, locale-independent ctype, FNV-1a hash, the obstack arena |
| 3 | OS helpers | `file.c`, `dir.c`, `path.c`, `fdio.c`, `atomic-file.c`, `treewalk.c`, `sysuser.c`, `subproc.c`, `command.c`, `pager.c`, `log.c` | 4,169 | fcntl locks, directory fsync, atomic file replacement, tree walker, passwd/group parsing, fork/exec/reap, `dpkg.log` and status-fd writers |
| 4 | Archives and compression | `ar.c`, `tarfn.c`, `compress.c`, `buffer.c`, `deb-version.c` | 3,267 | ar headers, a callback-driven tar extractor, the compressor table, copy loops with an MD5 tap |
| 5 | Versions and architectures | `version.c`, `arch.c` | 761 | Debian version ordering; the interned architecture list |
| 6 | In-memory package model | `dpkg-db.h`, `pkg*.c`, `depcon.c`, `pkg-spec.c`, `pkg-show.c`, `pkg-format.c` | 3,445 | `pkgset`/`pkginfo`/`pkgbin`/`dependency` structs, the package hash, dependency predicates, `--showformat` |
| 7 | deb822 parse and dump | `parse.c`, `fields.c`, `parsehelp.c`, `dump.c` | 2,907 | Stanza tokenizer, per-field parsers and writers, the `fieldinfos[]` table |
| 8 | Status and info database | `dbmodify.c`, `dbdir.c`, `db-ctrl-*.c` | 1,267 | `modstatdb_*`: open modes, locking, the `updates/` journal, the `info/` directory |
| 9 | Filesystem database | `fsys-*.c`, `pkg-files.c`, `db-fsys-*.c` | 1,864 | The global path-node hash, `.list` and `.md5sums` files, diversions, statoverrides |
| 10 | Triggers | `triglib.c`, `trigdeferred.c`, `trignote.c`, `trigname.c` | 1,574 | Interest files, the deferred-trigger queue, pending/awaited bookkeeping |
| 11 | Options | `options*.c` | 705 | The option parser and config-file loader |

Intended layering, bottom to top:

```
 presentation    pkg-format  pkg-show  options  pager  progress
 persistence     parse/fields/dump   dbmodify   db-ctrl-*   db-fsys-*   triggers
 core model      dpkg-db.h + pkg*.c            fsys-hash
 arena/config    nfmalloc    dbdir    fsys-dir    arch
 leaves          varbuf strvec string path fdio buffer version ar tarfn compress subproc ...
 foundation      ehandle + mustlib + report            <- everything depends on this
```

The layering is not strict. Field parsers write into the global package hash and trigger
lists while parsing; `modstatdb_open()` calls into the trigger code and the trigger code
calls back into `modstatdb_note()`; low-level files such as `arch.c` and `log.c` allocate
from the database arena.

## 3. Error handling: the defining design decision

Almost every function in libdpkg can exit non-locally. This is the single most important
fact for anyone planning to change or port the code.

**Raising.** `ohshit(fmt, ...)` and `ohshite(fmt, ...)` (the latter appends `strerror(errno)`)
format a message into the current *error context* and never return. Allocation wrappers
(`m_malloc`, `m_strdup`, ...) call them on failure, so do the parsers (`parse_error`), the
option parser (`badusage`) and most I/O helpers.

**Contexts.** An error context is a node on a global stack (`ehandle.c:59-82`) holding a
handler, a message printer and a list of cleanup entries. Two handler kinds exist:

- a *function* handler: the outermost one, installed by `dpkg_program_init()`, unwinds and
  calls `exit(2)`;
- a *jump* handler: `longjmp` back to a `setjmp` site. There are exactly three of those in
  the tree, all in `dpkg`: one per archive (`src/main/archives.c:1777`), one per queued
  package (`src/main/packages.c:288`) and one per deferred trigger run
  (`src/main/trigproc.c:160`). They let `dpkg` report a failure for one package and carry
  on with the next.

**Cleanups.** `push_cleanup(fn, mask, nargs, ...)` registers an undo or tidy-up action with
a flag mask. `pop_cleanup()` and `pop_error_context()` run entries newest-first if their
mask matches the reason for unwinding:

| Mask | Meaning | Typical use |
|---|---|---|
| `~0` | always | release a lock, close a directory |
| `~ehflag_normaltidy` | only when unwinding from an error | undo work: restore a file, run an abort script |
| `ehflag_bombout` | only on fatal unwinding | close a pipe |
| `ehflag_normaltidy` | only on success | the "ok" half of a fallback pair |

`push_checkpoint(mask, value)` rewrites the flag set for all *older* entries. The install
path uses it as a point of no return: past the checkpoint an error no longer rolls earlier
steps back (see [03-dpkg-programs.md](03-dpkg-programs.md#5-error-unwinding-in-the-install-path)).

**Other rules.**

- The error text is printed when the context unwinds, not when `ohshit()` is called.
- An error raised *inside* a cleanup is caught, printed as "error while cleaning up", and
  unwinding continues. After more than three nested failures the remaining cleanups are
  skipped.
- Inside a *fatal section* (`push_fatal_errors_section()`), any error prints "unrecoverable
  fatal error, aborting" and exits with status 2 without unwinding. Journal writes and
  allocation failures use this.
- `internerr()` prints `file:line:function` and calls `abort()`. No cleanups run, so no lock
  is released and no temporary file removed.
- Forked children get a fresh context, so an error before `exec` exits 2 without running
  the parent's cleanups.
- Newer APIs return errors through `struct dpkg_error` instead (tar extractor, buffer copy,
  `file_slurp`, `parseversion`, package specifiers, format parsing). That style maps
  directly onto a result type.

How widespread it is (call sites, counted with `grep`):

| Pattern | lib/dpkg | src | dselect |
|---|---|---|---|
| `ohshit(` / `ohshite(` | 226 | 315 | 30 |
| `internerr(` | 58 | 47 | 38 |
| `parse_error(` | 57 | 0 | 0 |
| `badusage(` | 13 | 100 | 1 |
| `push_cleanup*(` | 13 | 23 | 1 |
| `m_malloc`-family | 109 | 30 | 3 |

## 4. Memory and the in-memory database

**One arena.** Everything reachable from the database is allocated from a single global
obstack through `nfmalloc()`/`nfstrsave()` (`nfmalloc.c`). Objects are never freed one by
one; `nffreeall()` releases the lot, and only `pkg_hash_reset()` calls it. There are 44
arena allocation sites in the library and 34 more in `src/`.

**Two static hash tables.** Package sets live in `bins[65521]` (`pkg-hash.c`), path nodes in
`bins[262139]` (`fsys-hash.c`), both keyed by FNV-1a.

**Lookups insert.** `pkg_hash_find_set()` creates the entry if the name is unknown, so every
name mentioned in a `Depends` field, a trigger file or a diversion materialises a package
set. `fsys_hash_find_node()` does the same for paths unless told otherwise.

**The object graph** (sizes on x86-64):

| Struct | Size | Role |
|---|---|---|
| `pkgset` | 424 | One per package name; embeds the first `pkginfo` |
| `pkginfo` | 384 | One per (name, architecture) instance; state, selection, two embedded `pkgbin`s (`installed`, `available`), file list, trigger lists, opaque `clientdata` |
| `pkgbin` | 120 | Metadata of one binary package version: dependencies, architecture, version, conffiles, unknown fields |
| `dependency` / `deppossi` | 32 / 80 | A dependency clause and each `|` alternative; alternatives sit on a doubly-linked reverse-dependency list of the target package set |
| `fsys_namenode` | 80 | One per path: owning packages, diversion, statoverride, trigger interest, per-run flags and hashes |
| `trigaw`, `trigfileint` | 40 / 56 | Trigger bookkeeping nodes that are members of two lists at once |

The graph is cyclic and intrusively linked, and it is used field-by-field from outside the
library: `src/` contains 102 uses of `clientdata`, 54 of the path-node flags and 36 arena
allocations. `clientdata` even has a different struct definition in `dpkg` and in `dselect`.
In practice the struct layouts are the library's real interface.

Two more conventions matter:

- Architectures are interned, and pointer equality means architecture equality.
- Functions that receive a `pkgbin *` compare it with `&pkg->installed` to decide whether
  they are looking at installed or available data (13 such tests); the writers change what
  they emit accordingly.

## 5. The on-disk database

All paths are relative to the admin directory (default `/var/lib/dpkg`).

| Path | Format | Written by | Durability |
|---|---|---|---|
| `status` | deb822 stanzas, fixed field order | `writedb()` at checkpoints | write `status-new`, fsync, hard-link old file to `status-old`, rename, fsync directory |
| `updates/NNNN` | one stanza per file | `modstatdb_note()` | written into pre-padded `updates/tmp.i`, truncated, fsynced, renamed, directory fsynced |
| `available` | deb822 stanzas | `writedb()` at shutdown | atomic rename, **no** fsync, no backup |
| `info/<pkg>[:<arch>].list` | one absolute path per line | `write_filelist_except()` | fsync + rename + directory fsync |
| `info/<pkg>[:<arch>].md5sums` | `<32 hex>  <path>` | after unpack | fsync + rename + directory fsync |
| `info/<pkg>[:<arch>].{conffiles,preinst,postinst,prerm,postrm,triggers,...}` | package control files | `dpkg` (in `src/`) | renamed from a fsynced staging directory |
| `info/format` | a single integer (1 = multiarch layout) | info-db upgrade | fsync + rename |
| `diversions` | three lines per entry | `dpkg-divert` only (library reads) | atomic with backup |
| `statoverride` | `<user> <group> <mode> <path>` | `dpkg-statoverride` only (library reads) | atomic with backup |
| `triggers/File`, `triggers/<name>` | interest lists | trigger code | fsync + rename |
| `triggers/Unincorp` | deferred activations | trigger code, `dpkg-trigger` | rename and directory fsync, but **no fsync of the file itself** |
| `arch` | one architecture per line | `dpkg --add-architecture` | fsync + rename |
| `lock`, `lock-frontend`, `triggers/Lock` | empty lock files | — | — |

**The status journal.** Every state change of a package is first written as a single
stanza to `updates/NNNN`. After 250 journal entries, and at start-up when leftover entries
are found, the library rewrites the full `status` file and deletes the journal. On open,
leftover journal files are replayed in name order. The journal write runs inside a fatal
section: if it fails, the process exits immediately rather than trying to unwind over a
half-written journal.

**Locking.** A writer takes `lock-frontend` (unless `DPKG_FRONTEND_LOCKED` is set, which is
how a frontend such as apt that already holds it tells dpkg to skip it) and then `lock`.
Both are whole-file `fcntl(F_SETLK, F_WRLCK)` record locks, non-blocking. On contention the
library names the holder using `F_GETLK` and `/proc/<pid>/exe`. `triggers/Lock` is taken
with a *blocking* `F_SETLKW`. These are POSIX record locks, not `flock` locks, and any
replacement has to use the same primitive to interoperate.

**User and group lookup.** Statoverride entries name users and groups. The library resolves
them by parsing `/etc/passwd` and `/etc/group` itself (`sysuser.c`), honouring `DPKG_ROOT`,
rather than through NSS.

## 6. deb822 parsing and writing

`parsedb()` reads a whole file into memory and tokenizes stanzas with pointer arithmetic
(`parse_stanza`, `parse.c:631-752`). Field names are matched case-insensitively against
`fieldinfos[]` (31 current and 6 obsolete names); anything else is kept as an "arbitrary
field" in input order with its original spelling.

Four presets select strictness:

| Preset | Used for | Behaviour |
|---|---|---|
| `pdb_parse_status` | `status` | lax: many problems are warnings |
| `pdb_parse_update` | journal entries | as status, single stanza |
| `pdb_parse_available` | `available`, Packages files | lax, rejects status-only fields |
| `pdb_parse_binary` | `control` from a `.deb` | strict, single stanza |

Things a reader of the format should know:

- Output order is the fixed order of `fieldinfos[]`, then unknown fields in stored order.
- Package names are lowercased on insertion, although uppercase is accepted as input.
- `Priority` values are matched as a case-insensitive prefix of the known names and then
  checked for trailing junk: `Priority: weird` is accepted as a custom priority while
  `Priority: optionalx` is a hard error.
- `pkg_parse_verify()` carries repairs for databases written by very old dpkg versions
  (missing `Architecture`, stale selections, leftover conffiles of removed packages).
  Dropping any of them changes what the next `status` write contains.
- A final newline is mandatory, a `^Z` byte is treated as end-of-file padding, and a
  blank continuation line is only tolerated in lax mode.

There is no unit test for the parser or the writer; they are exercised only through the
program-level suites.

## 7. Archives, compression and I/O

- **ar** (`ar.c`): header helpers only. The loop that walks `.deb` members is in
  `src/deb/extract.c`.
- **tar** (`tarfn.c`): an extractor driven by a table of callbacks (`read`, `extract_file`,
  `link`, `symlink`, `mkdir`, `mknod`). It understands V7, ustar and GNU long names and
  base-256 numbers. PAX headers, sparse files and multi-volume archives are rejected.
  Symlinks are queued and created after every other entry, so an archive cannot create a
  symlink and then write through it.
- **Compression** (`compress.c`, 1,492 lines): a table with none, gzip, xz, zstd, bzip2 and
  lzma. Whether a format is handled in-process or by forking the command-line tool is
  decided at configure time. xz and zstd use multi-threaded encoders with a memory limit
  derived from `/proc/meminfo`. The filters always run in a forked child with file
  descriptors in and out, which makes this the best-isolated subsystem in the library.
- **buffer** (`buffer.c`): one generic copy loop with an optional MD5 tap; MD5 is the only
  digest the C code uses.
- **subproc/command** (`subproc.c`, `command.c`): fork, exec, reap, and translation of exit
  statuses into the familiar "subprocess failed with exit status N" messages.

## 8. Versions, architectures, dependency predicates

**Version comparison** (`version.c`) compares the epoch numerically, then the upstream
version, then the revision, each with the same routine: alternately compare a non-digit
run character by character and a digit run numerically. In non-digit runs `~` sorts before
everything including the end of the string, letters sort before all other characters, and
the end of the string sorts as zero. `parseversion()` (`parsehelp.c`) splits the epoch at
the first `:` and the revision at the last `-`.

**Architectures** (`arch.c`) are a short interned list: `all`, `any`, the native
architecture compiled in at build time, and foreign architectures recorded in the `arch`
file. The library does not read the `cputable`/`tupletable` files at all; those are a Perl
matter (see [04-perl-toolchain.md](04-perl-toolchain.md)).

**Dependency predicates** (`depcon.c`) encode the Multi-Arch rules for whether a given
package instance can satisfy a dependency alternative, and the rule that only unversioned
or `=` versioned `Provides` satisfy versioned dependencies.

## 9. Triggers

Trigger state is spread over three kinds of files (interest files, the `Unincorp` queue and
the `Triggers-Pending`/`Triggers-Awaited` fields in `status`) and over global lists in
memory. The library exposes a hook table (`trig_hooks`) that `dpkg` overrides to enqueue
deferred processing; the other programs keep the defaults. Deferred activations are
incorporated at every `modstatdb_open()`, including read-only opens. The model is specified
in `doc/spec/triggers.txt`.

## 10. Global state

About 80 file-scope variables hold program state: the error-context stack, the arena and
both hash tables, the admin and root directories, the open-database state and lock file
descriptors, the trigger lists and files, the log and status-fd lists, and a set of static
buffers returned to callers (`versiondescribe()` rotates through ten, `pkg_infodb_get_file()`
overwrites one). The full inventory is in the reference notes.

## 11. Portability

OS-specific code is concentrated in a few files: `execname.c` (six ways to map a pid to an
executable: Linux, Hurd, Solaris, macOS, AIX, FreeBSD), `fdio.c` (four preallocation
calls), `db-fsys-files.c` (Linux `FIEMAP` versus `posix_fadvise`), `meminfo.c` (Linux and
Hurd only) and `progname.c`. `lib/compat/` supplies `getopt_long`, obstack, `strnlen`,
`strndup`, `scandir`, `asprintf`, `fgetpwent` and friends where the C library lacks them,
which is what lets dpkg build on musl, the BSDs, macOS, Solaris and AIX.

## 12. What the unit tests cover, and what they do not

Well covered: string and buffer primitives, `c-ctype` (exhaustive), version parsing and
comparison (196 assertions), architecture list handling, path helpers, the command builder,
sysuser parsing, tar number decoding, the package hash and list primitives.

Not covered by any unit test: the deb822 parser and writer, all of `modstatdb` (journal,
locks, checkpoint), the info database and its upgrade, diversions and statoverride loading,
trigger state handling, compression, atomic files and locking, package specifiers, option
parsing. These are tested only indirectly through `src/at/` and `tests/`.

## 13. What this means for a Rust port

The detailed assessment is in [06-rust-rewrite-analysis.md](06-rust-rewrite-analysis.md).
In short:

- **Clean seams** (could be swapped behind the existing headers): version comparison, ar,
  `deb-version`, the string/buffer leaves, compression, most OS helpers.
- **Entangled** (cannot be replaced piecemeal without redesign): the error and cleanup
  machinery, the package model, the filesystem database, triggers, and the field layer of
  the parser. The entanglement is the `longjmp` unwinding, the arena, and the fact that
  callers reach into the structs.
- **Easy to get subtly wrong**: version edge cases, the fixed field order, legacy status
  repairs, the journal and lock protocols, exact message texts. A list is in
  [08-findings.md](08-findings.md).
