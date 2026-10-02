> **Reference notes.** Detailed working notes behind the chapters in the parent directory,
> produced by automated code analysis of dpkg 1.23.x (commit `27f661e21`) on 2026-10-02.
> Every non-trivial claim cites `path:line`; items marked "(unverified)" were inferred, not run.
> `<repo>` is the source tree, `<build>` an out-of-tree build of it, `<scratch>` a temporary
> directory that no longer exists. See [../README.md](../README.md) for how these notes were checked.

# 01 — libdpkg (`lib/dpkg/`, `lib/compat/`)

Analyst scope: the C support library shared by `dpkg`, `dpkg-deb`, `dpkg-query`, `dpkg-split`,
`dpkg-divert`, `dpkg-statoverride`, `dpkg-trigger`, `dselect` (C++), plus the compat shim library.
Checkout: `<repo>` (git HEAD `27f661e21`, upstream 1.23.x);
build tree: `<build>` (binary reports `1.23.11-1-gd0dbb (amd64)`).

## How numbers were obtained (methodology)

- Raw LOC: `wc -l` over the listed files. "Code LOC" is a rough count I made by stripping `/* */` and `//`
  comments with a small awk script and counting non-blank lines. It is approximate; no cloc/sloccount/tokei was
  installed.
- Call-site counts: `grep -rEo '<regex>'` over `*.c *.h *.cc`. **Counts include the prototype or macro
  definition in headers** (usually 1–3 per pattern). The `lib/dpkg` column excludes `lib/dpkg/t/`.
- Symbol counts: `nm -g --defined-only build/lib/dpkg/.libs/libdpkg.a` and `nm -u` over every `*.o` under
  `build/src`, `build/utils` and `build/dselect`, compared with `LC_ALL=C comm`.
- Runtime checks: I ran the already-built binaries (`build/src/dpkg`, `build/src/dpkg-query`, the
  `build/lib/dpkg/t/t-*` test programs) from my scratchpad directory, with a throwaway `--admindir` there.
  I also compiled a few probe programs **in the scratchpad** against `build/lib/dpkg/.libs/libdpkg.a`, some with
  `-fsanitize=address`. Nothing in the repo or the build tree was modified.
- "(unverified)" marks claims I inferred from reading code but did not execute or confirm.

Totals (`wc -l`): `lib/dpkg` has 78 `.c` files (19,297 lines) and 56 `.h` files (5,825 lines), so **25,122 lines
in 134 files**. Code-only is about 15.9k. `lib/dpkg/t/` has 45 files (6,660 lines). `lib/compat/` has 23 files
(3,690 lines), of which 1,909 are imported GNU code (getopt and obstack).

---

## A. Module map

### A.1 Subsystems and LOC

| # | Subsystem | Files | raw LOC | ~code LOC | Purpose |
|---|---|---|---|---|---|
| 1 | Error handling, reporting, program infra | `ehandle.[ch] error.[ch] cleanup.c mustlib.c report.[ch] progname.[ch] program.[ch] debug.[ch] macros.h dpkg.h i18n.[ch] color.[ch] perf.h test.h` (22) | 2,832 | 1,704 | setjmp/longjmp error contexts and the cleanup stack, `ohshit`/`warning`/`notice`/`hint`, "must" allocators that never return NULL, program name and setup, debug mask, gettext and locale, ANSI colours. `test.h` is the TAP test harness and `perf.h` holds benchmark timers. |
| 2 | Strings, memory, small containers | `varbuf.[ch] strvec.[ch] string.[ch] strhash.c strwide.c c-ctype.[ch] nfmalloc.c namevalue.[ch] glob.[ch] utils.c dlist.h` (17) | 2,331 | 1,460 | Growable buffer (`varbuf`, with C++ wrappers), string vector, string helpers, FNV-1a hash, display width, locale-independent ctype, the **obstack arena (`nfmalloc`)**, name↔enum tables, glob list, `fgets_checked`, intrusive doubly-linked-list macros. |
| 3 | OS, filesystem and process helpers | `file.[ch] dir.[ch] path.[ch] path-remove.c fdio.[ch] atomic-file.[ch] treewalk.[ch] sysuser.[ch] execname.[ch] meminfo.[ch] term.[ch] pager.[ch] subproc.[ch] command.[ch] progress.[ch] log.c` (30) | 4,169 | 2,383 | fcntl locks, file slurp and realpath, directory fsync, path canonicalisation, secure unlink and `rm -rf`, EINTR-safe read/write, the atomic-file protocol, directory tree walker, passwd/group parsing, pid→exe lookup, `/proc/meminfo`, terminal width, pager, fork/exec/reap, argv builder, progress line, `dpkg.log`, and the status-fd writer. |
| 4 | Archive, compression, buffered I/O | `ar.[ch] tarfn.[ch] compress.[ch] buffer.[ch] deb-version.[ch]` (10) | 3,267 | 2,331 | ar(1) header read/write helpers, a tar *extractor* driven by callbacks, the compressor table (none/gzip/xz/zstd/bzip2/lzma, in-process or external command), copy loops with an MD5 tap, and the `debian-binary` "X.Y" parser. |
| 5 | Versions and architectures | `version.[ch] arch.[ch]` (4) | 761 | 432 | Debian version comparison; interned architecture list (native/all/any/foreign) persisted in `arch`. |
| 6 | In-core package model | `dpkg-db.h pkg.[ch] pkg-hash.c pkg-list.[ch] pkg-queue.[ch] pkg-array.[ch] pkg-namevalue.c depcon.c pkg-spec.[ch] pkg-show.[ch] pkg-format.[ch]` (18) | 3,445 | 1,988 | `pkgset`/`pkginfo`/`pkgbin`/`dependency`/`deppossi` structs and the global package hash, list/queue/array helpers, Status/Want/Priority name tables, dependency satisfaction predicates, `name[:arch]` specifiers with globs, name rendering, and the `--showformat` engine. |
| 7 | deb822 parse and dump | `parse.c parsehelp.c fields.c dump.c parsedump.h` (5) | 2,907 | 2,133 | Stanza tokenizer, per-field parsers (`f_*`), writers (`w_*`), the `fieldinfos[]` table, version-string parser, and status/available writer. |
| 8 | Status DB and info DB | `dbdir.c dbmodify.c db-ctrl.h db-ctrl-access.c db-ctrl-format.c db-ctrl-upgrade.c` (6) | 1,267 | 842 | `modstatdb_*`: open modes, locking, the `updates/` journal, checkpointing; `info/` naming, the format file and the legacy→multiarch upgrade. |
| 9 | Filesystem DB | `fsys.h fsys-dir.c fsys-hash.c fsys-iter.c pkg-files.[ch] db-fsys.h db-fsys-load.c db-fsys-files.c db-fsys-digest.c db-fsys-divert.c db-fsys-override.c` (12) | 1,864 | 1,081 | Global filename-node hash, `--root` handling, `.list` loading and writing, `md5sums`, `diversions`, `statoverride`. |
| 10 | Triggers | `triglib.[ch] trigdeferred.[ch] trignote.c trigname.c` (6) | 1,574 | 1,045 | Interest files (`triggers/File`, `triggers/<name>`), the `Unincorp` deferred file and its lock, Triggers-Pending/Awaited bookkeeping, hooks overridden by `dpkg`. |
| 11 | Options and CLI | `options.[ch] options-dirs.c options-parsers.c` (4) | 705 | 472 | Long/short option parser driven by `cmdinfo` tables, config-file loader (`/etc/dpkg/<prog>.cfg[.d]`, `~/.<prog>.cfg`), `--root`/`--instdir`/`--admindir` setters. |
| | **Total** | 134 | **25,122** | ~15,871 | |

Largest files (`wc -l`): `compress.c` 1492, `parse.c` 948, `triglib.c` 880, `fields.c` 803, `tarfn.c` 619,
`dump.c` 580, `dpkg-db.h` 567, `dbmodify.c` 564, `ehandle.c` 559, `treewalk.c` 552, `varbuf.h` 525.

### A.2 Dependency layering

I computed the include graph between the groups above from `#include <dpkg/*.h>` and `"*.h"` lines in each `.c`
file, counting edges by number of includes:

```
ARCHIVE : INFRA(13) OS(7) STR(3)
FSYSDB  : INFRA(19) OS(11) PKG(11) STR(3) STATUSDB(2)
INFRA   : OS(1) STR(1)
OPTS    : INFRA(7) PKG(3) STR(3) FSYSDB(1)
OS      : INFRA(25) STR(6) ARCHIVE(1) FSYSDB(1) PKG(1)
PARSE   : INFRA(10) PKG(8) STR(7) OS(3) ARCHIVE(1) TRIG(1) VERARCH(1)
PKG     : INFRA(14) STR(4) VERARCH(3) FSYSDB(1) PARSE(1) STATUSDB(1)
STATUSDB: INFRA(10) OS(5) PKG(5) FSYSDB(3) STR(2) TRIG(1)
STR     : INFRA(12) PKG(2)
TRIG    : PKG(7) INFRA(6) STR(4) OS(3)
VERARCH : INFRA(4) STR(4) FSYSDB(1) OS(1) PKG(1)
```

The intended layering, bottom to top:

1. **Foundation (everything depends on it):** `ehandle` + `mustlib` + `report` + `macros.h`/`dpkg.h`. Every
   allocation (`m_malloc`, mustlib.c:49–53) and most failures end in `ohshit*()`, so nothing sits below the
   error-context machinery. `dpkg.h` is an umbrella header: it pulls in `progname.h`, `ehandle.h`, `report.h`,
   `string.h` and `program.h` (dpkg.h:115–119).
2. **Leaves on top of the foundation:** `varbuf`, `strvec`, `string`, `c-ctype`, `strhash`, `path`, `fdio`,
   `buffer`, `version`, `namevalue`, `deb-version`, `ar`, `tarfn`, `compress`, `subproc`, `command`, `dir`,
   `atomic-file`, `treewalk`, `term`, `meminfo`.
3. **Arena and global config:** `nfmalloc` (obstack), `dbdir` (admindir), `fsys-dir` (root dir), `arch`
   (interned architectures, allocated in the arena).
4. **Core model:** `dpkg-db.h` + `pkg*.c`, `fsys-hash`/`fsys.h`.
5. **Persistence:** `parse`/`fields`/`dump`, `dbmodify`, `db-ctrl-*`, `db-fsys-*`, `triglib`/`trigdeferred`.
6. **Presentation:** `pkg-format`, `pkg-show`, `options`, `pager`, `progress`.

Upward edges and cycles I verified per file:

- Includes that go upward: `varbuf.c` and `nfmalloc.c` include `dpkg-db.h`; `log.c` includes `dpkg-db.h`
  because it calls `nfmalloc`; `arch.c` includes `dpkg-db.h` and `fsys.h` (for `nfmalloc`, `dpkg_db_get_path`
  and `dpkg_fsys_get_dir`); `sysuser.c` includes `fsys.h` (for `dpkg_fsys_get_path`); `progname.c` includes
  `path.h`; `file.c` and `parse.c` include `buffer.h`; `pkg-format.c` includes `db-ctrl.h`, `db-fsys.h` and
  `parsedump.h`; `fields.c` includes `triglib.h`; `dbmodify.c` includes `triglib.h`.
  - `fsys.h` mixes the leaf-level "root directory" accessors (`dpkg_fsys_get_dir`, fsys.h:208–213) with the
    filename hash. That is why many low-level files include it.
- **Function-level cycle between status DB and triggers.** `modstatdb_open()` calls `trig_fixup_awaiters()` and
  `trig_incorporate()` (dbmodify.c:390–391). Trigger code calls `modstatdb_note_ifwrite()` and
  `modstatdb_note()` (triglib.c:89, 98, 126). `modstatdb_note()` in turn calls `trig_clear_awaiters()`
  (dbmodify.c:553).
- **Cycle between the parser and the model.** Field parsers mutate the global package hash and trigger lists
  while parsing: `f_name` calls `pkg_hash_find_set` (fields.c:113), `f_dependency` interns targets
  (fields.c:517), `f_trigaw` calls `pkg_spec_parse_pkg`, `trig_note_aw` and `trig_awaited_pend_enqueue`
  (fields.c:790–801). `dbmodify` calls `parsedb`/`writedb` (dbmodify.c:94, 116, 120, 382, 410, 436).
- `pkg_hash_reset()` calls `dpkg_arch_reset_list()` and then `nffreeall()` (pkg-hash.c:371–378), so the
  hash, arch and arena lifetimes are coupled.

Consumers (headers included per directory, from `grep -rhoE '#include <dpkg/...>'`):

- `src/` (41 files include some dpkg header): dpkg.h 36, i18n.h 33, dpkg-db.h 33, options.h 27, db-fsys.h 19,
  path.h 11, db-ctrl.h 11, buffer.h 11, string.h 9, triglib.h 8, subproc.h 8, debug.h 8, command.h 8,
  c-ctype.h 8, pkg.h 7, …
- `dselect/` (21 files, C++): dpkg-db.h 18, dpkg.h 15, i18n.h 13, string.h 3, c-ctype.h 3, and one each of
  subproc, options, fsys, file, debug, command.
- `utils/` (`update-alternatives`, `start-stop-daemon`) **do not link libdpkg**. They link only `libcompat`
  (utils/Makefile.am:35–36, 75–76) and include only the header-only `macros.h` and `i18n.h`.

---

## B. Core data model

### B.1 Structs (dpkg-db.h unless noted; sizes from a probe compiled on x86-64)

| struct | size | key fields | notes |
|---|---|---|---|
| `pkgset` (dpkg-db.h:253) | 424 | `next` (hash chain), `name`, **embedded** `pkg` (first `pkginfo`), `depended.{available,installed}` (heads of reverse-dependency lists), `installed_instances` | One per package name. The name is lowercased on insert (pkg-hash.c:73–77). |
| `pkginfo` (205) | 384 | `set` (back pointer), `arch_next` (other-arch instances), `want/eflag/status/priority`, `otherpriority`, `section`, `configversion`, **two embedded `pkgbin`s: `installed` and `available`**, `clientdata` (opaque per-program struct), `archives`, `trigaw.{head,tail}`, `othertrigaw_head`, `trigpend_head`, `files` + `files_list_valid` + `files_list_phys_offs`, `status_dirty` | One per (name, arch) instance. The first instance lives inside the `pkgset`; further ones are arena-allocated and chained through `arch_next` (pkg-hash.c:210–219). |
| `pkgbin` (115) | 120 | `depends`, `essential`, `is_protected`, `multiarch`, `arch` (pointer to an interned `dpkg_arch`), `pkgname_archqual` (lazily cached "name:arch"), the strings `description/maintainer/source/installedsize/origin/bugs`, `version`, `conffiles`, `arbs` (unknown fields) | Binary-package metadata. `installed` comes from status; `available` comes from the available file or the `.deb` control. |
| `dependency` (56) | 32 | `up` (owning pkginfo), `next`, `list` (alternatives), `type` | One per comma-separated clause. |
| `deppossi` (63) | 80 | `up`, `ed` (target **pkgset**), `next`, `rev_next`, `rev_prev` (doubly-linked reverse list hanging off `ed->depended.*`), `arch`, `version`, `verrel`, `arch_is_implicit`, `cyclebreak` | One per `|` alternative. |
| `conffile` (89) | 32 | `next`, `name` ("/…"), `hash`, `flags` (obsolete, remove-on-upgrade) | |
| `arbitraryfield` (74) | 24 | `next`, `name`, `value` | Unknown fields in input order, original case kept. |
| `archivedetails` (96) | 32 | `next`, `name`, `size`, `md5sum` | From the available file's Filename/Size/MD5sum. Parallel lists are filled column by column (fields.c:117–170). |
| `trigpend` (144) | 16 | `next`, `name` | Triggers-Pending. |
| `trigaw` (152) | 40 | `aw`, `pend`, `samepend_next`, `sameaw.{next,prev}` | Node on **two** lists at once: the awaiter's doubly-linked `trigaw` list and the pending package's singly-linked `othertrigaw_head`. |
| `fsys_namenode` (fsys.h:98) | 80 | `next` (hash chain), `name` (always starts with "/"), `packages` (`pkg_list` of owners), `divert`, `statoverride`, `trig_interested`, then per-run fields `flags` (`FNNF_*`), `oldhash`, `newhash`, `file_ondisk_id` | Per-run fields are reset by `fsys_hash_init()` (fsys-hash.c:44–57). |
| `fsys_namenode_list` | 16 | `next`, `namenode` | A package's file list, in forward order. |
| `fsys_diversion` (fsys.h:159) | 32 | `useinstead`, `camefrom`, `pkgset`, `next` | Paired nodes cross-linked between the contested and the redirected namenode. |
| `trigfileint` (triglib.h:56) | 56 | `pkg`, `pkgbin`, `fnn`, `options`, `samefile_next`, `inoverall.{next,prev}` | File-trigger interest; also on two lists. |
| `dpkg_version` (version.h) | 24 | `epoch` (unsigned), `version`, `revision` (const char*) | |
| `dpkg_arch` (arch.h) | 24 | `next`, `name`, `type` | Interned. **Pointer equality is architecture equality** (depcon.c:76, pkg-spec.c:177, pkg-hash.c:205). |

### B.2 Ownership and lifetime

- **Everything reachable from the hash tables is allocated in one global obstack** through `nfmalloc`,
  `nfstrsave` and `nfstrnsave` (nfmalloc.c:55–80; chunk size 8192, nfmalloc.c:39). Individual objects are never
  freed. `nffreeall()` releases the whole arena (nfmalloc.c:82–91). It is called only from `pkg_hash_reset()`
  (pkg-hash.c:374), which `modstatdb_shutdown()` calls (dbmodify.c:454).
  - Call sites: 44 `nf*` calls in `lib/dpkg` and 36 in `src/` (`grep`).
  - Things stored in the arena include the hash nodes, names, versions (`parseversion` copies into the arena,
    parsehelp.c:291), dependencies, conffiles, archive details, trigger nodes, fsys namenodes and their package
    lists (pkg-files.c:85–95), diversions, statoverride entries, `trigfileint`, foreign `dpkg_arch` nodes
    (arch.c:116–126), cached "name:arch" strings (pkg-show.c:86–99) and **status-fd list nodes** (log.c:105).
  - Some heap objects point into the arena: the native arch name (`dpkg_arch_load_native`, arch.c:315), the
    `ensure_diversions` static `db`, and `parsedb_state`.
- **Hash tables are static arrays of chain heads.**
  - Package hash: `static struct pkgset *bins[65521]` (pkg-hash.c:46–48), about 512 KiB of BSS.
  - Filename hash: `static struct fsys_namenode *bins[262139]` (fsys-hash.c:38–40), about 2 MiB of BSS.
  - Both use `str_fnv_hash` (FNV-1a, strhash.c:39–51) modulo a prime.
  - `fsys_hash_reset()` only memsets the table (fsys-hash.c:60–64); the nodes stay in the arena. Callers must
    therefore reset the fsys hash themselves after `pkg_hash_reset()` frees the arena. `src/main/main.c:983`
    does this, but in the `--command-fd` loop, which is commented out of the option table at main.c:801–803.
- **Lookup creates entries as a side effect.**
  - `pkg_hash_find_set()` always inserts if the name is missing (pkg-hash.c:68–98). Every name seen in a
    Depends field, a trigger file or a diversion package line therefore materialises a `pkgset`.
  - `pkg_hash_get_pkg()` creates per-arch instances (pkg-hash.c:177–223).
  - `fsys_hash_find_node()` inserts unless `FHFF_NO_NEW` is passed. With `FHFF_NO_COPY` it **stores the
    caller's pointer** (`name - 1`) instead of copying (fsys-hash.c:102–104). That is only safe because the
    single caller passes an arena string (`src/main/configure.c:393`, `conff->name`).
- **Aliasing and identity semantics.**
  - `pkgbin` pointers are compared by identity to know whether a function was handed the installed or the
    available data: `pkgbin == &pkg->installed` occurs 13 times in lib (pkg.c:197; dump.c:89, 153, 256, 397, 425,
    451; pkg-format.c:227, 240, 251, 262; triglib.c:377–378).
  - The `w_*` writers change behaviour on this test. For example Status, Config-Version, Conffiles and
    Triggers-* are only written for `installed`, and Filename/Size/MD5sum only for `available`.
- **Cross-grade slot reuse.** `parse_find_pkg_slot()` sometimes reuses a single installed slot for a different
  architecture, so `installed.arch` and `available.arch` can differ temporarily (parse.c:378–456).
- **Per-program extension.** `pkginfo.clientdata` is `struct perpackagestate *`, but dpkg-db.h:160–161 says
  "dselect and dpkg have different versions of this". The two definitions are in `src/main/main.h:57` and
  `dselect/pkglist.h:88`; there are 102 uses in `src/` and 87 in `dselect/`.

### B.3 How the in-memory databases are built

1. `modstatdb_open(mode)` (dbmodify.c:322–394):
   - checks privileges and access (`msdbrw_needsuperuser*` require uid and euid 0, dbmodify.c:332–335);
   - creates the admindir if it is missing;
   - takes the locks;
   - loads the arch list (`dpkg_arch_load_list`, :377);
   - `cleanupdates()` parses `status` with `pdb_parse_status`, then replays every `updates/NNNN` with
     `pdb_parse_update` (dbmodify.c:89–139);
   - optionally parses `available` (:382);
   - if writable, creates the padded `updates/tmp.i` (:386);
   - runs `trig_fixup_awaiters` and `trig_incorporate`.
2. `parsedb_parse()` (parse.c:786–857) parses each stanza into a stack-allocated temporary `pkgset tmp_set`
   (:789, :820), using `pkg_parse_field` and `fieldinfos[]`. It then validates (`pkg_parse_verify`, :164–309),
   finds or creates the real slot (`parse_find_pkg_slot`), skips older versions for available
   (:837–841), and copies with `pkg_parse_copy`. That copy does a `memcpy` of the whole `pkgbin` (:485), rewires
   reverse dependencies (`copy_dependency_links`, :902–948), and repoints `trigaw->aw` from the temporary to the
   real package (:497–505).
3. The filesystem DB is built lazily.
   - `ensure_packagefiles_available(pkg)` slurps `info/<pkg>[:arch].list`, interns every line into the fsys
     hash, and appends to `pkg->files` and to `namenode->packages` (db-fsys-files.c:74–168).
   - `ensure_allinstfiles_available()` does this for every package and prints `(Reading database ... NN%` and
     "N files and directories currently installed." to **stdout** (db-fsys-files.c:267–307).
   - On Linux it first sorts packages by the physical block of their `.list` file using the `FS_IOC_FIEMAP`
     ioctl (db-fsys-files.c:170–236). Otherwise it uses `posix_fadvise(WILLNEED)` (:237–258).
   - `ensure_diversions()` and `ensure_statoverrides()` reload their files only when the inode changed
     (`dpkg_db_reopen`, db-fsys-load.c:37–89).

---

## C. Error handling and cleanup (`ehandle.c`)

### C.1 Data structures

- `struct error_context` (ehandle.c:59–80): `next`; `handler_type` (FUNC or JUMP); a union of
  `error_handler_func*` and `jmp_buf*`; `printer{func,data}`; `cleanups` (a singly-linked stack); and `errmsg`.
  The global stack top is `static struct error_context *volatile econtext` (:82).
- `struct cleanup_entry` (:45–57) holds up to `NCALLS=2` `(mask, fn(argc, argv))` pairs, a checkpoint
  `cpmask`/`cpvalue`, and a variable-length `argv[]` of `void*`. Arguments arrive through **varargs `void*`**
  (`push_cleanup(fn, mask, nargs, ...)`, :392–401).
- An `emergency` static (:90–105) holds one preallocated cleanup entry with 20 argument slots and an 8 KiB
  error-message buffer. It is used when `malloc` fails.
- `static volatile int onerr_abort` (:41) is the "fatal section" depth.

### C.2 Lifecycle and unwinding

- **Setup.** `dpkg_program_init()` calls `push_error_context()`. That is a FUNC context with handler
  `catch_fatal_error` and printer `print_fatal_error` (program.c; ehandle.c:248–252).
- **Raising.** `ohshit(fmt, …)` formats into `econtext->errmsg` with `vasprintf`, falling back to the emergency
  buffer (:189–220), then calls `run_error_handler()` (:451–478). `ohshite` appends `": " strerror(errno)`
  (:512–537).
  1. If `onerr_abort > 0`: print "unrecoverable fatal error, aborting" and call `exit(2)` **with no unwinding**
     (:454–464).
  2. FUNC handler: `catch_fatal_error()` calls `pop_error_context(ehflag_bombout)` and then `exit(2)`
     (:113–118). It never returns.
  3. JUMP handler: `longjmp(*jump, 1)` (:473). The `setjmp` site must then call
     `pop_error_context(ehflag_bombout)` itself. Only three such sites exist in the whole tree, all per-item
     recovery loops in `dpkg`: `src/main/archives.c:1777`, `src/main/packages.c:288`,
     `src/main/trigproc.c:160`. Each pushes a JUMP context per archive or package so that one failure does not
     abort the run. `dselect` uses a FUNC handler (`dselect/main.cc:696`).
- **Unwinding.** `pop_error_context(flagset)` (:434–449) pops the context and calls `run_cleanups`
  (:260–311):
  - First it prints the error message through the context's printer. **The error text is therefore printed at
    unwind time, not at `ohshit` time** (:270–271). Printing is suppressed for `ehflag_normaltidy`.
  - Then it walks the cleanup stack in LIFO order. Each call whose `mask & flagset` is non-zero runs.
  - Checkpoint entries transform the flag set for the older entries:
    `flagset = (flagset & cpmask) | cpvalue` (:303–304; `push_checkpoint`, :322–344).
  - Each cleanup call runs under a temporary JUMP context (`recurserr`, :289–300). If the cleanup itself
    `ohshit`s, control `longjmp`s back, the recursion's own cleanups run with `bombout|recursiveerror`, and
    unwinding continues with the next entry. The printer here is "error while cleaning up".
  - More than 3 nested recursions set `flagset = 0`, which stops running cleanups, and bump `onerr_abort`
    (:273–280).
- **Conventional masks.**
  - `~ehflag_normaltidy` means "only on error", e.g. closing an fd during a parse (parse.c:575) or a stream in
    an atomic file (atomic-file.c:69).
  - `~0` means "always", e.g. unlocking a lock (file.c:343), closing a dir (db-ctrl-access.c:76), restoring
    signals (subproc.c:79).
  - `ehflag_bombout` means "only on fatal unwinding", e.g. aborting the info-DB upgrade
    (db-ctrl-upgrade.c:211).
- **`pop_cleanup(flagset)` pops whatever entry is on top** (:415–429). Callers rely on strict LIFO pairing:
  - `modstatdb_unlock()` pops twice, assuming the two lock cleanups are on top (dbmodify.c:310–319).
  - `trigdef_update_start()` pops the trigger-lock cleanup on early return (trigdeferred.c:106, 122).
  - `parsedb_close()`, `atomic_file_close()` and `subproc_signals_restore()` each pop one entry.
- **`internerr()`** (ehandle.h:97; ehandle.c:539–559) prints `prog:file:line:func(): internal error: msg` and
  calls `abort()`. **No cleanups run** (no lock release, no temporary-file removal).
- **Fatal sections.**
  - `push_fatal_errors_section()` and `pop_fatal_errors_section()` (:480–490) make any `ohshit` inside them
    exit immediately without unwinding. There are 19 uses in lib (grep).
  - They are used around allocation failure (mustlib.c:43, 53), `modstatdb_note` (dbmodify.c:519–556),
    `createimptmp`, `ensure_packagefiles_available`, `ensure_diversions`, `ensure_statoverrides`,
    `dpkg_db_reopen` and `subproc_fork` failure.
  - Quirk: pushing a new error context resets `onerr_abort = 0` (ehandle.c:232, 245).
- **Child processes.** `subproc_fork()` pushes a fresh FUNC context in the child, with printer
  "`prog (subprocess): msg`" (subproc.c:105–123). An error in the child therefore exits with code 2 without
  running the parent's cleanups, which the child inherited by `fork`.
- **Non-longjmp errors.** `struct dpkg_error {type, syserrno, str}` (error.h) is the return-code style used by
  newer APIs: `buffer_copy`, `tar_extractor`, `file_slurp`, `parseversion`, `pkg_spec_*`, `pkg_format_parse`,
  `compressor_check_params`. `dpkg_error_print()` converts it to `warning` or `ohshit` (error.c:94–111).
- **Verified quirk.** Calling `ohshit()` with no error context pushed segfaults: `error_context_errmsg_set`
  dereferences a NULL `econtext` (ehandle.c:203 → :177). The intended
  `internerr("error outside error context …")` path (ehandle.c:466–468) is unreachable. I reproduced this with a
  probe program linked against `libdpkg.a`; it exited with SIGSEGV.

### C.3 How pervasive the pattern is (grep -rEo counts; headers included)

| pattern | lib/dpkg | lib/dpkg/t | lib/compat | src | utils | dselect |
|---|---|---|---|---|---|---|
| `ohshit(` | 88 | 10 | 0 | 153 | 0 | 6 |
| `ohshite(` | 138 | 2 | 0 | 162 | 0 | 24 |
| `ohshitv(` | 2 | 0 | 0 | 1 | 0 | 0 |
| `internerr(` | 58 | 0 | 0 | 47 | 0 | 38 |
| `push_cleanup(` | 11 | 1 | 0 | 22 | 0 | 1 |
| `push_cleanup_fallback(` | 2 | 0 | 0 | 1 | 0 | 0 |
| `pop_cleanup(` | 15 | 1 | 0 | 11 | 0 | 2 |
| `push_checkpoint(` | 2 | 0 | 0 | 6 | 0 | 0 |
| `push_error_context[_func\|_jump](` | 11 | 8 | 0 | 4 | 0 | 1 |
| `pop_error_context(` | 8 | 9 | 0 | 7 | 0 | 0 |
| `push_fatal_errors_section(` | 19 | 0 | 0 | 1 | 0 | 0 |
| `setjmp(` / `longjmp(` | 2 / 1 | 3 / 1 | 0 | 3 / 0 | 1 / 1 (own impl) | 0 |
| `parse_error(` (noreturn wrapper of ohshit) | 57 | – | – | 0 | 0 | 0 |
| `badusage(` (noreturn, ohshit) | 13 | 0 | 0 | 100 | 54 (own impl) | 1 |
| `warning(` | 11 | 0 | 0 | 66 | 13 (own impl) | 0 |
| `dpkg_put_{error,errno,warn}(` | 51 | 7 | 0 | 0 | 0 | 0 |
| `m_{malloc,calloc,realloc,strdup,strndup,asprintf,vasprintf}(` | 109 | 9 | 0 | 30 | 0 | 3 |
| `nf{malloc,strsave,strnsave}(` | 44 | 0 | 0 | 34 | 0 | 0 |
| `str_fmt(` | 19 | 2 | 0 | 20 | 0 | 0 |
| `varbuf_*` identifiers | 418 | 182 | 0 | 315 | 0 | 2 |

`utils/update-alternatives.c` has its own setjmp-based error handling (`utils/update-alternatives.c:1397, 1513,
1659`) and does not use libdpkg's.

Summary: about 286 raising call sites (88+138+2+58, including ~4 header declarations) in lib (`ohshit` + `ohshite` + `ohshitv` + `internerr`) plus 57
`parse_error`. In lib, essentially **every function that allocates or does I/O can non-locally exit**: `m_malloc`
`ohshite`s on failure, and `varbuf_grow` `ohshit`s on overflow (varbuf.c:71–73).

---

## D. On-disk formats and protocols implemented in the library

### D.1 deb822 control data: status, available, update records, `.deb` control

- **Parse modes.** The `parsedbflags` presets are in dpkg-db.h:364–405:
  - `pdb_parse_status` = lax + weakclassification + allow_empty;
  - `pdb_parse_update` = status preset + single stanza;
  - `pdb_parse_available` = recordavailable + rejectstatus + lax + allow_empty;
  - `pdb_parse_binary` = recordavailable + rejectstatus + single stanza (strict).
- **Loading.** The whole file is read into memory. `mmap` is used only with `USE_MMAP`, an opt-in configure
  switch that is undefined here (config.h:416). Pipes are read through `fd_vbuf_copy` (parse.c:583–626).
- **Tokenizer** (`parse_stanza`, parse.c:631–752):
  - Pointer arithmetic over `[dataptr, endptr)` with the macros `parse_getc`/`parse_ungetc`
    (parsedump.h:67–69). Line numbers are counted for error messages.
  - Field names stop at whitespace or `:`. An empty name and a leading `-` are errors.
  - `^Z` (MSDOS EOF, `\032`) is treated like a newline when skipping blank lines, and is an error inside field
    names and values.
  - A value continues while the next line starts with whitespace.
  - A blank or whitespace-only continuation line is a *lax problem*: a warning in lax mode, otherwise an error
    (:705–710).
  - Trailing whitespace is trimmed from values (:737–740). A missing final newline is an error.
- **Field dispatch.** Field names are matched **case-insensitively** against `fieldinfos[]` (parse.c:53–96: 31
  current and 6 obsolete names). Duplicates are errors.
  - Unknown fields need a name of at least 2 characters; case-insensitive duplicates are errors (verified:
    `X-Custom` plus `x-custom` gives "duplicate value for user-defined field 'x-custom'"). They are kept in
    input order with their original case (parse.c:134–158).
  - Obsolete `Recommended`, `Optional`, `Class`, `Revision`, `Package-Revision` and `Package_Revision` are
    remapped with a warning (parse.c:87–94; fields.c:263–329).
- **Field semantics** (fields.c), with quirks verified against the built binaries:
  - Package names are lowercased by the hash; uppercase is *valid* input (`pkg_name_is_invalid` allows
    `[A-Za-z0-9]` and `-+._`, parsehelp.c:138–160). I verified that `Package: Foo` is shown as `foo`.
  - `Priority` lookup is a case-insensitive **prefix** match against the name table (namevalue.c:34–38, using
    `strncasecmp` with the table entry's length), followed by a "trailing junk" check (fields.c:86–91). Results
    I verified:
    - `Optional` becomes `optional`;
    - `weird` is kept as an "other" priority;
    - `optionalx` or `required extra` give the hard error "word in 'Priority' field: has trailing junk".
  - `Status:` is three words, want/eflag/status (fields.c:272–298), using the tables in pkg-namevalue.c.
  - `Conffiles` lines have the form ` <path> <hash> [remove-on-upgrade] [obsolete]`. They are parsed from the
    **right** with backwards pointer scans (`conffvalue_lastword`, fields.c:352–377). The path is normalised to
    a leading `/`.
  - Dependencies (fields.c:449–700):
    - Syntax is `name[:arch] [(op version)]`, joined by `|` and `,`.
    - Operators: `<<`, `<=`, `=`, `>=`, `>>`. A bare `<` or `>` warns as obsolete and means `<=`/`>=`. A
      missing operator warns and means `=`.
    - Provides allows only `=`, as a lax problem. Conflicts, Breaks, Provides and Replaces forbid `|`.
    - Conflicts, Breaks and Replaces get the implicit arch `any`; other types get the package's own arch,
      filled in later at parse.c:216–219.
  - `Triggers-Pending` and `Triggers-Awaited` are rejected in available or binary contexts. While parsing they
    write into the global trigger structures (fields.c:746–803).
  - Archive fields (`Filename`, `Size`, `MD5sum`) are space-separated parallel lists, filled column by column
    into one `archivedetails` chain (fields.c:116–170).
- **Validation and fix-ups** (`pkg_parse_verify`, parse.c:164–309) include legacy repairs:
  - Description and Maintainer are filled with empty strings and a warning if missing.
  - A missing Architecture warns, except for not-installed, half-installed and config-files packages "because
    dpkg < 1.10.19 didn't preserve it".
  - Conffiles of not-installed packages are dropped with a warning.
  - Leftover deinstall/purge selections of not-installed packages are reset.
  - Un-arch-qualified `install` selections are reset.
  - Origin and Bugs are blanked for dead packages (fix for dpkg < 1.13.10).
  - Status and trigger consistency is checked, and `Config-Version` against Status.
  - Multi-Arch co-installability is checked (parse.c:338–376, 425–434).
- **Dumping** (dump.c):
  - **Fields are written in the fixed order of `fieldinfos[]`**, followed by unknown fields in their stored
    order (`varbuf_stanza`, dump.c:484–497). I checked that `/var/lib/dpkg/status` on this host has this order,
    with `Homepage`/`Original-Maintainer` after `Description`.
  - The status and available writers sort packages by name, then by non-ambiguous arch
    (`pkg_sorter_by_nonambig_name_arch`, pkg-show.c:368–399). Stanzas that are not informative are skipped
    (pkg.c:193–217).
  - A static 8 KiB stdio buffer is used (dump.c:517).
  - `w_status` and `w_trig*` `internerr()` on inconsistent trigger state (dump.c:252–315, 418–469). One message
    has the typo "stata" (dump.c:292).

### D.2 Status journal (`updates/`) and `modstatdb` (dbmodify.c)

- **Modes** (`enum modstatdb_rw`, dpkg-db.h:277–292): `readonly`, `needsuperuserlockonly`, `writeifposs`,
  `write`, `needsuperuser`, plus the available flags `available_readonly`/`available_write` (bits 8–9). Write
  modes downgrade to `readonly` on `EACCES` unless `>= msdbrw_write` is required (dbmodify.c:355–362).
- **Locking** (dbmodify.c:232–319; file.c:227–344):
  1. `lock-frontend` (skipped if the environment variable `DPKG_FRONTEND_LOCKED` is set) and then `lock`.
  2. Both are opened `O_RDWR|O_CREAT|O_TRUNC, 0660`.
  3. Both get a whole-file `fcntl(F_SETLK, F_WRLCK)`, non-blocking (`FILE_LOCK_NOWAIT`).
  4. On contention the holder's pid comes from `F_GETLK` and its executable from `/proc/<pid>/exe` (execname.c),
     giving the message "… was locked by <exe> process with pid N" plus a wiki note (file.c:325–341).
  5. The unlock is registered as a cleanup with mask `~0` (file.c:343).
  - `modstatdb_is_locked()` uses `F_GETLK` and treats a lock held by its own pid as unlocked (file.c:271–284).
  - **This fcntl protocol is shared with apt and other frontends** (inferred from the `DPKG_FRONTEND_LOCKED`
    design; not verified against apt source).
- **Recording a change** (`modstatdb_note`, dbmodify.c:514–557):
  1. Logs `status <state> <pkg:arch> <version>` to `dpkg.log` and `status: <pkg>: <state>` to every status-fd,
     both only if `status_dirty`.
  2. Then `modstatdb_note_core` (:459–505):
     a. writes the single stanza into `updates/tmp.i`;
     b. `fflush`, `ftruncate` to the written length, `fsync`, `fclose`;
     c. `rename` to `updates/%04d`;
     d. `fsync` the `updates/` directory.
  - `tmp.i` is pre-written with 512 `#padded\n` lines (4 KiB) inside a fatal section (`createimptmp`, :141–165).
    The intent is presumably to preallocate the blocks so the critical write cannot hit ENOSPC (unverified;
    there is no comment).
  - After `MAXUPDATES=250` records the journal is checkpointed (:499–502; dpkg.h:95).
- **Checkpoint** (`modstatdb_checkpoint`, :402–430):
  1. `writedb(status, wdb_must_sync)`;
  2. unlink each `updates/NNNN`;
  3. `fsync` the `updates/` directory.
- **Replay on open** (`cleanupdates`, :88–139):
  - Scans `updates/` with `scandir` and `alphasort`, accepting only all-digit names.
  - It is an error if any name is longer than 10 characters or the names have different lengths
    (`update_file_filter`, :64–86).
  - Each file is parsed as an update.
  - If the DB is writable, it then writes the full status (synced), unlinks the update files, and fsyncs the
    directory.
- **Full writes** (`writedb`, dump.c:557–580, through `atomic_file`):
  1. write `status-new`;
  2. if `wdb_must_sync`: `ferror`, `fflush`, `fsync` (atomic-file.c:72–81);
  3. `fclose`;
  4. **hard-link** the current `status` to `status-old` (backup, atomic-file.c:92–105; skipped for available);
  5. `rename(status-new, status)`;
  6. `fsync` the parent directory (`dir_sync_path_parent`).
  - `available` is written at shutdown without sync and without backup (dbmodify.c:435–436; dump.c:563–564).

### D.3 Info DB (`info/`, db-ctrl-*)

- **File naming.** `info/<pkg>.<type>`, or `info/<pkg>:<arch>.<type>` for `Multi-Arch: same` packages when the DB
  format is multiarch (db-ctrl-format.c:128–148).
- **Hazard.** `pkg_infodb_get_file()` returns a pointer into a *static* varbuf (:131). Every call overwrites the
  previous result.
- **Format version** (`info/format`):
  - Missing file means 0 (legacy). The current version is 1 (multiarch), parsed with `fscanf("%u")`
    (db-ctrl-format.c:42–62).
  - If `info/format-new` exists, an upgrade was interrupted: the format is taken as format+1 and `db_upgrading`
    is set (:79–82).
  - Bogus or too-new values are fatal.
- **Upgrade 0 → 1** (db-ctrl-upgrade.c:202–247):
  1. Hard-link every `pkg.<type>` of an installed M-A: same package to `pkg:arch.<type>`, recording a rename list.
  2. Write `format-new` containing `1`, with sync, plus a directory sync.
  3. Unlink the old names.
  4. Commit (rename) the format file and sync the directory.
  - An error-only (bombout) cleanup re-links the old names and removes the new ones (:152–173).
  - The upgrade only runs when the DB was opened for writing (:241–242).
- **Iteration.** `pkg_infodb_foreach()` scans the directory and filters files by `<pkgname>.` prefix
  (db-ctrl-access.c:46–117).

### D.4 Filesystem DB files

- **`.list`.** One absolute path per line; the trailing `/` is stripped on read; the final newline is mandatory;
  empty names are errors (db-fsys-files.c:74–113). Written atomically with sync, no backup, followed by an
  `info/` directory sync (`write_filelist_except`, :320–349).
- **`.md5sums`.** Lines have the form `<32 hex>  <path without leading slash>` with exactly two spaces; the hash
  length is assumed to be 32 (db-fsys-digest.c:85–133). Written only if the available `pkgbin` does not already
  ship one (:57–58).
- **`diversions`.** Three lines per entry: original path, diverted path, package name or `:` for local. Lines
  are read with `fgets_checked`; lines over 1024 bytes are fatal (MAXDIVERTFILENAME, dpkg.h:53). The reader
  builds the cross-linked `fsys_diversion` pairs and errors on conflicts (db-fsys-divert.c:41–96). **The library
  only reads this file**; `dpkg-divert` writes it (src side).
- **`statoverride`.** Lines have the form `<user|#uid> <group|#gid> <octal mode ≤ 07777> <path>`
  (db-fsys-override.c:109–236). Unknown users or groups are errors unless `STATDB_PARSE_LAX`. Names are resolved
  through dpkg's own passwd/group reader (D.8), not NSS. Read-only in lib.
- **Reload rule.** Both files are reloaded only when the `(st_dev, st_ino)` changed. The old `FILE*` is kept open
  so the inode cannot be reused (db-fsys-load.c:56–74).

### D.5 Triggers (triglib.c, trigdeferred.c, trignote.c, trigname.c)

- **Names.** `trig_name_is_invalid()` rejects empty names and any byte `<= ' '` or `>= 0177` (trigname.c).
  Classification (triglib.c:170–191):
  - a name starting with `/` without `//` or a trailing `/` is a file trigger;
  - a valid package-name-like string without `_` is an explicit trigger;
  - anything else is unknown, and an interest in it is an error.
- **`triggers/File`.** Lines have the form `<path> <pkg>[:arch][/noawait]` (with `pnaw_same` arch
  qualification). It is read once (`trig_file_interests_ensure`, :530–574), rewritten atomically with sync, and
  removed when empty (:487–528).
- **`triggers/<name>`.** Explicit-trigger interest files with one `<pkg>[/noawait]` per line, rewritten
  atomically (`trk_explicit_interest_change`, :350–405).
- **`triggers/Unincorp`.** The deferred activation queue, protected by `triggers/Lock`. That lock uses a
  **blocking** `F_SETLKW` (trigdeferred.c:73–138).
  - Grammar: lines of printable ASCII; the first token is the trigger name; then package names made of
    `[0-9a-z]` followed by `[0-9a-z:+.-]`, where `-` alone means "no awaiter". Lines are at most
    `_POSIX2_LINE_MAX`. Parsed by hand (trigdeferred.c:185–252).
  - Rewrite: write `Unincorp.new`, `fclose`, `rename`, then sync the directory. **There is no `fsync` of the new
    file before the rename** (trigdeferred.c:262–281). This is unlike every other atomic write in the library.
  - Incorporation happens at every `modstatdb_open`, including read-only ones (dbmodify.c:391;
    triglib.c:810–862).
- **Package `triggers` control file.** Directives `interest[-await|-noawait]` and `activate[-await|-noawait]`;
  `#` starts a comment; lines are at most 256 bytes (MAXTRIGDIRECTIVE) (`trig_parse_ci`, triglib.c:715–775).
- **State machine.**
  - Recording a pending trigger moves the package to `triggers-pending`, or to `triggers-awaited` if it is
    itself awaiting something (trignote.c:64–75).
  - The awaiter is moved to `triggers-awaited` (triglib.c:94–99).
  - `trig_clear_awaiters` (:102–129) unlinks the cross lists and promotes awaiters back to
    `installed`/`triggers-pending`.
- **Hooks.** `dpkg` replaces the default hooks with `trig_override_hooks()` (`src/main/trigproc.c:565–578`) to
  enqueue deferred processing and do transitional activation. Other tools keep the defaults (triglib.c:864–880).

### D.6 Other small files

- **`arch`.** One architecture per line, foreign plus native, written atomically with sync and a db-dir sync
  (arch.c:322–380). `arch-native` is honoured only when `DPKG_ROOT` is non-empty, i.e. in chroots
  (arch.c:288–316).
- **`dpkg.log`.** `YYYY-MM-DD HH:MM:SS <msg>` in localtime; opened `O_APPEND` lazily; failures only produce a
  `notice` (log.c:47–89).
- **status-fd.** Each message is written to every registered fd with embedded newlines mapped to spaces and a
  trailing `\n`; a write error is fatal (log.c:111–134). The list nodes are allocated **in the arena**
  (log.c:105). After `pkg_hash_reset()` the list would dangle, which is a latent use-after-free. It is
  unreachable today because the only multi-action loop, `--command-fd`, is commented out
  (`src/main/main.c:801–803`). I confirmed the built `dpkg` rejects `--command-fd`. (Inferred hazard; not
  reproduced.)

### D.7 Durability summary (every fsync, directory sync and rename in lib, from grep)

| artifact | file fsync | dir fsync | backup | code |
|---|---|---|---|---|
| status (checkpoint, replay) | yes | parent dir | `status-old` hard link | dump.c:571–579 |
| available | no | no | no | dump.c:563–576 |
| updates/NNNN | yes, after ftruncate | `updates/` | – | dbmodify.c:474–489 |
| info/*.list, *.md5sums | yes | `info/` | no | db-fsys-files.c:341–346, db-fsys-digest.c:76–81 |
| info/format (upgrade) | yes | parent and `info/` | – | db-ctrl-upgrade.c:175–218 |
| triggers/File, triggers/<name> | yes (unless removing) | `triggers/` | no | triglib.c:392–404, 508–525 |
| **triggers/Unincorp** | **no** | `triggers/` | no | trigdeferred.c:262–281 |
| arch | yes | db dir | no | arch.c:371–376 |

`HAVE_FSYNC_DIR` gates every directory fsync; it is 1 here (config.h:114). `dir_sync_contents()` fsyncs every
file in a directory and is used for the control-info temporary directory (`src/main/unpack.c:1341`).

### D.8 System user and group lookup

`dpkg_sysuser_from_name()` and its group counterparts **parse `/etc/passwd` and `/etc/group` themselves** with
`fgetpwent`/`fgetgrent` (sysuser.c:46–239), not with `getpwnam` and NSS.

- Paths are prefixed by `DPKG_ROOT`, with a fallback to the host file when the root copy is missing.
- The paths can be overridden with `DPKG_PATH_PASSWD` and `DPKG_PATH_GROUP`.
- Only `root` is cached.

---

## E. Archive and I/O layer

- **`ar.c`** (251 lines) provides header helpers, not a reader loop. The member-reading loop lives in `src/deb`.
  - Header layout: `struct dpkg_ar_hdr` (ar.h).
  - `dpkg_ar_member_parse_size` parses a decimal size with space padding.
  - `dpkg_ar_normalize_name` strips trailing spaces and a GNU `/`.
  - Writing: a `%-16s%-12jd%-6lu%-6lu%-8lo%-10jd`\n` header, checked to be exactly 60 bytes; names longer than
    15 characters are rejected; odd sizes are padded with `\n` (ar.c:181–250).
  - Pipes are detected with `lseek` and their size forced to 0 (ar.c:41–64).
- **`tarfn.c`** (619 lines) is a callback-driven extractor (`struct tar_operations`: `read`, `extract_file`,
  `link`, `symlink`, `mkdir`, `mknod`; tarfn.h).
  - Dialects: V7 ("old"), **ustar** (with `prefix` + `name` concatenation, :300–301) and **GNU**
    (`L`/`K` long name and long link, :363–406). Numeric fields may be **base-256** (:131–167). The checksum is
    the unsigned byte sum only (:257–278); historic signed-sum archives are not accepted.
  - Unsupported, with errors: PAX `x`/`g`, GNU `V`/`M`/`S`/`D`, Solaris `X`/`A`, and anything else, including
    `'7'` (CONTIG) (:563–591).
  - `'\0'` is treated as a regular file (old bug compatibility). A regular file whose name ends in `/` is
    treated as a directory (:524–536).
  - **Symlinks are queued and created after all other entries** (:540–551, 598–605).
  - An end-of-archive block is detected as a checksum failure with an empty name (:483–495).
  - Errors are reported through `tar->err` (`dpkg_error`) and the return code, not through `ohshit`. The
    callbacks in `src/main/archives.c` do `ohshit`, however.
- **`compress.c`** (1492 lines) holds the table `compressor_array[]` with entries none, gzip(`.gz`),
  xz(`.xz`), zstd(`.zst`), bzip2(`.bz2`) and lzma(`.lzma`) (:1345–1352). Each entry has `fixup_params`,
  `compress` and `decompress` function pointers.
  - **Library vs external command is a compile-time choice per format.** `USE_LIBZ_IMPL` selects zlib or zlib-ng
    (compat-zlib.h maps `gz*` to `zng_gz*`); then `WITH_LIBLZMA`, `WITH_LIBZSTD`, `WITH_LIBBZ2`. Without a
    library, it forks `gzip`/`xz`/`zstd`/`bzip2` with `-c<level>`/`-dc`, after unsetting `GZIP`, `BZIP`/`BZIP2`,
    `XZ_DEFAULTS`/`XZ_OPT` and `ZSTD_CLEVEL`/`ZSTD_NBTHREADS` in the child (`fd_fd_filter`, :65–111). This build
    links all four libraries: `Z_LIBS=-lz -llzma -lzstd -lbz2` in `build/Makefile`; `WITH_LIB*` are 1 in
    config.h:517–529.
  - **Threading.**
    - xz uses `lzma_stream_encoder_mt` and `lzma_stream_decoder_mt` (`HAVE_LZMA_MT_*`, config.h:168–171).
      Threads = `lzma_cputhreads()` clamped by `threads_max`. The memory limit comes from
      `/proc/meminfo` MemFree+Buffers+Cached (meminfo.c), or half the physical RAM, or 128 MiB. The encoder
      drops threads until the estimated memory use fits (:651–764).
    - zstd sets `ZSTD_c_nbWorkers` to the online CPU count (:1048–1069, 1156–1158).
    - These threads exist only inside the compression libraries. dpkg code itself is single-threaded, and
      `compress_filter` and `decompress_filter` are called in forked children (`src/deb/build.c:580–583`,
      `src/deb/extract.c:331–335`).
  - **Parameters.** Levels are 0–9, except zstd at 1–`ZSTD_maxCLevel()`. Strategies: gzip
    `filtered`/`huffman`/`rle`/`fixed`; xz `extreme` (:1429–1460). gzip level 0 becomes "none"; bzip2 level 0
    becomes 1 (:190–196, 368–374). zstd output carries a content checksum (:1152); xz uses CRC64 (:723).
  - **Verified behaviour difference.** The in-process **xz, zstd and bzip2 decoders stop silently after the first
    stream or frame**: no `LZMA_CONCATENATED` flag (:711–713); `ret == 0` marks the end (:1123–1124);
    `BZ2_bzread` stops at the end of the first stream. gzip (`gzread`) and the external CLIs decode all
    concatenated members. With a probe program on two concatenated streams `hello\n` + `world\n`, the library
    produced only `hello` for xz, zstd and bzip2, and `hello world` for gzip; the CLIs produced `hello world`
    for all four.
  - Each compression routine closes `fd_out` itself (e.g. :249, 476, 638, 1245).
- **`buffer.c`** (295 lines) has one generic copy loop: read fd → optional MD5 → write to fd, varbuf or null
  (`buffer_copy`, :179–236), wrapped by macros `fd_fd_copy`, `fd_md5`, `fd_vbuf_copy`, `fd_skip` (buffer.h).
  - MD5 is the only digest; it comes from **libmd** `<md5.h>` (`MD5Init`/`Update`/`Final`).
  - `fd_skip` tries `lseek` and falls back to reading on `ESPIPE`.
  - There is a "type" switch on integer constants plus a `union {void*; int}` argument (buffer.h:40–58).
- **`fdio.c`.**
  - `fd_read` and `fd_write` loop on EINTR/EAGAIN and return **`-total` on an error after a partial transfer**
    (fdio.c:48, 76).
  - `fd_allocate_size()` is compiled to `ENOSYS` unless `USE_DISK_PREALLOCATE`, which is off here
    (config.h:401). The variants are `F_PREALLOCATE` (macOS), `F_ALLOCSP64` (Solaris), `fallocate` (Linux) and
    `posix_fallocate` (non-glibc) (fdio.c:89–171).
- **`subproc.c`.**
  - `subproc_fork` (D.2 and C.2).
  - `subproc_reap`/`subproc_check` produce the messages "%s subprocess failed with exit status %d", "… was
    interrupted", "… was killed by signal (%s)%s".
  - `subproc_signals_ignore` ignores SIGINT and SIGQUIT around a child and restores them via a cleanup entry.
  - **Quirk:** `SUBPROC_RETERROR` and `SUBPROC_RETSIGNO` are both `DPKG_BIT(3)` (subproc.h:46, 48), so asking
    for one also enables the other (used at `src/main/unpack.c:112`).
- **`command.c`.**
  - argv builder; `command_exec` is `execvp` followed by `ohshite`.
  - `command_shell` runs `$SHELL -i` or `sh -c [--] cmd` (`HAVE_DPKG_SHELL_WITH_DASH_DASH`).
  - `command_in_path` scans `PATH` and is fatal if `PATH` is unset (command.c:227–259).
- **`file.c`.** Locks (D.2), `file_slurp` (dpkg_error style), realpath and canonicalize (with a lexical fallback
  through `path_canonicalize`), `file_copy_perms`, and `file_show` through the pager.
- **`dir.c`.** `dir_make_path` (`mkdir -p`), `dir_sync_path`, `dir_sync_path_parent`, `dir_sync_contents`.
- **`treewalk.c`** (552 lines) is used by dpkg-deb build/info and by `dpkg`.
  - Pre-order depth-first iterator with sorted children (`strcmp`); the comment calls it "Breath first visit",
    but the code is depth-first (treewalk.c:545–547).
  - Detects directory loops by `(dev, ino)`. Lazy `stat`: d_type is used when available, except for dirs and
    symlinks.
  - Options: `TREEWALK_FOLLOW_LINKS`, `TREEWALK_FORCE_STAT`. Callbacks: `visit`, `sort`, `skip`.
  - Minor over-allocation: the child array is sized with `sizeof(*node)` instead of `sizeof(node*)`
    (treewalk.c:219).
- **`path-remove.c`.**
  - `secure_unlink` does `chmod 0600` before unlinking setuid/setgid/sticky regular files and unknown file types.
  - `path_remove_tree` tries `rmdir`, then `unlink`, then **forks `rm -rf --`** (path-remove.c:120–163).

---

## F. Other logic

### F.1 Version comparison (`version.c`, exact algorithm)

- `dpkg_version_compare(a, b)` (version.c:140–155): compare `epoch` numerically (unsigned); then
  `verrevcmp(version)`; then `verrevcmp(revision)`. NULL and "" are equivalent (version.c:84–87).
- `verrevcmp(a, b)` (:82–117) runs while either string has characters:
  1. **Non-digit prefix.** While `*a` is a non-digit or `*b` is a non-digit, compare `order(*a)` with
     `order(*b)` and return the difference if they differ. A string that has ended contributes `order('\0') = 0`.
  2. Skip leading `'0'`s on both sides.
  3. **Digit run.** While both are digits, remember the first difference `*a - *b`.
  4. If `a` still has a digit, return 1; if `b` does, return -1; otherwise return the first difference if
     non-zero.
- `order(c)` (:67–79):
  - digit → 0;
  - letter `[A-Za-z]` → its ASCII code;
  - `~` → −1;
  - NUL → 0;
  - **any other byte → c + 256**.
- Consequences: `~` sorts before everything, including the end of the string; letters sort before
  non-letters; numeric runs compare by value ignoring leading zeros (`1.01 == 1.1`, verified).
- **Platform-dependent quirk.** `c` comes from a plain `char`. With signed `char` (x86) a byte ≥ 0x80 maps to
  128..255, which sorts **below** ASCII punctuation (`.` → 302) but above letters. With unsigned `char` (arm64,
  ppc64el, s390x) the same byte maps to 384..511, which sorts **above** punctuation.
  - Verified on x86-64: `dpkg --compare-versions $'1\xc3' lt 1.` is true (with a "bad syntax" warning).
  - The unsigned-char half is inferred from the code (unverified).
- The return value is a non-normalised difference; callers use only its sign (unverified across all callers).
- `dpkg_version_relate()` handles `NONE`/`EQ`/`LT`/`LE`/`GT`/`GE` (`LE = LT|EQ`, `GE = GT|EQ`; version.h).
- **`parseversion()`** (parsehelp.c:242–324):
  - Trims blanks (space and tab only); embedded blanks are an error.
  - The epoch is parsed with **`strtol`**, so `+1:1.0` is valid (epoch 1) and `-0:1.0` is valid (epoch 0), both
    verified. `00:1` is valid. The epoch must be ≤ `INT_MAX` even though it is stored as unsigned.
  - The revision is split at the **last** `-`. An empty revision after `-` is an error.
  - Character-class problems are *warnings* (`dpkg_put_warn`). They become errors only when the parse is not lax
    (`parse_db_version`, :339–353).
- **Rendering.** `versiondescribe` returns one of **10 rotating static buffers** (parsehelp.c:194–214).
  `vdew_nonambig` prints the epoch if it is non-zero or if the upstream or revision part contains a `:`
  (:170–176).

### F.2 Architectures (`arch.c`)

- Built-in static items: none (`""`), empty (`""`), `any`, `all`, and native (`ARCHITECTURE` = "amd64").
- Other names are appended as arena nodes typed `UNKNOWN`, `INVALID` or `FOREIGN` (arch.c:139–167).
- `dpkg_arch_reset_list()` truncates the list back to the built-ins and must run before `nffreeall()`
  (arch.c:209–220).
- Valid names start with an alphanumeric and contain only alphanumerics and `-` (arch.c:58–80).

### F.3 Dependency predicates (`depcon.c`)

- `deparchsatisfied` encodes the Multi-Arch rules (depcon.c:41–82):
  - an implicit arch is satisfied by an M-A: foreign provider;
  - `:any` is satisfied by M-A: allowed, and always for Conflicts, Breaks and Replaces;
  - otherwise the arch pointers must be equal after mapping `all`/none to native.
- `pkg_virtual_deppossi_satisfied` only honours unversioned or `=` versioned Provides (:113–127).

### F.4 Package specifiers (`pkg-spec.c`)

- `name[:arch]` with `fnmatch` globs when `PKG_SPEC_PATTERNS` and the string contains any of `*[?\`.
- `PKG_SPEC_ARCH_SINGLE` is an error if more than one instance is installed. `PKG_SPEC_ARCH_WILDCARD` is the
  alternative.
- `pkg_spec_parse_pkg` returns NULL plus an error, but can still `ohshit` through `pkg_hash_find_singleton`
  (pkg-hash.c:149–161).
- Messages come from a static 1024-byte buffer (pkg-spec.c:65).

### F.5 Name rendering and status tables (`pkg-show.c`, `pkg-namevalue.c`)

- `pnaw_never`, `nonambig`, `same`, `foreign` and `always` decide when `:arch` is shown (pkg-show.c:34–74).
- `pkgbin_name()` caches the result in the arena.
- The `*_abbrev` tables give `dpkg -l` letters (pkg-namevalue.c:61–103).
- The tables use `NAMEVALUE_DEF(n, v)` = `[v] = {…}` designated indexes (namevalue.h), so they are indexable by
  enum value.
- `pkg_source_version` parses `Source: name (ver)` and `ohshit`s on a bad version (pkg-show.c:414–441).
- `pkg_sorter_by_nonambig_name_arch` returns −1 for *both* orderings when neither instance needs an arch
  qualifier but the arch pointers differ (pkg-show.c:383–390). That makes it a non-antisymmetric qsort
  comparator; the practical impact is unverified.

### F.6 Format engine (`pkg-format.c`)

- Syntax: `${field}` or `${field;width}`. A negative width means left-justify, enforced by
  `%*s` plus truncation. String escapes are `\n`, `\t`, `\r` and `\<c>` (pkg-format.c:73–150, 400–461).
- Field lookup order: `fieldinfos` (case-insensitive) → 12 virtual fields → the package's arbitrary fields:
  - `binary:Package`, `binary:Synopsis`, `binary:Summary`;
  - `db:Status-Abbrev`, `db:Status-Want`, `db:Status-Status`, `db:Status-Eflag`;
  - `db-fsys:Files`, `db-fsys:Last-Modified`;
  - `source:Package`, `source:Version`, `source:Upstream-Version` (:368–381).
- The real fields are rendered by the same `w_*` writers, with `fw_printheader` cleared.

### F.7 Options and configuration (`options.c`)

- Config files are processed in this order: `CONFIGDIR/<prog>.cfg.d/*` (names `[A-Za-z0-9_-]`, `alphasort`),
  then `CONFIGDIR/<prog>.cfg`, then `$HOME/.<prog>.cfg`.
- Line syntax: `name[ =]value`; quotes are stripped; `#` starts a comment; lines are at most 1024 bytes
  (options.c:70–239).
- On the command line, `takesvalue` 0, 1 or 2 means no value, `=value` or a separate argument, and the
  `--option-value` concatenation respectively (options.h:38–48; options.c:236–338).
- `badusage` appends the program's "Type … --help" text and `ohshit`s.
- `set_instdir` touches `dpkg_db_get_dir()` first, so the admindir is not re-derived from the new root
  (options-dirs.c:33–41).

### F.8 Small utility modules

- **`varbuf.c`** (376 lines).
  - Grows to `(size + need) * 2`; overflow is fatal; the buffer is always NUL-terminated.
  - Snapshot and rollback are offset-based.
  - `varbuf_detach` returns `""` from `m_strdup` when empty.
  - `varbuf_has_suffix` uses `strcmp`, so it is unsafe with embedded NULs.
  - **varbuf.h has C++ member wrappers** for dselect (varbuf.h:56–116, 218–523).
- **`strvec.c`.** split, join, push, pop, drop; used by `path_canonicalize`.
- **`string.c`.**
  - `str_fmt` (always succeeds or `ohshit`s), `str_escape_fmt`, `str_strip_quotes`, `str_rtrim_spaces`.
  - **`str_quote_meta` allocates `2*strlen` without room for the NUL.** ASan confirms a heap-buffer-overflow at
    string.c:172 for an all-metacharacter input. It is only used by unit tests (grep).
- **`strwide.c`.** `str_width` and `str_gen_crop` use `mbsrtowcs` and `wcswidth`, falling back to bytes on
  invalid multibyte input unless `DPKG_UNIFORM_ENCODING`.
- **`c-ctype.c`.** A 256-entry table; `c_isbits` casts to `unsigned char`. Classes: blank `[ \t]`, white
  `[ \t\n]`, space `[ \v\t\f\r\n]`.
- **`glob.c`.** A list of pattern strings (no matching logic).
- **`i18n.c`.**
  - `DPKG_NLS=0` or empty disables `setlocale`.
  - A C locale created with `newlocale` is switched in with `uselocale` (for logging versions in the C locale,
    dbmodify.c:536).
  - On macOS a dummy `gettext("")` call warms up CoreFoundation before any `fork` (i18n.c:65–77).
- **`color.c`.**
  - `DPKG_COLORS=auto|always|never`. **`auto` checks `isatty(STDOUT)` even for messages written to stderr**
    (color.c:36–51).
- **`pager.c`.**
  - Pager choice: `DPKG_PAGER`, then `PAGER`, then `pager`, `less`, `more`, falling back to `cat` when stdin or
    stdout is not a tty.
  - Sets `LESS=-FRSXMQ` if unset, ignores SIGPIPE while paging, and redirects fd 1 through a pipe
    (pager.c:53–159).
- **`progress.c`.** Prints `text` plus `NN%\r` in 5 % steps, on a tty only.
- **`report.c`.**
  - Output formats: `prog: warning: msg`, `prog: msg` (notice), `prog: hint: msg` to stdout, and `prog: msg`
    (info) to stdout.
  - Keeps a warning counter (used by dpkg-deb's "ignoring N warnings").
  - The warning printer can be replaced.
  - `dpkg_set_report_buffer` makes stdout line-buffered on a tty, otherwise uses `piped_mode`.
- **`debug.c`.** An octal mask from `-D` or `DPKG_DEBUG`; output lines are `D0%05o: [fn(): ]msg`. The octal
  values are a documented interface (debug.h:41–55).
- **`meminfo.c`.**
  - Sums MemFree + Buffers + Cached, **not** MemAvailable; Linux and Hurd only.
  - **One-byte stack overflow if the file is ≥ 4096 bytes**: `buf[4096]` and `buf[bytes] = '\0'`
    (meminfo.c:83–108). ASan confirms this at meminfo.c:108 with a 5000-byte input. The real `/proc/meminfo`
    here is 1615 bytes.
- **`sysuser.c`.** See D.8.
- **`execname.c`.** `/proc/<pid>/exe` on Linux, libps on Hurd, `/proc/<pid>/execname` on Solaris,
  `proc_pidpath` on macOS, psinfo on AIX, `sysctl KERN_PROC_PATHNAME` on FreeBSD.
- **`progname.c`.** `program_invocation_short_name`, `__progname`, `getprogname`, `getexecname` or `getprocs64`,
  depending on the platform.
- **`term.c`.** `COLUMNS`, then `TIOCGWINSZ` on `/dev/tty`, then 80.
- **`deb-version.c`.** Strict `X.Y\n` parser for `debian-binary`.
- **`utils.c`.** `fgets_checked` makes a too-long line or a missing newline fatal.

---

## G. Global mutable state inventory

Counted with grep heuristics:

- About 80 file-scope mutable variable declarations in 28 `.c` files (71 lines from the first regex plus 8 from
  the second; a few lines declare several variables, and a few entries are tables that are never mutated).
- 18 function-scope `static` variables.
- One non-static global, `cipaction` (options.c:455).

| area | globals (file:line) |
|---|---|
| Error handling | `econtext` (ehandle.c:82), `onerr_abort` (:41), `emergency` (:90), `preventrecurse` (:263, function-static) |
| Reporting | `piped_mode`, `warning_printer_func`/`_data`, `warn_count`, `hints_enabled` (report.c:37–134); `progname` (progname.c:36); `debug_mask`, `debug_output` (debug.c:35–36); `color_mode`, `use_color` (color.c:32–33); `dpkg_C_locale` (i18n.c:34); `pager_enabled` (pager.c:39) |
| Directories | `db_dir` (dbdir.c:33), `fsys_dir` (fsys-dir.c:32), `db_infodir`, `db_format`, `db_upgrading` (db-ctrl-format.c:37–39) |
| Arena and hashes | `db_obs`, `dbobs_init` (nfmalloc.c:35–36); `bins[65521]`, `npkg`, `nset` (pkg-hash.c:48–49); `bins[262139]`, `nfiles` (fsys-hash.c:40–41); arch list and statics (arch.c:85–113) |
| modstatdb | `db_initialized`, `cstatus`, `cflags`, 6 path strings, `importanttmp`, `nextupdate`, `updateslength`, `updatefn` + state, `uvb`, `dblockfd`, `frontendlockfd` (dbmodify.c:49–62, 232–233) |
| fsys DB | `diversions` (db-fsys-divert.c:39); `allpackagesdone`, `saidread` (db-fsys-files.c:56, 71); static `struct dpkg_db` in `ensure_diversions` and `ensure_statoverrides` |
| Triggers | `triggersdir`, `triggersfilefile`, `trigh` hooks, `dtki`, `trig_activating_name`, `trk_explicit_f`/`_fn`/`_trig`, `filetriggers` list, `filetriggers_edited`, `trk_file_trig` (triglib.c:46–631); `fn`, `newfn`, `trigdef`, `triggersdir`, `lock_fd`, `old_deferred`, `trig_new_deferred` (trigdeferred.c:42–51); `trig_awaited_pend_head` (trignote.c:104); `rename_head` (db-ctrl-upgrade.c:49) |
| Logging | `log_file`, `status_pipes` (log.c:38, 96); function-static `log` varbuf and `logfd` (log.c:50–51); `vb` (log.c:114) |
| Misc | `sa_save[]` (subproc.c:43); `root_pw`, `root_gr` (sysuser.c:43–44); `printforhelp`, `cipaction` (options.c:41, 455) |
| Static return buffers (non-reentrant) | `pkg_name_is_invalid` buf[150] (parsehelp.c:142); `dpkg_arch_name_is_invalid` buf[150] (arch.c:60); `versiondescribe` 10 rotating varbufs (parsehelp.c:198); `pkg_infodb_get_file` varbuf (db-ctrl-format.c:131); `pkg_spec_is_invalid` msg[1024] (pkg-spec.c:65); `f_dependency` depname/version/arch varbufs (fields.c:455, 535); `scan_word` (fields.c:714); `combuf` (compress.c:95); `writebuf[8192]` (dump.c:517) |

**Signals.** The library installs **no signal handlers**. It only sets `SIG_IGN`: for SIGINT and SIGQUIT around
child processes (subproc.c:65–96), and for SIGPIPE while a pager runs (pager.c:113–120, 156).

**fork.** In the library, `fork` happens in `subproc_fork` (subproc.c:109). Its library callers are the
external-compressor fallback (compress.c:70), `path_remove_tree` (path-remove.c:155) and `pager_spawn`
(pager.c:122). `src` calls it directly 14 times.

**Threads.** There is no pthread use and no thread-local storage anywhere in lib (grep for `pthread`,
`_Thread_local`, `__thread`). Everything assumes one thread: the global error and cleanup stacks, the arena,
the static return buffers, and the `uselocale` switching. The only threads in the process are xz and zstd
workers inside the libraries, run in forked compression children.

**Environment variables read by lib:** `DPKG_ADMINDIR`, `DPKG_ROOT`, `DPKG_COLORS`, `DPKG_DEBUG`, `DPKG_NLS`,
`DPKG_FRONTEND_LOCKED`, `DPKG_PAGER`, `PAGER`, `SHELL`, `PATH`, `HOME`, `TMPDIR`, `COLUMNS`,
`DPKG_PATH_PASSWD`, `DPKG_PATH_GROUP` (grep `getenv`).

---

## H. C patterns relevant to a Rust port

| pattern | where and how many (method) |
|---|---|
| **Non-local exits (setjmp/longjmp)** | See C.3. About 286 raising sites in lib plus 57 `parse_error`; every allocator. Rust frames must never be longjmp'd over while they own values with destructors (undefined behaviour; unverified detail of current Rust rules — the "plain old frames" doctrine). Rust cannot itself call `setjmp`. |
| **Arena allocation never freed individually** | `nfmalloc` (obstack): 44 call sites in lib and 34 in src. Pointers into the arena are stored in every model struct and outlive any single function. Freed in bulk by `pkg_hash_reset`. `src/main/archives.c:78–115` keeps a second obstack (`tar_pool`). |
| **Intrusive linked lists** | 37 `struct X *next/prev/...` fields across headers and sources (grep). These include doubly-linked reverse-dependency lists (`deppossi.rev_next/rev_prev`), nodes on two lists at once (`trigaw`, `trigfileint` using the `dlist.h` macros `LIST_LINK_TAIL_PART`/`LIST_UNLINK_PART`), hash chains, `arch_next`, and the embedded first `pkginfo` inside `pkgset`. |
| **Cyclic object graph** | `pkgset ↔ pkginfo` (`set`, `pkg`, `arch_next`), `dependency.up` → `pkginfo`, `deppossi.ed` → `pkgset` with the reverse list back, `trigaw.aw/pend`, `fsys_namenode.packages` → `pkginfo` with `pkginfo.files` → namenode, and diversion pairs. |
| **Pointer-arithmetic parsers** | `parse_stanza` (`dataptr`/`endptr` macros), `f_dependency`, `conffvalue_lastword` (backward scan), `fsys_list_parse_buffer`, `parse_filehash_buffer`, `ensure_statoverrides`, `trigdef_parse`, `tar_atol8`/`tar_atol256`, `meminfo`, `pkg_format_parse`. All parse NUL-terminated or `[ptr,end)` buffers *in place*, writing NULs into the buffer. |
| **Unions and tagged data** | `error_context.handler` (ehandle.c:67), `io_zstd_stream.ctx` (compress.c:1019), `buffer_data.arg` (buffer.h:52, with an int `type` tag). `fieldinfo.integer` is **overloaded**: a struct offset for `f_charfield`/`f_boolean`/`f_multiarch`/`f_version`/archives, and an `enum deptype` for `f_dependency` (parse.c:55–94). |
| **offsetof-based field setters** | `PKGIFPOFF`/`ARCHIVEFOFF` and `STRUCTFIELD(klass, off, type)` (parsedump.h:98–101); 23 grep hits. `f_multiarch` and `w_multiarch` write and read an `enum pkgmultiarch` field **as `int`** (fields.c:214, dump.c:198). |
| **Function-pointer tables** | `fieldinfos[]` (37 entries × read and write functions), `virtinfos[]` (12), `compressor_array[]` (6 × 3), 3 `trigkindinfo` tables (× 4), `trig_hooks` (5), `trigdefmeths` (3), `tar_operations` (6), `treewalk_funcs` (3), `cmdinfo` (`call`, `action`, `iassignto`/`sassignto` raw pointers into the program's globals), and cleanup callbacks `fn(int argc, void **argv)`. |
| **Code-generating macros** | `TRIGHOOKS_DEFINE_NAMENODE_ACCESSORS` (defines 3 static functions, triglib.h:83–92), `NAMEVALUE_DEF` (designated index), `FIELD()` (two initialisers), `ACTION`/`ACTION_MUX`/`OBSOLETE` (options.h), the `fd_*` copy macros (buffer.h), `TAR_ATOUL`/`TAR_TYPE_MAX` (tarfn.c:50–64), the `internerr` macro (`__FILE__`/`__LINE__`/`__func__`), `debug`/`debug_at`, and the `test.h` harness (`test_try` uses setjmp). |
| **Printf-style varargs APIs** | 38 `DPKG_ATTR_PRINTF`/`VPRINTF` annotations in installed headers (about 34 functions): `ohshit*`, `warning*`, `notice`, `info`, `hint`, `dpkg_put_*`, `varbuf_*fmt`, `str_fmt`, `m_asprintf`, `log_message`, `statusfd_send`, `debug_print`, `badusage`, `parse_error`/`warn`/`problem`, `compress_filter`, `trigdef_update_printf`, …. There are 422 `_()`/`N_()`/`C_()`/`P_()` call sites in lib. `po/dpkg.pot` has 1332 msgids, 848 of them flagged `c-format`, across 44 `.po` catalogs. **The translated texts are C printf format strings.** |
| **Bitfields** | None (grep for `: N;` in structs found nothing). Flag enums use `DPKG_BIT(n)` plus `DPKG_ATTR_ENUM_FLAGS`. |
| **goto** | 7 in lib: fields.c:362, 366 (`malformed`); triglib.c:179, 454, 469, 544. |
| **Generated code (lex/yacc)** | None: no `.l` or `.y` files in the tree. All parsers are handwritten. |
| **Aliasing of `installed` and `available`** | 13 identity tests, `pkgbin == &pkg->installed` (B.2). `parse.c` builds stanzas into a temporary `pkgset` on the stack and `memcpy`s the `pkgbin` (parse.c:485), then patches back-pointers (:497–505). |
| **Static buffers returned to callers** | See G. |
| **C++ coexistence** | `varbuf.h` exposes C++ methods; `t-headers-cpp.cc` checks that every header compiles as C++. `dselect` (C++) relies on longjmp from `ohshit` crossing C++ frames, so its destructors are skipped (inferred). |

---

## I. Public interface

- **What is built and installed.**
  - `libdpkg.la` is a libtool library installed into `devlibdir` (`devlib_LTLIBRARIES`,
    lib/dpkg/Makefile.am:24). Only the static library is allowed: configure uses `LT_INIT([disable-shared])`
    (configure.ac:48), and `DPKG_BUILD_SHARED_LIBS` **errors out on `--enable-shared`** unless `AUTHOR_TESTING`
    (m4/dpkg-build.m4:6–10). The build tree has only `libdpkg.a`.
  - 52 headers are installed to `$(includedir)/dpkg/` (`pkginclude_HEADERS`). Not installed: `dlist.h`,
    `i18n.h`, `perf.h`, `test.h`.
  - `libdpkg.pc` declares `Libs: -ldpkg` and puts `PS/MD/Z/LZMA/ZSTD/BZ2` libraries in `Libs.private`
    (libdpkg.pc.in).
  - Debian ships this as `libdpkg-dev`: `usr/include/dpkg/*.h`, `libdpkg.a`, `libdpkg.pc`, and the aclocal macros
    (debian/libdpkg-dev.install; debian/control:88–).
- **`libdpkg.map`.**
  - Two version nodes: `LIBDPKG_0` with 22 symbols (error reporting, locales, program name and init, ar, execname)
    and `LIBDPKG_PRIVATE` with 401 symbols.
  - It is used only as `--version-script` when the linker supports it, or turned into `libdpkg.sym` by `sed`
    (Makefile.am:30–46, 198–207). Both matter only for a shared build.
  - It has drifted from the code. `libdpkg.a` defines 499 global symbols, of which **77 are not in the map**,
    including `buffer_copy_IntInt`, `buffer_copy_IntPtr` and `buffer_skip_Int`. **`src/` objects reference those
    three** through the `fd_*` macros, so a shared build would break if the map were enforced. The map also
    lists `color_reset`, which is `static inline` in color.h:88.
- **Stability policy** (doc/README.api): libdpkg.a's "Status: volatile … only supposed to be used internally by
  dpkg … Header files, functions, variables and types might get renamed, removed or change semantics".
  Consumers must define `LIBDPKG_VOLATILE_API`. `macros.h:29–31` has an `#error` without it, and configure
  defines it for in-tree builds (configure.ac:248).
- **Consumers inside the repo.**
  - Of the 499 global symbols in `libdpkg.a`, 323 are referenced by objects in `build/src` and `build/dselect`
    (`nm -u` intersection).
  - `src/` reaches into internals directly, not only through accessor functions:
    - struct fields: 102 `clientdata` uses; 54 `FNNF_*` flag uses on `fsys_namenode`; 23 uses of
      `trigpend_head`/`trigaw.`/`othertrigaw_head`; 22 uses of `depended.` and `rev_next`;
    - 36 `nf*` arena allocations;
    - `trig_override_hooks` with `TRIGHOOKS_DEFINE_NAMENODE_ACCESSORS` (src/main/trigproc.c:565–578);
    - `parsedb_new`/`parse_stanza`/`parse_error` (src/split/split.c:82);
    - `fieldinfos` (src/deb/info.c);
    - three JUMP error contexts (C.2).
  - `dselect` uses mainly `dpkg-db.h`, `dpkg.h` and `i18n.h`, plus `varbuf_stanza`, and defines its own
    `perpackagestate`.
  - `utils/` does not link libdpkg.
  - In practice the API is the set of struct layouts and globals, not a narrow function interface.

---

## J. Portability

- **`lib/compat/` contents (3,690 lines).** Each object is compiled only if configure finds the function missing
  (lib/compat/Makefile.am:42–114):
  - `getopt.c`, `getopt1.c`, `getopt*.h`: GNU getopt and getopt_long, used only by `utils/start-stop-daemon.c`.
  - `obstack.[ch]`: GNU obstack, needed by nfmalloc and `src/main/archives.c` on non-glibc systems (musl, BSDs,
    macOS; inferred from what glibc provides).
  - `strnlen`, `strndup`, `strchrnul`, `strerror` (with `sys_errlist`), `strsignal` (with its own
    `sys_siglist`), C99 `snprintf`/`vsnprintf`, `asprintf`/`vasprintf`, `fgetpwent`/`fgetgrent` (getent.c, added
    2025, for systems lacking them), `alphasort`, `scandir`, `unsetenv`.
  - `compat.h` macros: `offsetof`, `countof`, `makedev`, `O_NOFOLLOW`, `P_tmpdir`, `WCOREDUMP`, `va_copy`.
  - `compat-zlib.h`: zlib-ng `zng_*` mapping. `gettext.h`: NLS macros.
  - A `libcompat-test.la` variant compiles everything with `test_`-prefixed names for `t-compat-getent`.
  - On this glibc build **`libcompat.a` contains only `empty.o`** (`nm` shows `libdpkg_empty_dummy_symbol`).
- **OS-specific code in lib/dpkg** (grep of `#if` lines):
  - `execname.c:30–186`: Linux, Hurd (`__GNU__`, libps), Solaris (`__sun`), macOS, AIX, FreeBSD/kFreeBSD.
  - `meminfo.c:162`: Linux and Hurd only; elsewhere the MT xz memory limit falls back to `lzma_physmem()/2`.
  - `i18n.c:65`: macOS gettext and fork workaround.
  - `fdio.c:110–163`: macOS `F_PREALLOCATE`, Solaris `F_ALLOCSP64`, Linux `fallocate`, and
    `posix_fallocate` on non-glibc or kFreeBSD (that condition spans multiple lines).
  - `db-fsys-files.c:26–31, 170–264`: Linux `FIEMAP` ioctl, else `posix_fadvise`.
  - `progname.c`: glibc, BSD, Solaris and AIX name sources.
  - `tarfn.c:25`: `sys/sysmacros.h`.
  - `dir.c`: `HAVE_FSYNC_DIR`.
  - `command.c`: `HAVE_DPKG_SHELL_WITH_DASH_DASH`.
  - Feature `#if` counts (grep): `HAVE_USELOCALE` 5, `WITH_LIBLZMA` 4, `WITH_LIBZSTD` 3, `USE_MMAP` 3,
    `USE_LIBZ_IMPL*` 6, `HAVE_LZMA_MT_*` 5, `HAVE_FSYNC_DIR` 3, `ENABLE_NLS` 3, and others.
- **External libraries** (m4/dpkg-libs.m4; build/Makefile):
  - **libmd** (`MD5Init` in `<md5.h>`) is **mandatory**. Configure fails without it unless libc has it built in,
    as on BSDs (m4/dpkg-libs.m4:9–26).
  - **zlib or zlib-ng, liblzma, libzstd, libbz2** are each optional through `--with-lib<x>` (check, yes, static,
    no). Without one, the corresponding external CLI is used. This build links all four.
  - **libselinux** is linked into `src/` only. **It is not used in lib/dpkg** (grep shows no selinux reference).
  - **libps** (Hurd, `execname.c`) and **libkvm** (BSD, src side) are platform extras.
  - `LIBINTL` is empty with glibc.

---

## K. Unit tests (`lib/dpkg/t/`)

- **Harness.** `test.h` produces TAP: `test_plan`, `test_pass`/`fail`/`str`/`mem`/`warn`/`error`, todo and skip,
  plus `test_try`/`catch` built on setjmp and `push_error_context_jump`.
- **Programs.** 36 C/C++ test programs, 5 helper programs (`b-fsys-hash`, `b-pkg-hash` benchmarks;
  `c-tarextract`, `c-treewalk`, `c-trigdeferred` drivers), and 3 Perl `.t` scripts that drive the `c-*` helpers
  against real `tar` output, real trees and Unincorp files. **The sum of `test_plan(N)` is 3950**, of which
  `t-c-ctype` alone is 2063 (exhaustive over 256 bytes).
- **Execution.** I ran all 36 built test programs from the scratchpad. All passed their plans. The `not ok` lines
  in `t-test` (3) and `t-pager` (2) are TODO-marked by design. `t-file` needs `abs_builddir` set; without it, it
  segfaults in the harness.

| test | plan | covers |
|---|---|---|
| t-test, t-test-skip, t-macros, t-headers-cpp | 22, –, 12, 1 | harness; macros; all headers compile as C++ |
| t-c-ctype, t-namevalue, t-string, t-strvec, t-varbuf(+cpp), t-path | 2063, 5, 74, 99, 205+226, 47 | string and buffer primitives (thorough) |
| t-ehandle, t-error | 3, 24 | func and jump handlers, a cleanup that errors; `dpkg_error` |
| t-file, t-buffer, t-meminfo, t-progname, t-subproc, t-command, t-pager, t-sysuser, t-compat-getent | 43, 10, 10, 3, 6, 73, 3, 210, 236 | OS helpers (pager only partially, since it needs a tty) |
| t-ar, t-tar, t-deb-version | 4, 38, 24 | ar helpers; tar numeric fields (`tar_atoul`/`atosl`); deb-version |
| t-arch, t-version | 58, 196 | arch list; version parse and compare |
| t-pkginfo, t-pkg-list, t-pkg-queue, t-pkg-hash, t-pkg-show, t-pkg-format | 28, 14, 38, 72, 14, 24 | model primitives |
| t-fsys-dir, t-fsys-hash, t-trigger, t-mod-db | 16, 35, 9, 5 | root dir; fsys hash; trigger-name validation; admindir getter and setter only |
| t-tarextract.t / t-treewalk.t / t-trigdeferred.t (Perl) | 3 `ok`-style assertions each, looping over cases | tar extraction vs GNU tar; tree walk order; Unincorp parsing |

**Notable gaps.** No unit test references these functions (grep of `lib/dpkg/t/`):

- the deb822 parser and writer (`parsedb`, `parse_stanza`, `writedb`, `varbuf_stanza`);
- **all of `modstatdb`** (journal, locks, checkpoint);
- the info DB (`pkg_infodb_*`, the format upgrade);
- `ensure_diversions`, `ensure_statoverrides`, `ensure_packagefiles_available`, `write_filelist_except`,
  `parse_filehash`;
- trigger state handling (`trig_parse_ci`, `trig_incorporate`, `trig_note_*`, file-trigger interests);
- `compress_filter`/`decompress_filter` (no unit test at all);
- `atomic_file`, `file_lock`, `dir_sync*`, `path_remove_tree`, `secure_*`;
- `pkg_spec_*`, `pkg_array_*`, the `depcon` predicates;
- `options` parsing and config loading;
- `log_message`, `statusfd_send`, `str_width`, `str_gen_crop`, `progress`, `color`, `debug`, `execname`, `term`,
  `treewalk` (only via the Perl driver), and the `nfmalloc` lifecycle.

These are exercised only indirectly, through the program-level autotests in `src/at/*.at` and the separate
`tests/` functional suite (other analysts' areas).

---

## L. Rust-port assessment per subsystem

Legend:

- Difficulty: S (days), M (weeks), L (month+), XL (multi-month, design work).
- Seam: whether a Rust implementation could sit behind the existing C header through FFI while the rest stays C.

| subsystem | raw / ~code LOC | difficulty | seam? | reason and precise entanglement |
|---|---|---|---|---|
| ehandle + mustlib + error + cleanup | ~1,100 / ~700 | M to write, **XL to interoperate** | **Entangled (the root problem)** | Callers depend on `ohshit` never returning, on JUMP contexts `longjmp`ing to C `setjmp` sites (3 in src), and on cleanups being run by whoever unwinds. A Rust `ohshit` could call libc `longjmp` into C frames, but any Rust frame on the path holding a `Drop` value makes that undefined behaviour, and Rust cannot host the `setjmp` site. A mixed port needs C "firewall" shims that `push_error_context_jump`, call the C code and convert to `Result`, around every C call from Rust, and the reverse for Rust code that C expects to raise. The printf-style `ohshit(fmt, ...)` cannot be *defined* in stable Rust (C-variadic definitions are unstable; unverified as of 2026), so it needs a C shim that `vasprintf`s first. |
| Reporting, i18n, colour, debug, progname, program | ~900 / ~500 | S | Mostly a seam | Simple globals. The catch is printf-format translations: gettext catalogs hold C format strings (848 `c-format` msgids), so a Rust port needs a runtime printf-compatible formatter or must keep calling C `vfprintf`. Output text and colour rules must match (C.2, F.8). |
| varbuf, strvec, string, c-ctype, strhash, namevalue, glob, path, utils | ~2,300 / ~1,450 | S | **Seam with layout constraints** | Pure leaves, except that OOM and overflow `ohshit`. `struct varbuf {used,size,buf}` is public and touched directly (~217 regex hits in src, ~215 in lib), and buffers are freed with `free()` by callers (`varbuf_detach`), so a Rust version must keep `#[repr(C)]` and the malloc allocator. The varbuf C++ wrappers must keep compiling for dselect. |
| nfmalloc arena | 91 / ~50 | S | **Clean seam** (3 functions) | Trivial to reimplement as a bump allocator. The hard part is the *semantics*: everything lives until `nffreeall`, and some objects (status-fd list, fsys hash) silently outlive it. A Rust model would want an arena with explicit lifetimes or index-based handles. |
| version (+ parseversion, varbufversion) | ~450 / ~280 | S | Clean seam | Pure functions. `parseversion` allocates into the arena (call `nfstrnsave` through FFI). Must reproduce `strtol` epoch quirks and the signed-char `order()` quirk (F.1). The Perl `Dpkg::Version` must keep agreeing (other analyst). |
| arch | 468 / ~280 | S–M | Partial seam | Pointer-identity interning: `arch == arch` compares pointers (depcon.c:76 and others). Nodes are in the arena, and `dpkg_arch_reset_list` must precede `nffreeall`. `pkgbin.arch` points into this list everywhere. |
| Package model (pkg-hash, pkg, pkg-list/queue/array, depcon, pkg-spec, pkg-show, pkg-namevalue) | ~2,900 / ~1,650 | **L–XL** | **Entangled** | The structs *are* the API. src and dselect read and write fields directly (`clientdata` with a different type per program; reverse-dependency lists; trigger lists; namenode flags). The graph is cyclic, uses intrusive multi-membership lists and embedded first instances, and lives in the arena. Lookups insert as a side effect. Replacing it means replacing most of `src/main` at the same time, or exposing a `#[repr(C)]` mirror with raw pointers (giving up most of Rust's guarantees). |
| deb822 parse/dump (parse, fields, parsehelp, dump) | 2,907 / 2,133 | M (tokenizer) / L (semantics) | Tokenizer is a seam; field layer is entangled | `parse_stanza` is self-contained, but errors must surface as `ohshit` text with "parsing file '%s' near line %d package '%s':\n " prefixes (parsehelp.c:39–55). Field handlers write through `offsetof` into C structs, intern into global hashes, link trigger lists, and `parse_error` (noreturn) mid-parse. Dumping is a pure function of the model, so a Rust dumper reading C structs is feasible (M). Exact field order and legacy fix-ups (D.1) must be preserved byte for byte. |
| pkg-format | 475 / ~330 | S–M | Seam (read-only over the model) | Reuses the `w_*` writers and `fieldinfos`. Must keep the escape, width and virtual-field semantics. |
| Status DB, journal, locks (dbmodify, dbdir) | ~680 / ~450 | M | **Entangled** | Global state machine. Lock release is a cleanup entry popped in strict LIFO order (dbmodify.c:310–319). Mutual recursion with triggers (A.2). Calls parse and dump. Fatal sections around journal writes. The on-disk protocol (D.2, D.7) is simple and well-defined, so a standalone Rust implementation of the *format* is easy. Embedding it in C dpkg is the hard part. |
| info DB (db-ctrl-*) | ~585 / ~390 | S–M | Mostly a seam | Path computation is pure (watch the static return buffer). The upgrade relies on a bombout-only cleanup for rollback. |
| Filesystem DB (fsys-*, pkg-files, db-fsys-*) | 1,864 / 1,081 | M–L | **Entangled** | The global 262k-bucket hash in the arena; per-node flags owned by `src/main/archives.c` and friends (54 `FNNF_*` uses); diversion cross-links; `files_list_valid` state matrix (dpkg-db.h:232–244). Reload-on-inode-change semantics. The file formats themselves are trivial. |
| Triggers | 1,574 / 1,045 | L | **Entangled** | Global state; function-pointer hooks overridden by `dpkg`; intrusive doubly-linked cross lists; the parser writes trigger state; mutual recursion with `modstatdb_note`; three on-disk files with different durability (D.5, D.7). |
| ar | 374 / ~250 | S | Clean seam | Small helpers; errors become `ohshit` text. |
| tarfn | 753 / ~500 | M | Partial seam | Returns codes and `dpkg_error`, but its callbacks (in `src/main/archives.c`) can `ohshit`. A Rust `tar_extractor` would be longjmp'd over while holding heap state (the symlink queue and names), which is undefined behaviour or a leak. It would need C callback shims that catch errors, or the callbacks must also be ported. Dialect coverage (D/E) and deferred symlinks are must-keeps. |
| compress + buffer | ~1,990 / ~1,400 | M | **Good seam** | Already process-isolated: it runs in forked children, and any error ends the child with exit 2. Rust crates exist for every format. Watch for: (1) concatenated-stream behaviour differs today between the library and CLI paths (verified, E); (2) byte-reproducible `.deb` output depends on the exact compressor implementation and settings: zlib vs miniz, xz MT block layout, zstd checksum and workers (inferred, unverified); (3) environment-variable scrubbing for the CLI fallbacks. |
| OS helpers (file, dir, fdio, atomic-file, path-remove, subproc, command, pager, treewalk, sysuser, execname, meminfo, term, progress, log) | ~4,170 / ~2,380 | S–M each | Mostly seams | Thin syscall wrappers that fit Rust well. Caveats: `subproc_fork` pushes an error context in the child; `file_lock` registers an unlock cleanup and requires a statically allocated fd variable (file.c:303–304); fcntl (not flock) lock semantics must be kept exactly for frontend interoperability; `sysuser` must keep bypassing NSS and honouring `DPKG_ROOT`; `log.c` keeps the status-fd list in the arena. Forking without exec (compression children, `subproc_fork` users in src) is only safe in Rust if the process has no other threads at that point. |
| options | 705 / 472 | S–M | Partial seam | The `cmdinfo` tables hold raw pointers to C globals and C callbacks; `badusage` `ohshit`s. The config-file grammar and precedence (F.7) are easy to reproduce. |

### L.1 Behaviours a reimplementation could get subtly wrong

1. Version ordering:
   - `~` before end of string;
   - letters before other punctuation;
   - non-ASCII bytes ordered according to the platform's `char` signedness;
   - leading zeros ignored;
   - `+1:`, `-0:` and `00:` epochs accepted by `strtol` while Rust's `u32::from_str` would reject `-0`;
   - revision split at the last `-`;
   - empty and NULL revision equal;
   - lax parsing (warnings) for status and available, strict parsing for `.deb` control files.
2. deb822 details: case-insensitive field names; canonical capitalisation and **fixed field order** on output;
   unknown fields after the known ones, in input order with original case; `^Z` handling; blank continuation
   lines being lax; trailing whitespace trimmed; a missing final newline being an error; and duplicates of
   unknown fields detected case-insensitively.
3. Name-table prefix matching: `Priority: optionalx` is a hard error, while `Priority: weird` is accepted as an
   "other" priority (verified).
4. Package names are lowercased by the database, while uppercase is valid input (verified).
5. Legacy status fix-ups in `pkg_parse_verify` (D.1). Dropping any of them changes what the next status write
   contains.
6. Multi-Arch slot selection and cross-grade reuse (parse.c:378–456), and implicit dependency architectures
   (fields.c:562–574).
7. Journal semantics: `%04d` names, checkpoint after 250 records, the same-length-name check, the 4 KiB
   pre-padded `tmp.i`, and fsync ordering (D.2, D.7). `status-old` is a hard link, not a copy. `available` is not
   synced.
8. Lock protocol: `lock-frontend` then `lock`, fcntl `F_WRLCK` over the whole file, `O_TRUNC|O_CREAT 0660`,
   non-blocking, with `DPKG_FRONTEND_LOCKED` skipping the frontend lock, and the "was locked by <exe> process
   with pid N" text. `triggers/Lock` uses a *blocking* lock.
9. The `Unincorp` grammar (lowercase-only package tokens, `-` meaning no awaiter) and its weaker durability.
10. Status-fd line formats (`status: <pkg>: <state>`; dpkg adds `processing:` and `status: … : error :` lines in
    src). These are consumed by frontends (inferred). The `dpkg.log` line format is also consumed by external
    tools (inferred).
11. Error and warning texts:
    - `prog: error: …`, `prog: warning: …`, `prog (subprocess): …`, `error while cleaning up`,
      `unrecoverable fatal error, aborting`, `internal error`;
    - the parse prefix `parsing file '%s' near line %d package '%s':\n `.
    The program autotests compare exact texts: `src/at/deb-fields.at:34–38` matches the parse-warning prefix,
    and grep finds 31 error and warning matches in `src/at/*.at`. Most strings are also translation keys in 44
    catalogs. Error text is printed at *unwind* time, which affects ordering relative to other output.
12. Exit codes: fatal errors exit with 2 (ehandle.c:117, 463); `internerr` aborts with SIGABRT and runs no
    cleanups.
13. tar: base-256 numbers, the GNU `L`/`K` lengths with or without NUL, ustar prefix joining, `'\0'` as a file,
    a trailing `/` making a directory, symlinks created last, end-of-archive detected by checksum failure, and
    rejection of PAX archives.
14. Decompression stopping after the first xz, zstd or bzip2 stream in the library path (verified). A Rust
    crate's default "multi-stream" decoder would *change* behaviour.
15. `.list` and `md5sums` parsing: a trailing `/` stripped, exactly two spaces after a 32-character hash, a
    missing final newline being fatal.
16. passwd and group resolution through file parsing (no NSS), and `DPKG_ROOT` fallback to the host file.
17. Colour `auto` keyed on stdout even for stderr messages; the pager environment variables and the `LESS`
    default.
18. Known latent bugs that a port would naturally "fix", changing behaviour only in edge cases:
    - `str_quote_meta` off-by-one (ASan-confirmed, test-only);
    - `meminfo` one-byte stack overflow on files of 4 KiB or more (ASan-confirmed);
    - `ohshit` without a context segfaults (confirmed);
    - `SUBPROC_RETERROR == SUBPROC_RETSIGNO`;
    - the non-antisymmetric package sort comparator;
    - the status-fd list kept in the arena.

### L.2 Evidence in favour of a port (enablers)

- Clean, documented on-disk formats. Each file format is small, line- or stanza-oriented, and parsed by hand
  with no generated code (no lex or yacc). That makes them easy to specify and fuzz.
- Many leaf modules are pure or nearly pure (version, varbuf/strvec/string, c-ctype, path, ar, deb-version,
  namevalue, pkg-format logic) and have good unit tests: 3950 planned assertions, all passing on this build.
- Compression and decompression already run in separate processes behind a narrow fd-in/fd-out interface, and
  mature Rust or `-sys` crates exist for all formats.
- Newer APIs already use the return-code style `struct dpkg_error` (tar, buffer, file_slurp, pkg_spec,
  pkg_format, parseversion), which maps well to `Result`.
- Single-threaded design with no signal handlers in lib. Global state is explicit and inventoried.
- The library is static-only and declared volatile with no external ABI promise (README.api, macros.h). Changing
  the interface is permitted by policy.

### L.3 Evidence against an easy port (obstacles)

- Non-local error exits are everywhere: about 343 raising sites (286 + 57 `parse_error`) in lib, every allocator, and JUMP recovery in
  src. Cleanup is coupled to a global LIFO stack whose pairing is implicit. This is incompatible with Rust
  frames in mixed C/Rust call stacks.
- The arena-allocated, cyclic, intrusively linked global object graph is used directly, field by field, by src
  and dselect. There is no encapsulating API to put an FFI seam behind.
- Significant behaviour is in quirks: legacy fix-ups, `strtol` and signed-char edge cases, case and prefix
  rules, field order, durability ordering, lock protocol, message texts with printf-format translations. Some
  of it is only covered by program-level tests, not unit tests (K gaps: modstatdb, parse/dump, triggers,
  compress).
- Function-pointer hook tables and callbacks cross the library/program boundary in both directions (trigger
  hooks, tar operations, cmdinfo, cleanups). Each crossing needs error-model translation.
- One consumer, dselect, is C++ and uses C++ wrappers in `varbuf.h` and longjmp across C++ frames. It must keep
  compiling against whatever replaces the headers.

---

## Things I could not verify

- The unsigned-`char` half of the version-ordering quirk (arm64, ppc64el, s390x). Only x86-64 was tested.
- Whether apt and other frontends depend on the exact lock-error texts, the status-fd strings or the `dpkg.log`
  format. The fcntl lock interoperability is inferred from the `DPKG_FRONTEND_LOCKED` design.
- The purpose of the 4 KiB `tmp.i` padding (presumed ENOSPC avoidance; no comment in the code).
- The status-fd arena use-after-free. It is inferred from code and is unreachable while `--command-fd` stays
  disabled; I did not reproduce it.
- Whether xz MT output and zlib-vs-other-deflate output differ in ways that break reproducible `.deb` builds.
- The practical impact of the non-antisymmetric `pkg_sorter_by_nonambig_name_arch`.
- The current stabilisation status of Rust C-variadic function definitions and the exact Rust rules for
  `longjmp` over Rust frames (from general knowledge, not checked here).
- Platform claims for `lib/compat` (which OSes lack which function) are inferred from configure checks, not
  tested.
- Whether callers rely only on the sign of `dpkg_version_compare` (not audited exhaustively in src).
