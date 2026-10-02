> **Reference notes.** Detailed working notes behind the chapters in the parent directory,
> produced by automated code analysis of dpkg 1.23.x (commit `27f661e21`) on 2026-10-02.
> Every non-trivial claim cites `path:line`; items marked "(unverified)" were inferred, not run.
> `<repo>` is the source tree, `<build>` an out-of-tree build of it, `<scratch>` a temporary
> directory that no longer exists. See [../README.md](../README.md) for how these notes were checked.

# 02 — The C programs and shell scripts in `src/`

Scope: `dpkg` (`src/main/` + `src/common/`), `dpkg-deb` (`src/deb/`), `dpkg-split`, `dpkg-query`,
`dpkg-divert`, `dpkg-statoverride`, `dpkg-trigger`, `dpkg-realpath`, `src/sh/`, `src/*.sh`,
`src/completion/bash/`, and the autotest suite `src/at/`.

Conventions

* All paths are relative to the repo root `<repo>` (HEAD `27f661e21`, version
  string of the build is `1.23.11-1-gd0dbb`). `file:N` / `file:N-M` are line numbers in that tree.
* "Verified (exp)" means I observed the behaviour by running the built binaries in
  `<build>/src/` against a throwaway `--root` inside the scratchpad
  (non-root, with `--force-not-root --force-script-chrootless --no-debsig`). Experiment log is
  in the Appendix. "(unverified)" marks inferences I did not confirm.
* Build facts that matter when reading numbers: the build tree is configured with
  `prefix=/usr/local`, so the compiled-in admindir is `/usr/local/var/lib/dpkg` and
  pkgconfdir `/usr/local/etc/dpkg` (`build/src/Makefile:454,500,503`); `WITH_LIBSELINUX 1`,
  `WITH_LIBZ`, `WITH_LIBLZMA`, `WITH_LIBZSTD`, `WITH_LIBBZ2` are on and
  `DEB_DEFAULT_COMPRESSOR COMPRESSOR_TYPE_XZ` (`build/config.h:29,517-529`).
* libdpkg is linked **statically** into every program (`build/lib/dpkg/.libs/libdpkg.a`;
  `ldd` of each binary shows only libmd/libc, plus libselinux for dpkg and dpkg-statoverride,
  plus zlib/lzma/zstd/bz2 for dpkg-deb only).

How numbers were obtained

* Raw LOC: `wc -l`. SLOC: a Python script stripping `/* */` comments and blank lines
  (shell: blank and `#` lines). Function lengths: a Python scan for column-0 `name(` …
  `{` … `}` definitions over `src/**/*.c` (385 functions found).
* libdpkg modules used per program: `nm -u` of each program's object files joined (C locale)
  against `nm --defined-only` of `libdpkg.a` (78 libdpkg modules).
* CLI surface: a Python parser over each `cmdinfos[]` table, cross-checked with `--help`.
* Test counts: `src/at/testsuite --list` (80 groups) and the previous run's
  `build/src/at/testsuite.log` ("All 80 tests were successful"; that log is from a
  `testsuite -C at -j32` run dated 2026-10-02 14:06 that I did not start — probably a parallel
  analyst; I only ran the read-only `--list`).

---------------------------------------------------------------------------------------------

## A. Program inventory

LOC = raw / SLOC. "libdpkg" column lists the modules with the most distinct symbols used
(number of distinct libdpkg symbols / number of libdpkg modules touched in parentheses).

| Program | Purpose | Sources | LOC (raw/SLOC) | Main libdpkg subsystems used (syms/modules) | External programs exec'd |
|---|---|---|---|---|---|
| `dpkg` | Install/unpack/configure/remove/purge/trigger packages; DB queries that need write logic (audit, selections, avail, verify, arch list, version compare); forwards query/archive actions to `dpkg-query`/`dpkg-deb` | `src/main/*.c,h` (23 files) + `src/common/{force.c,selinux.c,*.h}` | 12 842 / 9 227 (main 12 083/8 678; common 759/549) | varbuf, ehandle, triglib, options, dbmodify, pkg, command, arch, pkg-show, treewalk, pkg-spec, fsys-iter, subproc, path-remove, log, depcon, cleanup, pager, tarfn, db-fsys-*, db-ctrl-* (221 / 58) | `dpkg-split -Qao` (`src/main/unpack.c:108`), `debsig-verify -q` (`unpack.c:150`, only if in PATH and not `--no-debsig`), `dpkg-deb --control` (`unpack.c:1329`), `dpkg-deb --fsys-tarfile` (`unpack.c:1631`), maintainer scripts (`script.c:216`), hooks via `system()` (`main.c:622`), status loggers via `sh -c` (`main.c:660`), `diff -Nu` + pager (`configure.c:207-219`), `$SHELL -i` at conffile prompt (`configure.c:254`), `rm -rf` (libdpkg `path_remove_tree`, `lib/dpkg/path-remove.c:157`), and *exec-replaces itself* with `dpkg-query`/`dpkg-deb` for forwarded actions (`main.c:853-868`) |
| `dpkg-deb` | Build, inspect, extract `.deb` archives | `src/deb/{main,build,extract,info}.c, dpkg-deb.h` | 2 225 / 1 671 | ar, treewalk, compress, options, pkg-format, command, subproc, deb-version (83 / 23) | GNU `tar -cf` for both members at build (`build.c:552-574`); GNU `tar -x/-t/-xv/-tv` for any listing/extraction (`extract.c:350-396`); compressors only if a compression lib was not linked (`lib/dpkg/compress.c:59-88`, not the case in this build) |
| `dpkg-split` | Split a .deb into parts / join / auto-accumulate parts in a depot queue | `src/split/*.c,h` (6 files) | 1 477 / 1 094 | ar, options, parse, nfmalloc, buffer, subproc, deb-version (60 / 20) | `dpkg-deb --info <deb> control` to read the control file when splitting (`split/split.c:75`) |
| `dpkg-query` | Read-only DB queries: `-s -W -l -L -S -p`, `--control-{path,list,show}` | `src/query/main.c` + `common/actions.h` | 1 095 / 830 | options, pkg-show, pkg-spec, pkg-format, pkg-array, fsys-hash/iter, pager, strwide, term, db-fsys-files/divert (79 / 33) | pager (`query/main.c:316`), `file_show` pager for `--control-show` (`:841`) |
| `dpkg-divert` | Manage file diversions DB (+ optional rename of the file) | `src/divert/main.c` | 947 / 715 | atomic-file, fsys-hash/iter, pkg-hash, glob, dbmodify, db-fsys-files/divert (60 / 26) | none |
| `dpkg-statoverride` | Manage stat-override DB (+ optional `--update` chown/chmod) | `src/statoverride/main.c` + `common/{force.c,selinux.c}` | 1 229 / 913 incl. shared common code (main.c alone 470) | atomic-file, db-fsys-override, sysuser, glob, fsys-hash (55 / 20) | none |
| `dpkg-trigger` | Record a trigger activation in `triggers/Unincorp` from a maintainer script; `--check-supported` | `src/trigger/main.c` | 301 / 230 | trigdeferred, pkg-spec, trigname (29 / 15) | none |
| `dpkg-realpath` | Resolve a pathname relative to `--root`/`--instdir` (symlink-following, root-clamped) | `src/realpath/main.c` | 274 / 193 | varbuf, file (31 / 12) | none |
| `dpkg-maintscript-helper` | Shell helper for maintainer scripts: `rm_conffile`, `mv_conffile`, `symlink_to_dir`, `dir_to_symlink`, `supports` | `src/dpkg-maintscript-helper.sh` | 685 / 533 | (shell) sources `sh/dpkg-error.sh` | `dpkg --compare-versions`, `dpkg --validate-version`, `dpkg-query -W/-L`, `dpkg-realpath`, `md5sum`, `sed`, `mv`, `rm`, `find`, `xargs`, `readlink`, `realpath`, `grep`, itself recursively (`_internal_pkg_must_own_file`) |
| `dpkg-db-backup` | Daily rotation of `arch status diversions statoverride` + `alternatives` tarball into `/var/backups` | `src/dpkg-db-backup.sh` | 84 / 50 | – | `tar`, `savelog`, `cmp`, `cp`, `touch` |
| `dpkg-db-keeper` | Optional post-invoke hook that commits the admindir into git | `src/dpkg-db-keeper.sh` | 56 / 18 | – | `git` |
| `sh/dpkg-error.sh` | Shell reporting library (colors, `error/warning/notice/info/hint/badusage/debug`) | `src/sh/dpkg-error.sh` | 136 / 86 | – | `basename` |
| bash completion | Completion for `dpkg`, `dpkg-deb`, `dpkg-query` | `src/completion/bash/*` | 438 / 287 | – | reads `/var/lib/dpkg/status` with awk; `dpkg --get-selections` |

Totals: 44 C/H files in `src/*/`, 19 553 raw / 14 279 SLOC; 1 499 raw lines of shell (incl.
completion). The installed set is declared in `src/Makefile.am:25-43` (8 `bin_PROGRAMS`,
`dpkg-maintscript-helper` as `bin_SCRIPTS`, `dpkg-db-backup`/`dpkg-db-keeper` as
`pkglibexec_SCRIPTS`, `sh/dpkg-error.sh` as data). Shell scripts get build-time substitution of
`version=`, `PKGDATADIR_DEFAULT=`, `ADMINDIR=`, `BACKUPSDIR=`, `TAR=` (`build-aux/subst.am:7-13`;
verified by diffing `build/src/dpkg-maintscript-helper` vs source).

Longest functions in `src/` (lines incl. braces):

| Lines | Function | Location |
|---|---|---|
| 551 | `process_archive` | `src/main/unpack.c:1271-1821` |
| 492 | `tarobject` | `src/main/archives.c:739-1230` |
| 443 | `depisok` | `src/main/depcon.c:320-762` |
| 311 | `extracthalf` | `src/deb/extract.c:94-404` |
| 309 | `usage` (dpkg help text) | `src/main/main.c:82-390` |
| 219 | `cmpversions` (mostly a table) | `src/main/enquiry.c:662-880` |
| 188 | `do_build` | `src/deb/build.c:619-806` |
| 181 | `deppossi_ok_found` | `src/main/packages.c:408-588` |
| 177 | `pkg_remove_old_files` | `src/main/unpack.c:724-900` |
| 170 | `removal_bulk_remove_configfiles` | `src/main/remove.c:512-681` |
| 165 | `dependencies_ok` | `src/main/packages.c:699-863` |
| 158 | `process_queue` / `deferred_configure_conffile` | `packages.c:183-340` / `configure.c:380-537` |

Within dpkg, the package-mutating core (archives/unpack/configure/remove/packages/depcon/
trigproc/script/cleanup/errors/help/filters/…) is 6 655 SLOC; the remaining actions
(`main.c`, `enquiry.c`, `select.c`, `update.c`, `verify.c`) are 2 023 SLOC.

---------------------------------------------------------------------------------------------

## B. dpkg main program, in depth

### B.1 Startup and command dispatch (`src/main/main.c`)

`main()` (`main.c:993-1051`):

1. `dpkg_locales_init`, `dpkg_program_init("dpkg")` — the latter sets progname, debug, pushes
   the outermost error context (handler `catch_fatal_error` → `exit(2)`), and **sets
   `umask(022)`** (`lib/dpkg/program.c:39-50`).
2. `set_force_default(FORCE_ALL)` (`main.c:1001`) — built-in defaults or `DPKG_FORCE` (B.2).
3. `dpkg_options_load(DPKG, cmdinfos)` (`main.c:1002`) — config files (B.2). Only `dpkg` loads
   config files; no other program in `src/` calls `dpkg_options_load` (grep).
4. `dpkg_options_parse(&argv, …)` — command line.
5. If root without `--force-not-root` and gid≠0 → `setgid(0)` (`main.c:1010-1012`).
6. No action → `badusage` (exit 2).
7. **Always exports** `DPKG_ADMINDIR`, `DPKG_ROOT`, `DPKG_FORCE` (comma list from
   `get_force_string()`) into the environment of every subprocess (`main.c:1017-1025`).
8. `f_triggers` default: `--triggers-only <pkgs>` ⇒ `-1` (no consequential trigger
   processing), else `1` (`main.c:1027-1032`).
9. Pre-invoke hooks + status loggers for "invoke actions" (B.2/C), run the action, post-invoke
   hooks, return `reportbroken_retexitstatus(ret)` (`main.c:1034-1050`; `errors.c:108-127`).

The action table `cmdinfos[]` (`main.c:764-851`) mixes actions and options. 44 action entries:

* 30 handled in-process: `-i/--install`, `--unpack`, `-A/--record-avail` → `archivefiles`;
  `--configure`, `-r/--remove`, `-P/--purge`, `--triggers-only` → `packages`; `-V/--verify`;
  `--get/--set/--clear-selections`; `--update/--merge/--clear-avail` → `updateavailable`;
  `--forget-old-unavail` (obsolete no-op warning, `update.c:125-135`); `-C/--audit`;
  `--yet-to-unpack` (deprecated); `--assert-<feature>` (an `ACTION_MUX`, 7 features
  `enquiry.c:300-336`); `--add/--remove/--print-architecture`, `--print-foreign-architectures`;
  `--predep-package`; `--validate-{pkgname,trigname,archname,version}`; `--compare-versions`;
  `-?/--help`, `--version`.
* 14 forwarded by **exec-replacing dpkg** (`execbackend`, `main.c:853-868`, builds
  `<backend> --<action> -- <args>`): `-L -s -p -l -S` → `dpkg-query`; `-b -c -e -I -f -x -X
  --ctrl-tarfile --fsys-tarfile` → `dpkg-deb`. Because `main()` has already exported
  `DPKG_ADMINDIR/DPKG_ROOT/DPKG_FORCE`, the backend sees `--admindir/--root` via env.
  `--no-pager` additionally sets `DPKG_PAGER=cat` for backends (`main.c:470-477`).
  Verified (exp): `dpkg --root=… -l 'foo*'` exits 0 via dpkg-query.
* Dead code: `commandfd()` (`main.c:870-991`, 121 lines) is compiled (symbol `T commandfd` in
  the binary) but its table entry is commented out (`main.c:801-803`); `dpkg --command-fd`
  → "unknown option" (verified).

### B.2 Option parsing, config files, `--force-*`

Parser (`lib/dpkg/options.c:235-337`) is dpkg's own, not getopt:

* Stops at the first argument not starting with `-` (`options.c:249`) — **no GNU-style option
  permutation**. Verified (exp): `dpkg-split -a -o x.deb part.deb --admindir=X` treats
  `--admindir=X` as a 2nd positional and fails with exit 2.
* `--` ends options (`:251`); long `--opt=value` or `--opt value`; `takesvalue==2` entries take
  the suffix form `--force-<thing>` / `--refuse-<thing>` / `--no-force-<thing>`
  (`options.c:264-268`); short options can be bundled (`-Dhelp`, `-iE`, …).
* Actions are just options whose callback is `setaction`; two actions → "conflicting actions"
  badusage (`options.c:464-475`).

Config files (`options.c:70-232`): loaded in this order, each applied as if typed before the
command line: (1) every file in `<pkgconfdir>/dpkg.cfg.d/` whose name matches
`[0-9A-Za-z_-]+` (no dot, `options.c:164-180`), in `alphasort` order; (2) `<pkgconfdir>/dpkg.cfg`;
(3) `$HOME/.dpkg.cfg`. Each line is `optname [=] value`, `#` comments, quotes stripped,
`force-<thing>` style accepted (`:115-131`). An unknown option is a **fatal** error
(`config_error` → `ohshit`, `:134-135`). Because actions live in the same table, a config file
can set an action: verified (exp) — `print-architecture` in `~/.dpkg.cfg` makes bare `dpkg`
print `amd64`, and then `dpkg --print-foreign-architectures` fails with "conflicting actions".
The Debian packaging ships `debian/dpkg.cfg` with `no-debsig` and `log /var/log/dpkg.log`;
**there is no compiled-in default log file** — `log_message` is a no-op unless `--log` was
given (`lib/dpkg/log.c:57-58`; only caller of `log_set_file` is `main.c:543`).

Force system (`src/common/force.c`, `force.h`):

* 28 flags as bits 0–27 (`force.h:30-60`) + `all` (`0xffffffff`); 29 names in `forceinfos[]`
  (`force.c:64-221`) with type `[*]` default-on (`security-mac`, `downgrade`), `[!]` "damage"
  (overwrite-dir, unsafe-io, script-chrootless, conf*, architecture, breaks, conflicts, depends,
  depends-version, remove-reinstreq/-protected/-essential, all), or plain.
* `set_force_default(mask)` (`force.c:347-367`): if `DPKG_FORCE` is set in the environment,
  built-in defaults are **not** applied and the env string is parsed instead (empty ⇒ all off).
  Since dpkg exports `DPKG_FORCE` (B.1), the force set propagates to every subprocess, including
  `dpkg-statoverride` called from maintainer scripts (its mask only limits what `--force-help`
  shows, `force.c:296-299`; parsing accepts all names).
* `parse_force` (`force.c:282-345`): comma list; unknown name → badusage; `help` prints table
  and `exit(0)`. `-G` is a short alias for `--refuse-downgrade` (`main.c:822-823`).
* `forcibleerr(flag, …)` (`force.c:383-396`): warning if forced, else `ohshit`. 11 call sites,
  all in `src/main/`. `forcible_nonroot_error(rc)` swallows EPERM under `--force-not-root`
  (`force.c:398-405`) — used for every chown/chmod during unpack.
* Other dpkg options worth noting: `--abort-after N` (default 50, `main.c:411`), `--ignore-depends`
  list, `--path-exclude/--path-include` (B.10), `--pre-invoke/--post-invoke/--status-logger`
  (ordered lists, `main.c:582-608`), `--status-fd N` (repeatable), `--log`, `--no-act/--dry-run/
  --simulate`, `-D/--debug=<octal>` with 13 flags (`main.c:417-436`; `-Dhelp` exits 0).

### B.3 Package state machine (as driven by the programs)

States (`help.c:43-52`): `not-installed` < `config-files` < `half-installed` < `unpacked` <
`half-configured` < `triggers-awaited` < `triggers-pending` < `installed`; plus the
`reinstreq` error flag (`PKG_EFLAG_REINSTREQ`) and the selection `want`
(unknown/install/hold/deinstall/purge). Every state change goes through
`pkg_set_status` + `modstatdb_note` which journals the record to `updates/NNNN`
(libdpkg) **inside a fatal-errors section** (`lib/dpkg/dbmodify.c:515-553`), emits
`status: <pkg>: <state>` to status-fds and `status <state> <pkg> <ver>` to the log.

Per-run transient state lives in `pkg->clientdata` (`struct perpackagestate`, `main.h:57-70`;
allocated by `ensure_package_clientdata`, `perpkgstate.c:31-44`): `istobe`
(NORMAL/REMOVE/INSTALLNEW/DECONFIGURE/PREINSTALL), cycle `color`, `enqueued`,
`replacingfilesandsaid`, `cmdline_seen`, `trigprocdeferred`.

Verified transitions (exp, status-fd output):

* fresh install: `half-installed → unpacked → half-configured → installed`.
* upgrade: `half-configured` (old prerm) `→ unpacked → half-installed → unpacked → half-configured → installed`.
* remove: `half-configured` (prerm) `→ half-installed → config-files`; purge → `not-installed`.

### B.4 Install/unpack: `archivefiles` and `process_archive`

`archivefiles` (`archives.c:1676-1822`):

1. `trigproc_install_hooks()`; `modstatdb_open` readonly for `--no-act`,
   `readonly|available_write` for `--record-avail`, `write` with `--force-not-root`, else
   `needsuperuser` (takes the locks; `archives.c:1686-1695`).
2. `checkpath()` — warns/forcible error if `sh rm tar diff dpkg-deb ldconfig
   start-stop-daemon` are missing from PATH (`help.c:83-124`). dpkg itself never execs
   `ldconfig`/`start-stop-daemon`/`tar`; the check is a sanity contract.
3. `pkg_infodb_upgrade()`, log `startup archives <action>`.
4. `-R/--recursive`: treewalk following symlinks, collect regular files ending `.deb`
   (`:1702-1739`). All args must `stat` as regular files (`:1747-1759`).
5. `ensure_diversions()`, `ensure_statoverrides()` once (`:1773-1774`).
6. **Per archive**: `setjmp` + `push_error_context_jump(…, print_error_perarchive, name)`;
   `dpkg_selabel_load()`; `process_archive()`; flush stdout/stderr inside a fatal section;
   `pop_error_context(ehflag_normaltidy)` (`:1776-1794`). An error longjmps back, runs the
   context's cleanups with `ehflag_bombout`, and continues with the next archive unless
   `abort_processing` (B.10).
7. For `--install` (and the fall-through cases) `process_queue()` configures everything that
   was unpacked (`:1804-1816`), then `trigproc_run_deferred()`, `modstatdb_shutdown()`.
   So `-i a.deb b.deb` unpacks all archives first, then configures (verified: two "Unpacking"
   lines precede one "Setting up" when the same package is given twice).

`process_archive` (`unpack.c:1271-1821`) phase by phase:

| # | Lines | Phase |
|---|---|---|
| 1 | 1304-1309 | `clear_cleanup_state()`; stat archive |
| 2 | 1312-1315 | if `f_act`: `deb_reassemble` — fork/exec `dpkg-split -Qao <admindir>/reassemble.deb <file>` for **every** archive; exit 0+file ⇒ use reassembled file; exit 0 no file ⇒ part filed, skip; exit 1 ⇒ not a part (`unpack.c:91-134`). Pushes `cu_pathname` (never popped; removed at end of the archive's context) |
| 3 | 1318-1319 | `deb_verify`: if `debsig-verify` is in PATH and not `--no-debsig`, run it; failure is `--force-bad-verify`-able (`unpack.c:136-169`) |
| 4 | 1322-1333 | control dir `<admindir>/tmp.ci/` (or `mkdtemp` under TMPDIR for `--no-act`, `unpack.c:173-213`), push `cu_cidir`; fork/exec `dpkg-deb --control <deb> <cidir>` |
| 5 | 1341 | `dir_sync_contents(cidir)` — fsync every extracted control file and the dir (comment: neither dpkg-deb nor tar fsync) |
| 6 | 1343-1365 | `parsedb(cidir/control, pdb_parse_binary|pdb_ignore_archives[|lax])` → `pkg`; record archive size |
| 7 | 1367-1373 | `--record-avail` stops here ("Recorded info about …") |
| 8 | 1375-1381 | architecture must be all/native/foreign, else `--force-architecture` |
| 9 | 1383-1391 | `clear_deconfigure_queue`, `clear_istobes`; `wanttoinstall()` (`archives.c:1832-1881`: selection, `--skip-same-version`, downgrade under `--force-downgrade` [default on]) |
| 10 | 1393-1405 | other installed Multi-Arch instances of the set at a different version are queued for deconfiguration (`reason = PKG_WANT_UNKNOWN`) |
| 11 | 1407 | `pkg_check_depcon` (`unpack.c:216-287`): sets `istobe=INSTALLNEW`; walks Conflicts (`check_conflict`), Breaks (`check_breaks`), Provides-vs-installed-Conflicts, Pre-Depends (`depisok(…, allowunconfigd=true)`; may run `trigproc(…, TRIGPROC_REQUIRED)` on a triggers-awaited dependee), and reverse Conflicts |
| 12 | 1409-1425 | load all file lists, init fs hash and file-trigger interests; print "Preparing to unpack"; `log_action install|upgrade` (+ `processing:` status line); `--no-act` stops here |
| 13 | 1431-1434 | activate triggers: package processing + `triggers` control file `activate*` directives |
| 14 | 1437-1449 | push `cu_fileslist`; `deb_parse_conffiles` (`unpack.c:360-485`: flags `remove-on-upgrade`, copies old hash from owning package); mark old conffiles of pkg and of every conflictor (`FNNF_OLD_CONFF`) |
| 15 | 1451-1474 | if old status ≥ half-configured: set half-configured, push `cu_prermupgrade` (if > half-configured), run **old prerm `upgrade NEW`**, fallback **new prerm `failed-upgrade OLD NEW`** (`maintscript_run_old_or_new`), set unpacked |
| 16 | 1476 | `pkg_deconfigure_others` (`unpack.c:290-354`): for each queued deconfiguration: activate triggers, set half-configured, push fallback pair `cu_prermdeconfigure`/`ok_prermdeconfigure`, run **prerm `deconfigure in-favour PKG VER [removing CONFL VER]`** |
| 17 | 1478-1504 | each conflictor being removed that is configured-ish: half-configured, push `cu_prerminfavour`, run **prerm `remove in-favour PKG VER`**, set half-installed |
| 18 | 1506-1535 | set `reinstreq`, status half-installed; push exactly one of `cu_preinstverynew` (not-installed) / `cu_preinstnew` (config-files) / `cu_preinstupgrade`; run **new preinst** `install` / `install OLD NEW` / `upgrade OLD NEW` |
| 19 | 1537-1547 | "Unpacking X (V) [over (OLD)] …" |
| 20 | 1624-1660 | `m_pipe`, push `cu_closepipe`; fork/exec `dpkg-deb --fsys-tarfile <deb>` writing into the pipe; push second `cu_fileslist`; `tar_extractor()` (libdpkg `tarfn.c`) with callbacks all = `tarobject` (B.5); drain trailing zeros; reap child with `SUBPROC_NOPIPE` |
| 21 | 1662 | `tar_deferred_extract` (B.5: batched fsync + rename) |
| 22 | 1664-1675 | if old status was half-installed/unpacked (always true for upgrades by now): half-installed, push `cu_postrmupgrade`, run **old postrm `upgrade NEW`**, fallback **new postrm `failed-upgrade OLD NEW`** |
| 23 | 1691 | **`push_checkpoint(~ehflag_bombout, ehflag_normaltidy)`** — point of no return (H) |
| 24 | 1696 | `pkg_remove_old_files` (`unpack.c:723-900`): first `remove-on-upgrade` conffiles (`pkg_remove_conffile_on_upgrade`: unmodified → unlink, modified → `.dpkg-old`), then old files not in the new archive in **reverse** path order; directories only if not used by others; dev/ino comparison to avoid deleting a file that is the same inode as a new file under another name; vanished old conffiles become *obsolete* conffiles; removal uses `secure_unlink_statted` |
| 25 | 1700 | write new `info/<pkg>.list` |
| 26 | 1705-1709 | trigger interests: delete old, add new, save `triggers/File` |
| 27 | 1717 | `pkg_infodb_update` (`unpack.c:526-630`): rename each new control file from tmp.ci into `info/<pkg>[:arch].<name>`; old ones without new versions unlinked; rejects subdirectories and names >100 chars; `control` skipped; layout switch for M-A:same; `dir_sync_path(info)` |
| 28 | 1720 | write `info/<pkg>.md5sums` from the digests **computed during extraction** (the package's shipped md5sums is overwritten) |
| 29 | 1734 | `pkg_update_fields`: copy available→installed (deps rebuilt with reverse links via `copy_dependency_links`), new conffiles list with obsolete/remove-on-upgrade flags |
| 30 | 1741-1755 | crossgrade fix-up of duplicate pkginfo for same arch |
| 31 | 1766 | `pkg_disappear_others` (`unpack.c:1028-1138`): any other package all of whose files are now owned by the new one (and not needed by a Depends/Pre-Depends/Recommends) "disappears": **postrm `disappear PKG VER`**, info files deleted, status not-installed |
| 32 | 1775 | `pkg_remove_files_from_others`: rewrite other packages' file lists without the files we took over; mark their conffiles obsolete |
| 33 | 1781-1782 | status **unpacked** |
| 34 | 1793 | remove `.dpkg-tmp` backups (`path_remove_tree`) |
| 35 | 1797-1799 | clear `reinstreq`; second checkpoint |
| 36 | 1805-1817 | `removal_bulk(conflictor)` for each conflictor ("Removing X, to allow configuration of Y") — **postrm `remove`** etc. (B.8) |
| 37 | 1819-1820 | for `--install`, `enqueue_package_mark_seen(pkg)` |

Maintainer-script call sequences (verified (exp) unless noted; "NEW/OLD" are versions):

| Scenario | Sequence |
|---|---|
| fresh install | new `preinst install` → unpack → new `postinst configure ""` (an explicit empty 2nd argument; `configure.c:682-686`) |
| reinstall from config-files | new `preinst install OLD NEW` (code `unpack.c:1518-1525`, not exercised) |
| upgrade | old `prerm upgrade NEW` → new `preinst upgrade OLD NEW` → unpack → old `postrm upgrade NEW` → configure: new `postinst configure OLD` |
| same-version reinstall | old `prerm upgrade V`… new `preinst upgrade V V` → old `postrm upgrade V` (verified with the same .deb twice) |
| old prerm fails | new `prerm failed-upgrade OLD NEW`; if that succeeds, continue ("… it looks like that went OK") |
| old postrm fails | new `postrm failed-upgrade OLD NEW` |
| new preinst fails (upgrade) | new `postrm abort-upgrade OLD NEW` → old `postinst abort-upgrade NEW`; final state `ii` OLD |
| error during unpack (fresh install, e.g. file conflict) | files rolled back → new `postrm abort-install`; package back to not-installed; exit 1 |
| old postrm **and** new postrm failed-upgrade fail | old `preinst abort-upgrade NEW` → files rolled back → new `postrm abort-upgrade OLD NEW` (if this fails too, old `postinst abort-upgrade` is **skipped**); final state `iHR` (half-installed, reinst-required) |
| conflicts+replaces (baz C/R foo) | foo `prerm remove in-favour baz 1.0` → baz `preinst install` → unpack → foo `postrm remove` → baz `postinst configure ""` |
| breaks with `-B` | foo `prerm deconfigure in-favour qux 1.0` → qux `preinst install` → … → qux `postinst configure`; then foo is re-queued for configure (`ok_prermdeconfigure`, `cleanup.c:158-165`) and fails "dependency problems - leaving unconfigured" (exit 1) |
| breaks without `-B` | fails before any script: "installing qux would break foo, and deconfiguration is not permitted" |
| disappear | rep `preinst install` → "Replacing files in old package small" → small `postrm disappear rep 1.0` → rep `postinst configure` |
| deconfigure rollback / conflictor rollback | `postinst abort-deconfigure in-favour PKG VER [removing CONFL VER]` / `postinst abort-remove in-favour PKG VER` (`cleanup.c:171-222`; not exercised) |

Notes:

* The error message text calls a post-unpack postinst "old … postinst" even though it is the
  new version's script, because after phase 27 the "installed" infodb holds the new scripts
  (verified, scenario D). Output text is a compat surface only for humans/log scrapers.
* Conflict detection with existing files is lazy: it happens per tar member inside
  `tarobject`, i.e. **after** the new preinst ran (verified, scenario E).

### B.5 File-by-file extraction: `tarobject` (`archives.c:739-1230`)

Called by libdpkg's `tar_extractor` for every member. Important: `tar_extractor` collects all
**symlinks** and creates them only after every other member (`lib/dpkg/tarfn.c:540-551,598-605`),
so an archive cannot create a symlink and then write through it.

Per member:

1. `tar_entry_update_from_system` maps uname/gname to local uid/gid (`tarfn.c:443-459`).
   Reject names containing `\n` (`:760-762`). Only sanity check on the name — see G.6 for `..`.
2. `fsys_hash_find_node(name)` normalises leading `/` and `./` (`lib/dpkg/fsys-hash.c`); fail if
   the node is a `remove-on-upgrade` conffile actually shipped (`:766-768`). Append to the
   new-files queue (allocated from an obstack, `archives.c:78-144`), set `FNNF_NEW_INARCHIVE`.
3. Diversion check: writing the "diverted-to" name of somebody else's diversion requires
   `--force-overwrite-diverted` (`:789-806`).
4. Stat override replaces the tar uid/gid/mode (`:808-813`). `namenodetouse` applies the
   diversion (write to the `useinstead` path unless we are the diverting package,
   `help.c:55-78`). `trig_file_activate` fires file triggers (`:818`). Conffiles are
   dereferenced through symlinks with `conffderef` (≤25 hops, `configure.c:704-797`).
5. `setupfnamevbs` builds `<instdir><name>`, `….dpkg-tmp`, `….dpkg-new` (`:671-687`).
   `lstat` the target; if absent, try `rename(.dpkg-tmp → name)` to recover from an
   interrupted earlier run (with EROFS special-casing) (`:831-877`).
6. Existing-directory handling (`:882-914`): a symlink member whose target is an existing dir,
   or a symlink to the same dir (`linktosameexistingdir`, `:689-735`), is a no-op; a dir member
   over an existing dir (following symlinks, `stat`) is a no-op. Consequence: **dpkg never
   replaces a directory by a symlink nor a symlink-to-dir by a directory** — that is what
   `dpkg-maintscript-helper symlink_to_dir/dir_to_symlink` exist for (F.1).
7. Ownership conflict loop over other owners of the path (`:916-1053`): skip same-set
   M-A:same instances (enables refcounting if the set is "getting in sync", i.e. all instances
   at the new version, `unpack.c:1146-1165`); skip if a diversion involves either package; skip
   if a new dir over a nonexistent path; Replaces bookkeeping (`replacingfilesandsaid` 1 = we
   replace them → "Replacing files in old package"; 2 = they replace us → keep existing file,
   skip member); skip config-files packages and packages being removed; take over obsolete
   conffiles; else `forcibleerr(FORCE_OVERWRITE_DIR | FORCE_OVERWRITE)`.
8. `--path-exclude` filter (`filter_should_skip`) marks `FNNF_FILTERED` and skips (`:1064-1070`).
9. Refcounted M-A:same files: compute the on-disk hash and, unless `--force-overwrite`, only hash
   the member without extracting; then `tarobject_matches` requires identical content/type
   (`:602-669`, `:1077-1126`).
10. Otherwise: remove stale `.dpkg-new`/`.dpkg-tmp`, **push `cu_installnew(namenode)`**
    (`:1106`), and extract to `.dpkg-new` (`tarobject_extract`, `:381-497`): regular files are
    created `O_CREAT|O_EXCL|O_WRONLY` **mode 0**, preallocated with `fallocate` if ≥16 KiB
    (`lib/dpkg/fdio.c`), data copied while computing MD5, `sync_file_range(WRITE)` hint,
    `fchown`/`fchmod` (EPERM tolerated under `--force-not-root`), `FNNF_DEFERRED_FSYNC` unless
    `--force-unsafe-io`; FIFOs `mkfifo(…, 0)`, devices `mknod`, hardlinks `link()` to the
    target's `.dpkg-new` if that one is still pending rename, symlinks `symlink()`, dirs
    `mkdir(…, 0)`.
11. `chown/chmod` (non-regular), `utimensat` (mtime from tar, atime = start time), SELinux
    `lsetfilecon_raw` with the context looked up for the final path (`:1128-1130`).
12. Conffiles stop here: they stay as `.dpkg-new` for `--configure` (`:1143-1148`).
13. Backup the old object as `.dpkg-tmp`: rename if either side is a directory
    (`FNNF_NO_ATOMIC_OVERWRITE`), recreate a copy for symlinks, `link()` for everything else
    (`:1152-1197`).
14. Regular files, hardlinks and symlinks get `FNNF_DEFERRED_RENAME`; directories, devices and
    FIFOs are renamed into place immediately (`:1204-1227`).

`tar_deferred_extract` (`archives.c:1268-1328`), after the whole tar stream: first
`tar_writeback_barrier` issues `sync_file_range(WAIT_BEFORE)` on every deferred file
(`:1232-1260`), then for each `DEFERRED_RENAME` node: open, `fsync`, close, `rename(.dpkg-new →
name)`, mark `FNNF_PLACED_ON_DISK`. The design batches I/O: write all, wait all, then
fsync+rename. Verified (exp, strace): with a package of 1 regular file + 1 conffile, both get
`sync_file_range(WRITE)` and `(WAIT_BEFORE)`, but **only the regular file is fsynced**; the
conffile's `.dpkg-new` is never fsynced and its later rename in `--configure`
(`configure.c:522`) is not followed by an fsync either. No `fsync` of the parent directories of
installed files was observed (only admindir directories are dir-synced). `--force-unsafe-io`
removed exactly one `fsync` (37→36) and both `WAIT_BEFORE` calls in that test.

### B.6 Configure (`configure.c`)

`deferred_configure(pkg)` (`configure.c:544-691`):

1. Status must be unpacked or half-configured; all other instances of an M-A set must be ≥
   unpacked and at the same version (`:552-590`).
2. At `dependtry ≥ 2` try `findbreakcycle` (`:592-594`). `dependencies_ok` → DEFER re-enqueues
   with `istobe=INSTALLNEW` (`:596-603`). `trigproc_reset_cycle()`. `breakses_ok` → HALT ⇒
   "dependency problems - leaving unconfigured" (`:615-621`). `reinstreq` ⇒
   `--force-remove-reinstreq`.
3. "Setting up …", `log_action configure`, activate triggers; `--no-act` stops (marks
   installed in memory).
4. If unpacked: process every non-disappearing conffile with `deferred_configure_conffile`, then
   half-configured. Run **postinst `configure <configversion or "">`**; clear reinstreq and
   pending triggers; `post_postinst_tasks(INSTALLED)` picks installed / triggers-pending /
   triggers-awaited and incorporates triggers (`script.c:51-66`).

Conffile decision (`deferred_configure_conffile`, `configure.c:379-537`), hashes are MD5 hex
(`md5hash`, `:809-831`; `nonexistent` and `-` sentinels, `dpkg.h:59-61`):

| Condition (evaluated in order) | Action |
|---|---|
| no `<file>.dpkg-new` | sync hash from another M-A instance (`deferred_configure_ghost_conffile`), done |
| current == new-dist | `CFO_IDENTICAL` (= keep; delete `.dpkg-new`) |
| current missing **and** `--force-confmiss` | install new |
| recorded hash is `newconffile` (first install of this conffile): current missing → install; else prompt with "File on system created locally" (`useredited=distedited=1`, `CFOF_IS_NEW`) |
| otherwise `useredited = recorded≠current`, `distedited = recorded≠new` → table `conffoptcells[useredited][distedited]` (`:80-84`): (0,0) keep, (0,1) install, (1,0) keep, (1,1) prompt (default keep); `--force-confask` and user-edited ⇒ prompt-keep; current missing ⇒ add `CFOF_USER_DEL` |

Prompt (`show_prompt`, `:86-193`): `tcflush` stdin; prints to **stderr**; forced answers:
`--force-confnew` → y, `--force-confold` → n, `--force-confdef` uses the default when one
exists; else reads a line from stdin; EOF ⇒ error "end of file on <standard input> at conffile
prompt" (verified; package left `iU` with `.dpkg-new` in place). `D` runs `diff -Nu` through
the pager; `Z` spawns `$SHELL -i` (default `sh`) with `DPKG_SHELL_REASON=conffile-prompt`,
`DPKG_CONFFILE_OLD`, `DPKG_CONFFILE_NEW` (`:236-259`). The status-fd line `status: <file> :
conffile-prompt : '<old>' '<new>' <useredited> <distedited> ` is emitted **whenever the
decision includes PROMPT, even if a force option then answers it** (`:292-297`; verified with
`--force-confold`). Log line `conffile <file> install|keep` (`:311-312`).

Outcomes (`:479-528`): keep+backup ⇒ `.dpkg-new → .dpkg-dist` (verified with confold); keep ⇒
unlink `.dpkg-new`; install+backup ⇒ remove old `.dpkg-dist`/`.dpkg-old`, `link(file →
.dpkg-old)` (unless user-deleted), rename `.dpkg-new → file` (verified with confnew); install ⇒
rename. Permissions of the installed file are copied onto `.dpkg-new` first
(`file_copy_perms`, `:425-429`). Most failures here are warnings, only the final rename is fatal.

### B.7 Processing queue and dependency checking (`packages.c`, `depcon.c`)

`packages()` (`packages.c:145-180`) is the entry for `--configure/--remove/--purge/
--triggers-only`: open DB (needsuperuser), `checkpath`, log `startup packages <action>`,
enqueue `--pending` set (`enqueue_pending`, `:73-122`) or named packages (rejecting names
ending `.deb`; for `--configure` also `trigproc_populate_deferred`), `ensure_diversions`,
`process_queue`, `trigproc_run_deferred`.

`process_queue` (`packages.c:182-340`): FIFO `pkg_queue`; duplicates are dropped at enqueue
(`enqueued` flag). Each pop:

* Progress tracking: `sincenothing` counts pops without progress. If it exceeds
  `3·len+2` ⇒ `dependtry++`; else if it exceeds `2·len+2` and `dependtry ≥ 3` and some
  triggers-pending package `progress_bytrigproc` would unblock a dependency ⇒ process that
  package's triggers instead (counts as progress); else `dependtry++`. Reaching
  `DEPEND_TRY_LAST` (7) is an `internerr` (`:248-277`).
* `setjmp` + per-package error context (`print_error_perpackage`), dispatch: install/configure
  → `trigproc(TRY_QUEUED)` if triggers pending else `deferred_configure`; remove/purge →
  `deferred_remove`; triggers-only requires pending triggers (`:288-335`). On error `istobe` is
  reset to NORMAL and processing continues.

`dependtry` semantics (`main.h:197-240`; the older comment at `packages.c:344-377` is stale):
1 normal; 2 also break dependency cycles; 3 also start trigger processing of queued
packages; 4 also check trigger cycles when deferring; 5 version mismatches become warnings if
`--force-depends-version`; 6 everything allowed if `--force-depends`. The force tries act
through `found_forced_on()` (`packages.c:386-393`).

`dependencies_ok` (`packages.c:698-863`) checks Depends/Pre-Depends of the installed pkgbin:
per OR-group the best of `deppossi_ok_found` over real packages (arch-filtered iterator,
`depcon.c:42-99`) and Provides; result OK / DEFER (dependee is INSTALLNEW, or awaiting
triggers with a fixer, or `--force-configure-any` enqueues it) / FORCED / NONE;
`--ignore-depends` short-circuits. A package being removed whose deps would HALT is DEFERred
instead (`:852-854`). `breakses_ok` (`:669-692`) checks reverse Breaks against the package and
its Provides.

Cycle breaking (`depcon.c:101-250`): DFS with white/gray/black coloring from the package;
when a dependency edge reaches a package already on the current path, mark one
`deppossi->cyclebreak=true`, preferring an edge **out of a package without a postinst**
(null operation), else the edge where the cycle closes (`foundcyclebroken`, `:110-155`). A
`cyclebreak` edge counts as satisfied (`packages.c:740-745`).

`depisok` (`depcon.c:319-762`) is the unpack-time checker (also used by audit/predep): for
Depends-like types, satisfied by INSTALLNEW-at-right-version, installed/triggers-pending at
right version, or (with `allowunconfigd`, Pre-Depends) unpacked whose *configured* version also
satisfies; Provides of to-be-installed or installed packages; builds the human "whynot" text.
For Conflicts/Breaks it counts matching present/to-be-installed packages and their providers
and returns the single `canfixbyremove` candidate. Observation: `*canfixbytrigaw = …` at
`depcon.c:548` is not NULL-guarded, while the equivalent at `:446` is; callers pass NULL for
Depends-type checks (`archives.c:1581,1603`, `unpack.c:1092,1116`). A NULL dereference would
need an installed provider in triggers-awaited state — reachability (unverified).

### B.8 Remove / purge (`remove.c`)

`deferred_remove` (`remove.c:102-227`): sets want deinstall/purge (unless `--pending`);
not-installed ⇒ warning; config-files with `--remove` ⇒ warning; Essential / Protected ⇒
`--force-remove-essential/-protected`; reverse-dependency check over the package and its
Provides (`checkforremoval`, `:53-100`, cycle breaking at try ≥2) ⇒ DEFER/HALT;
`--force-remove-reinstreq`; `--no-act` prints "Would remove or purge". Then "Removing …",
`log_action remove`, if ≥ half-configured: half-configured, push `cu_prermremove`, run
**prerm `remove`**, set unpacked, `removal_bulk`.

`removal_bulk` (`remove.c:688-748`):

* `removal_bulk_remove_files` (`:285-409`): half-installed + **checkpoint**; reverse walk of the
  file list: skip files shared with another M-A instance; keep conffiles as leftovers; dirs only
  if no conffiles/children of ours/other owners; never `/.`; remove `.dpkg-tmp`/`.dpkg-new`,
  then `rmdir` or `secure_unlink` (setuid/setgid/sticky regular files and unknown types are
  `chmod 0600` first, `lib/dpkg/path-remove.c:36-54`); EBUSY/EPERM on dirs ⇒ "may be a mount
  point?" warning. Rewrite the list with leftovers, **postrm `remove`**, drop trigger
  interests, delete info files except `list`/`postrm`, set **config-files**, clear
  essential/protected, checkpoint.
* No postrm and no conffiles ⇒ treated as purge. Purge: `removal_bulk_remove_configfiles`
  (`:511-681`) drops conffiles that changed owner or are diverted, unlinks each conffile and its
  backups matching `~ .bak % .dpkg-tmp .dpkg-new .dpkg-old .dpkg-dist`, `name~N~`, and `#name#`
  in the same directory (`dpkg.h:56-57`, `:627-670`), runs **postrm `purge`**; then
  `removal_bulk_remove_leftover_dirs`, unlink `list` and `postrm`, status not-installed.

Verified: prerm failure ⇒ `postinst abort-remove`, package back to installed (`ri`); postrm
`remove` failure ⇒ no abort script (past the checkpoint), state half-installed (`rH`).

### B.9 Triggers (`trigproc.c`)

Spec: `doc/spec/triggers.txt` (818 lines). Program side:

* `trig_activate_packageprocessing` parses the package's `info/<pkg>.triggers` for
  `activate*` directives whenever the package is processed (`trigproc.c:179-187`).
* Deferred queue (`:92-174`): packages that got triggered are pushed by the libdpkg hook
  `enqueue_deferred` (skipped when `f_triggers < 0`, i.e. `--no-triggers`/`--triggers-only
  <pkgs>`). `trigproc_run_deferred()` runs at the end of `archivefiles` and `packages` with a
  per-package error context. `trigproc_populate_deferred` (for `--configure <pkgs>`) adds every
  triggers-pending/awaited package with want install/hold — the in-code comment says this
  exists because apt does not call a final `--configure --pending` (`:108-122`).
* `trigproc(pkg, type)` (`:374-486`): if not yet at try 3 and called from the queue ⇒ postpone;
  cycle breaking at try ≥2; `dependencies_ok`: DEFER ⇒ (at try ≥4 check trigger cycle) and
  re-enqueue; HALT ⇒ silently skipped for opportunistic deferred processing (and the cycle
  tracker reset), error otherwise; then `check_trigger_cycle`, "Processing triggers for …",
  `log_action trigproc`, status half-configured (which clears pending triggers), **postinst
  `triggered "<names separated by spaces>"`**, `post_postinst_tasks`. Verified (exp): explicit
  trigger (`dpkg-trigger mytrig` from postinst) ⇒ `tgt postinst triggered mytrig`; file
  trigger (`interest-noawait /usr/share/watched`) during `--unpack` ⇒ `postinst triggered
  /usr/share/watched`, processed at the end of the unpack run.
* Trigger-cycle detection (`:191-368`): tortoise/hare over snapshots ("trigcyclenode") of every
  package's pending-trigger list taken at each trigproc; if the hare's pending set is a superset
  of the tortoise's, no progress is possible ⇒ print the chain, give up on the **earliest**
  package (status half-configured, error "triggers looping, abandoned").
* Transitional activation (`:490-561`) rebuilds interests/pending state for DBs from
  pre-triggers dpkg.

### B.10 Other files

* `enquiry.c`: `--audit` (11 problem classes, `:92-165`; warns if the DB is locked);
  `--yet-to-unpack`; `--assert-*` (compares `DPKG_RUNNING_VERSION` env against the feature
  version; outside a maintainer script always 0, `:338-380`); `--predep-package` (exit 1 if
  none, prints an available stanza, `:427-552`); `--print-architecture`,
  `--print-foreign-architectures`; `--validate-*`; `--compare-versions` with 16 operators incl.
  `-nl` variants and the obsolete `<`/`>` (= `<=`/`>=`), exit 0/1 (`:661-880`). A version that
  fails to parse only prints a warning and the comparison proceeds (`:841-858`; verified:
  `1:a!b lt 2` → warning, exit 1).
* `select.c`: `--get-selections` (name + tabs + want; patterns), `--set-selections` (stdin
  parser, unknown packages warn and point to the FAQ, `:120-235`), `--clear-selections`
  (everything non-essential/non-protected → deinstall).
* `update.c`: `--update-avail/--merge-avail/--clear-avail` take the DB lock themselves
  (`modstatdb_lock`) and rewrite `available` (`:37-123`).
* `verify.c`: `--verify` checks existence and MD5 vs `info/<pkg>.md5sums` (or conffile hash),
  output format "rpm" only: 9-char `??5??????` mask + `c` for conffiles + path (`:59-113`).
* `errors.c`: per-package/per-archive error printer emits `status: <x> : error : <msg>` and
  queues the name; when `nerrs` reaches `--abort-after` (default 50) sets `abort_processing`
  ("too many errors, stopping"); exit status 1 if any error was queued
  (`reportbroken_retexitstatus`, `:108-127`). Verified with `--abort-after=1`.
* `script.c` — see G.1.
* `help.c`: `namenodetouse` (diversion resolution), `checkpath`, force helpers,
  `dir_has_conffiles`/`dir_is_used_by_*`, conffile flags, `log_action` (writes the log line and
  `processing: <action>: <pkg>`).
* `filters.c` (`:46-134`): `--path-exclude/--path-include` stored in order; **last match wins**;
  `fnmatch(pattern, name+1, 0)` — flag 0 means `*` matches `/`; excluded directories/symlinks
  are re-included if any include pattern's literal prefix (up to the first `*?[\`) is a prefix
  of the path, erring on the side of keeping directories.
* `file-match.c`: tiny linked list used to defer renames during `pkg_infodb_update`.
* `perpkgstate.c`: `ensure_package_clientdata` (arena-allocated, never freed).

---------------------------------------------------------------------------------------------

## C. External contracts a reimplementation must preserve

### C.1 CLI surface (counted from the `cmdinfos[]` tables)

| Program | Actions | Options | Notes |
|---|---|---|---|
| dpkg | 44 (30 native, 5 → dpkg-query, 9 → dpkg-deb) | 32 table entries (3 of them are the `force/refuse/no-force` prefixes) | 29 force names; 13 debug flags; 7 assert features; 16 compare operators; default action none |
| dpkg-deb | 13 (`-b -c -e -I -f -x -X -R --ctrl-tarfile --fsys-tarfile -W --help --version`) | 13 (`--deb-format --debug -v --nocheck --no-check --root-owner-group --threads-max --[no-]uniform-compression -Z -z -S --showformat`) | env `DPKG_DEB_THREADS_MAX`, `DPKG_DEB_COMPRESSOR_TYPE`, `DPKG_DEB_COMPRESSOR_LEVEL` (`deb/main.c:374-382`) |
| dpkg-query | 11 (`-L -s -p -l -S -W -c --control-list --control-show --help --version`) | 5 (`--admindir --root --load-avail -f/--showformat --no-pager`) | stdout fully buffered (`dpkg_set_report_piped_mode(_IOFBF)`, `query/main.c:1000`) |
| dpkg-split | 8 (`-s -j -I -a -l -d --help --version`) | 7 (`--admindir --root --depotdir -S -o -Q --msdos(obsolete)`) | |
| dpkg-divert | 7 (`--add --remove --list --listpackage --truename --help --version`) | 10 | default action `--add` (`divert/main.c:938-939`) |
| dpkg-statoverride | 5 (`--add --remove --list --help --version`) | 9 (incl. deprecated bare `--force`) | |
| dpkg-trigger | 3 (`--check-supported --help --version`) | 6 | default action = activate trigger named by the argument |
| dpkg-realpath | 0 (help/version are options that exit) | 5 | only `argv[0]` is used; extra args ignored silently |

Parser quirks that are part of the contract: no permutation (options must precede operands),
`--opt=value` and `--opt value`, short bundling, `--force-<x>` suffix syntax, config files
can set actions (dpkg only), conflicting actions → exit 2.

### C.2 Exit codes (verified)

* All programs: **2** for usage errors and fatal errors (`badusage`, `catch_fatal_error` →
  `exit(2)` at `lib/dpkg/ehandle.c:113-118`; "unrecoverable fatal error" also exits 2,
  `:454-464`). `internerr` calls `abort()` (`:540-559`).
* dpkg: **1** if any package/archive error was recorded (`errors.c:126`); 0 otherwise; check
  commands (`--compare-versions`, `--validate-*`, `--assert-*`, `--predep-package`) return 0/1.
* dpkg-query: 1 if any pattern/package/path not found; 2 for a bad `--showformat`
  (`query/main.c:637-642`); `--control-*` misses are fatal (2).
* dpkg-split: `--auto` on a non-part → 1 (verified); otherwise 0/2.
* dpkg-statoverride: `--list` with no match → 1; `--remove` of a missing override → **2**
  unless `--force-statoverride-remove` (`statoverride/main.c:364-371`; verified).
* dpkg-trigger `--check-supported`: 1 when `triggers/` or `Unincorp` missing.
* A failing `--pre-invoke` hook is fatal (exit 2) and the message prints the raw wait status
  ("exit code 256" for `false`, verified; `main.c:622-625`).

### C.3 `--status-fd` / `--status-logger` protocol

Lines (newlines in messages mapped to spaces, `lib/dpkg/log.c:111-130`; all fds set CLOEXEC,
`log.c:103`):

* `status: <pkg>: <state>` — every `modstatdb_note` with a dirty status (`dbmodify.c:534-538`).
* `processing: <stage>: <pkg>` — `install|upgrade|configure|trigproc|disappear|remove|purge`
  (`help.c:352-353`).
* `status: <pkg-or-archive> : error : <msg>` (`errors.c:90,103`).
* `status: <conffile> : conffile-prompt : '<real-old>' '<real-new>' <useredited> <distedited> `
  (trailing space; `configure.c:295-297`).

Documented in `man/dpkg.pod:1243-1287`. Status loggers are spawned with `sh -c -- <cmd>`
(`DPKG_DEFAULT_SHELL`, not `$SHELL`, because `command_shell` only uses `$SHELL` for the
interactive case, `lib/dpkg/command.c:195-217`) with the pipe as stdin; they and the hooks run
only for invoke actions, only when not `--no-act`, and only as root or with `--force-not-root`
(`main.c:546-580`; verified both cases). Observed quirk: a `--remove` run emits `status: act:
installed` before `processing: remove: act` (the selection change marks the record dirty).

### C.4 Log file format (`--log`)

`YYYY-MM-DD HH:MM:SS ` (localtime) + one of: `startup archives|packages <action>`;
`status <state> <pkg:arch> <installed-version>`; `<action> <pkg:arch> <installed-ver>
<available-ver>` (`<none>` when absent); `conffile <path> install|keep`
(`lib/dpkg/log.c:48-89`, `help.c:347-350`, `dbmodify.c:534`, `archives.c:1700`,
`packages.c:156`, `configure.c:311`). Verified sample in Appendix. Opened `O_APPEND`, mode 0644,
CLOEXEC; write failures are notices, not errors.

### C.5 Hooks and environment

* Hooks: `--pre-invoke`/`--post-invoke` run via `system()` (so `/bin/sh -c`, SIGINT/SIGQUIT
  ignored by libc during the call) with `DPKG_HOOK_ACTION=<action>`, in configuration order;
  non-zero ⇒ fatal (`main.c:610-629`).
* Exported by dpkg to **all** children: `DPKG_ADMINDIR`, `DPKG_ROOT`, `DPKG_FORCE`
  (`main.c:1017-1025`); `DPKG_PAGER=cat` with `--no-pager`.
* Maintainer scripts additionally get `DPKG_MAINTSCRIPT_PACKAGE` (set name, no arch),
  `DPKG_MAINTSCRIPT_PACKAGE_REFCOUNT` (installed instances of the set),
  `DPKG_MAINTSCRIPT_ARCH`, `DPKG_MAINTSCRIPT_NAME`, `DPKG_MAINTSCRIPT_DEBUG` (0/1),
  `DPKG_RUNNING_VERSION` (`script.c:202-208`). In the chroot case `DPKG_ROOT=""` and
  `DPKG_ADMINDIR` is rewritten relative to the chroot (`script.c:116-119`). Verified values
  (chrootless): `ROOT=<instdir>`, `ADMIN=<instdir>/usr/local/var/lib/dpkg`,
  `FORCE=security-mac,downgrade,not-root,script-chrootless`, `REFCNT=1`, `DBG=0`, cwd =
  instdir.
* Conffile shell: `DPKG_SHELL_REASON`, `DPKG_CONFFILE_OLD/NEW`. Pager: `LESS=-FRSXMQ` if unset
  (`lib/dpkg/pager.c:125`).
* Consumed: `DPKG_FORCE`, `DPKG_ROOT`, `DPKG_ADMINDIR`, `DPKG_FRONTEND_LOCKED`, `DPKG_DEBUG`,
  `DPKG_COLORS`, `DPKG_NLS`, `DPKG_PAGER`/`PAGER`, `SHELL`, `HOME` (config), `TMPDIR`,
  `COLUMNS`, `PATH`, `DPKG_PATH_PASSWD/GROUP` (`man/dpkg.pod:1386-1512`); `SOURCE_DATE_EPOCH`
  (dpkg-deb, dpkg-split); `DPKG_MAINTSCRIPT_PACKAGE/ARCH` (dpkg-divert, dpkg-trigger);
  `DPKG_RUNNING_VERSION` (dpkg `--assert-*`); `TAR_OPTIONS` is explicitly unset before running
  tar (`build.c:676`, `extract.c:380`).

### C.6 Locks

* `<admindir>/lock-frontend` then `<admindir>/lock`, both `open(O_RDWR|O_CREAT|O_TRUNC, 0660)`
  and whole-file POSIX record locks `fcntl(F_SETLK, F_WRLCK, start=0, len=0)` — non-blocking;
  the frontend lock is skipped if `DPKG_FRONTEND_LOCKED` is set (`lib/dpkg/dbmodify.c:263-307`;
  `doc/spec/frontend-api.txt:16-30`). Verified (strace): fd3 lock-frontend `F_SETLK`, fd4
  lock `F_SETLK`, released in reverse at exit. On contention the error names the holder via
  `F_GETLK` + `/proc/<pid>` (`lib/dpkg/file.c:310-347`). Lock fds are CLOEXEC, so maintainer
  scripts do not hold them; record-lock semantics (released on any `close()` of that file by
  the process, not inherited by fork) are part of the contract.
* `triggers/Lock` — blocking `F_SETLKW`, taken by anything that rewrites `triggers/Unincorp`
  (`lib/dpkg/trigdeferred.c:78-92`): dpkg and dpkg-trigger.
* Who locks what: dpkg mutating actions (`needsuperuser`/`write`) take both DB locks;
  `--update/--merge/--clear-avail` lock explicitly; `dpkg-query`, `dpkg-divert`,
  `dpkg-split` and `dpkg-statoverride` take **no** DB lock (divert opens `msdbrw_readonly`,
  `divert/main.c:539,689,773,825,852`; statoverride never opens the DB) even though divert and
  statoverride rewrite DB files; they rely on being run from maintainer scripts while dpkg holds
  the lock, and dpkg re-reads diversions after every maintainer script (`script.c:68-76,282-296`).

### C.7 Admin-dir layout as touched by `src/` (observed after test installs)

`status` (+`status-old`), `available`, `lock`, `lock-frontend`, `arch` (only after
`--add-architecture`), `diversions`(-old), `statoverride`(-old), `updates/NNNN` journal +
`updates/tmp.i`, `info/format`, `info/<pkg>[:<arch>].{list,md5sums,conffiles,preinst,postinst,
prerm,postrm,triggers,…}`, `triggers/{File,Unincorp,Lock,<trigger-name>}` (interest files list
`<pkg>[/noawait]`), `tmp.ci/` (control staging, removed after each archive), `reassemble.deb`,
`parts/` (dpkg-split depot, `<md5>.<maxpartlen-hex>.<n-hex>.<max-hex>`, `split/queue.c:47-83`).
Backup/journal semantics belong to the libdpkg analyst; the programs depend on
`modstatdb_note` being durable before returning (it runs under a fatal-errors section).

### C.8 Other observable contracts

* Package-file naming during unpack/configure: `.dpkg-new`, `.dpkg-tmp`, `.dpkg-old`,
  `.dpkg-dist` (documented `man/dpkg.pod:1684-1728`), plus `.dpkg-divert.tmp` (divert),
  `.dpkg-backup/.dpkg-remove/.dpkg-bak/.dpkg-staging-dir` (maintscript-helper).
* Human output parsed by tools (unverified which): "Selecting previously unselected package",
  "(Reading database ... N files and directories currently installed.)", "Unpacking", "Setting
  up", "Processing triggers for", `-l` table header, `--get-selections` format, `-S` output
  `pkg: /path` and diversion lines, `-L` diversion annotations.
* Bash completion reads `/var/lib/dpkg/status` directly (`completion/bash/dpkg:43,55`) and
  derives options from `--help` output.
* apt: beyond the documented lock protocol and the `trigproc.c` comment, I did not verify from
  in-repo sources which flags apt relies on (unverified: `--status-fd`, `--configure
  --pending`, `--no-triggers`, `--triggers-only`, `--auto-deconfigure`, `--assert-multi-arch`,
  `--print-foreign-architectures`, `--force-*`).

---------------------------------------------------------------------------------------------

## D. dpkg-deb

### D.1 Build (`src/deb/build.c`)

`do_build` (`build.c:618-806`):

1. Destination: given file, or `<dir>.deb`, or when given a directory
   `<dir>/<pkg>_<version-without-epoch>_<arch>.deb` (`:470-510`).
2. Checks unless `--no-check` (`:438-458`): parse `DEBIAN/control` with libdpkg `parsedb`
   (name charset `[a-z0-9+-.]`, architecture required, user-defined Priority warning);
   `DEBIAN` mode must satisfy `(mode & 07757) == 0755`, maintainer scripts
   (`preinst postinst prerm postrm config`) must be regular files or symlinks with
   `(mode & 07557) == 0555` (`:211-249`); root dir not owned by 0:0 ⇒ warning + hint
   `--root-owner-group` (`:410-429`); `conffiles` syntax incl. `remove-on-upgrade` flag,
   existence, duplicates (`:255-375`); newline in any pathname is fatal (`:173-175`).
3. Timestamp: `SOURCE_DATE_EPOCH` if set (strict integer), else `time(NULL)` (`:664-668`).
4. Creates the ar file (`dpkg_ar_create`, mode 0644, ar mtime = timestamp), unsets
   `TAR_OPTIONS`.
5. Control member: `tarball_pack(DEBIAN, control_treewalk_feed, mode "u+rw,go=rX",
   owner/group root:0 always)` into an unlinked `mkstemp` temp file; compressor = the data
   compressor if `--uniform-compression` (default on) else gzip (`:691-700`).
6. `tarball_pack` (`:523-597`): pipe filenames → **exec GNU tar** `tar -cf - --format=gnu
   --mtime @<ts> --clamp-mtime [--mode M] [--owner root:0 --group root:0] --null --no-unquote
   --no-recursion -T -` (verified with strace) in `chdir(dir)`; a second forked child (no exec)
   runs libdpkg `compress_filter` from tar's stdout to the temp file; parent writes the
   NUL-separated file list. Order: libdpkg `treewalk` sorts each directory's entries with
   `strcmp` (`lib/dpkg/treewalk.c:224-236`), `DEBIAN*` is skipped (`build.c:168`), and all
   **symlinks are appended after all other entries** (`:177-195`) so they never precede their
   targets.
7. Format 2.0: `!<arch>\n`, `debian-binary` = `2.0\n`, `control.tar[.gz|.xz|.zst]`,
   `data.tar[.ext]` (`:733-797`). Format 0.939000: `0.939000\n<ctrl-len>\n` + control
   tar.gz + data tar concatenated (`:717-732,752-756`; gzip forced, `deb/main.c:392-393`).
8. `fsync` the output, close.

Compressors: `-Z gzip|xz|zstd|none` (bzip2/lzma rejected as obsolete for build,
`deb/main.c:289-305`); with uniform compression only none/gzip/xz/zstd allowed (`:398-404`);
`-z` level, `-S` strategy, `--threads-max`. In this build compression is in-process via
zlib/liblzma/libzstd (`lib/dpkg/compress.c`), external compressors are a fallback only when a
library is missing at build time.

Reproducibility (verified): two builds with the same `SOURCE_DATE_EPOCH` are byte-identical
(sha256 equal); without it they differ. Note: dpkg-deb does not add `Installed-Size`
(verified `-W` shows it empty); that is `dpkg-gencontrol`'s job (Perl side).

### D.2 Extract (`src/deb/extract.c`)

`extracthalf(deb, dir, taroptions, admininfo)` (`extract.c:93-404`) is the single reader:

1. `dpkg_ar_open` (path or `-` for stdin; pipes get size 0, `lib/dpkg/ar.c`), read first line.
2. `!<arch>\n` ⇒ new format: odd total size rejected; for each member header: fmag check,
   name normalisation (trailing spaces and GNU `/`), decimal size parse (rejects non-digits);
   first member must be `debian-binary` and its major version must be 2 (minor/extra lines
   ignored); members starting with `_` are skipped; `control.tar<ext>` must come before any
   other member, at most once, non-empty, compressor in {none,gz,xz,zst}; next must be
   `data.tar<ext>` (any known compressor incl. bz2/lzma), non-empty; overflow and truncation
   checks against file size when seekable (`:121-260`). Members after `data.tar` are never
   read.
3. `0.93…` ⇒ old format: minor ≥ 939000, control length line via `sscanf("%jd%c%d")`
   (`:261-301`).
4. Pipeline (`:311-403`): **child c1** (no exec) copies exactly `memberlen` bytes from the
   archive into pipe p1; **child c2** (no exec) runs libdpkg `decompress_filter` p1 → p2 (or
   stdout); if a tar operation is requested **child c3 execs GNU `tar`** (`-x`, `-xv`, `-tv`,
   plus `-p`, `-m`, `-f - --warning=no-timestamp`, after `mkdir`/`chdir` into the target);
   the parent reaps c3, c2 (`SUBPROC_NOPIPE`), c1.
5. Actions: `--fsys-tarfile`/`--ctrl-tarfile` = passthrough, no tar parsing; `-e/--control`
   extracts control via GNU tar into `DEBIAN` or the given dir; `-x/-X` with `-p`; `-R`
   (`--raw-extract`) data then control into `<dir>/DEBIAN` with "must not pre-exist"
   (`DPKG_TAR_CREATE_DIR`); `-c/--contents` = `tar -tv`.

dpkg itself never runs tar for package data: it runs `dpkg-deb --fsys-tarfile` and parses the
tar stream in-process with libdpkg `tar_extractor` (B.5). It does use GNU tar indirectly for the
control member (`dpkg-deb --control`).

### D.3 Info (`src/deb/info.c`)

`-I`, `-f`, `-W` first extract the control member with GNU tar into a `mkdtemp` dir (cleanup
`cu_info_prepare` chmods dirs 0755 then `path_remove_tree`, `:53-101`). `-I` without
components lists files with size/lines/`*` exec mark/shebang interpreter (≤PATH_MAX chars,
`:151-183`) and prints `control` indented; with components dumps them raw (missing ⇒ error
with count). `-f` parses `control` with libdpkg (`fieldinfos` + arbitrary fields) and prints
selected fields (with `Field: ` headers when more than one). `-W` uses libdpkg
`pkg_format_parse` (default `${Package}\t${Version}\n`).

### D.4 Formats and limits

* `.deb` 2.0 (`ARCHIVEVERSION`) and 0.939000 (`OLDARCHIVEVERSION`) (`dpkg-deb.h:70-79`);
  `--deb-format` accepts only these two (`deb/main.c:231-245`).
* ar member size: 10 decimal digits (`lib/dpkg/ar.h:57`) ⇒ < 10^10 bytes (≈9536.74 MiB) per
  member (`man/deb.pod:53-54`); member names ≤15 chars + `/`.
* Tar inside: v7, ustar (prefix+name), GNU longname/longlink, GNU base-256 numeric fields
  (`lib/dpkg/tarfn.c:129-180`); PAX, sparse, multivolume, Solaris types rejected
  (`tarfn.c:563-590`). Entry size >8 GiB requires GNU base-256.
* `versionbuf[40]` first-line limit; conffile lines ≤ 1000 chars (`MAXCONFFILENAME`); control
  member names ≤ 100 (`MAXCONTROLFILENAME`, enforced by dpkg at `unpack.c:578-580`).

### D.5 Places that parse untrusted input (in my area)

1. ar header/sizes/names (`extract.c:131-252` + `lib/dpkg/ar.c`), debian-binary version
   (`extract.c:150-175`), old-format header (`:261-292`).
2. Decompressors (zlib/liblzma/libzstd/libbz2) in child c2.
3. GNU tar (external) for control extraction (`dpkg -i` via `dpkg-deb --control`) and for all
   `dpkg-deb -x/-e/-c/-I/-f/-W/-R`. GNU tar refuses `..` members (verified: "Member name
   contains '..'", exit 2).
4. libdpkg `tar_extractor` + dpkg `tarobject` for data.tar during install.
5. Control file → `parsedb` (dpkg `unpack.c:1353`, dpkg-deb `info.c:283,329`, dpkg-split via a
   `dpkg-deb --info` pipe `split/split.c:58-89`).
6. `conffiles` (`unpack.c:360-485`), `triggers` control file (`trig_parse_ci`), shebang scan
   (`info.c:151-183`).
7. dpkg-split part headers (`split/info.c:92-243`, very defensive) and depot filenames
   (`queue.c:57-83`).
8. dpkg `--set-selections` stdin, `--update/--merge-avail` Packages files, config files,
   `triggers/Unincorp` (`lib/dpkg/trigdeferred.c:185-252`).

Upstream's security stance: examining/extracting untrusted .debs with dpkg-deb is a security
boundary (`man/dpkg-deb.pod` SECURITY); **installing** untrusted packages is not
("must never be done on untrusted packages", `man/dpkg.pod:1730-1748`).

---------------------------------------------------------------------------------------------

## E. The smaller programs

### E.1 dpkg-split

* `-s/--split <deb> [prefix]` (`split/split.c:107-240`): MD5 of the whole file, exec
  `dpkg-deb --info <deb> control` to get name/version/arch, part payload = partsize − 1024
  (`HEADERALLOWANCE`), default partsize 450 KiB (`dpkg-split.h:79-89`); each part is an ar
  with member `debian-split` (`2.1\n<pkg>\n<ver>\n<md5>\n<total>\n<partsize>\n<n>/<N>\n<arch>\n`)
  and `data.<n>`; ar mtime = `SOURCE_DATE_EPOCH` or now.
* `-j/--join` (`join.c:103-158`): all parts must agree on pkg/version/md5/length/count/partsize;
  output defaults to `<pkg>_<ver>_<arch>.deb`; `fsync` then close. **The MD5 of the
  reassembled file is never verified** — verified (exp): flipping a data byte in one part still
  joins with exit 0 into a file that differs from the original.
* `-I/--info` prints part metadata; non-parts are reported, still exit 0.
* `-a/--auto -o <out> <part>` (`queue.c:202-268`): used by dpkg for every archive
  (`dpkg-split -Qao <admindir>/reassemble.deb <deb>`, `unpack.c:108`). Non-part ⇒ exit 1.
  Otherwise scan the depot (`<admindir>/parts`), copy the part to `t.<pid>` then `fsync` +
  rename to `<md5>.<len>.<n>.<N>` and `dir_sync`; when complete, reassemble to `-o` and unlink
  the depot parts. **No locking** of the depot.
* `-l/--listq`, `-d/--discard [pkg…]`. Verified (exp) output bug: the header "Packages not yet
  reassembled:" is printed once **per package** because `part_found` is reset to `false`
  instead of `true` (`queue.c:330-333`).
* DB files touched: only `parts/` and `reassemble.deb`. Complexity: low, self-contained.

### E.2 dpkg-query

* Read-only, no lock. `-l` (table sized from the widest name/version/arch/description, terminal
  width or `COLUMNS`, through a pager, `query/main.c:117-344`); `-W/--show` with
  `--showformat` (libdpkg `pkg-format`; loads file lists only if the format needs `db-fsys:*`
  fields, `:628-682`); `-s/--status`, `-p/--print-avail` (stanza dumps); `-L/--listfiles` (with
  diversion annotations, `:542-610`); `-S/--search` (arguments without a leading `*[?/` are
  wrapped as `*arg*`; non-glob arguments looked up exactly after trimming `/` and `/.`;
  otherwise `fnmatch` over every known path; prints diversion lines, `:347-448`);
  `--control-path/--control-list/--control-show` (hide `list` and `conffiles`, reject `/` and
  `.` in names, `:684-844`). Patterns use libdpkg `pkg-spec` (glob + arch wildcards).
* Exit 1 on any miss (see C.2).

### E.3 dpkg-divert

* `--add` (default action): file must be absolute, not a directory, no newline; default
  divert-to `<file>.distrib`; package from `--package`, `--local`, or
  `DPKG_MAINTSCRIPT_PACKAGE` (`divert/main.c:934-936`); idempotent re-add prints "Leaving";
  clash ⇒ error; `--rename` checks writability by creating `<name>.dpkg-divert.tmp` in both
  places, refuses to overwrite a different existing file, skips renaming a file owned by the
  diverting package, warns for Essential files; rename falls back to copy+fsync+rename+unlink
  across filesystems (`:237-349`). DB written atomically with backup
  (`atomic_file_new(…, ATOMIC_FILE_BACKUP)`, `:433-466`) **before** the rename.
* `--remove`: must match `--divert`/`--package` if given; a diversion still needed by another
  architecture instance of the set (judged by `DPKG_MAINTSCRIPT_ARCH`) is kept (`:641-671`).
* `--list [glob]`, `--listpackage` (`LOCAL` for local diversions), `--truename`.
* Default `--no-rename` with a deprecation warning when neither given (`:180-189`), although
  the warning says the default "will change … not before 1.23.x" and this tree is 1.23.12.
* DB files: `diversions` (rw), `status` + file lists (read, for ownership checks). No lock.
* Best-tested small program: 29 autotest groups + 12 chdir groups.

### E.4 dpkg-statoverride

* `--add <owner> <group> <mode> <path>` (owner/group by name or `#id`, must exist; existing
  override ⇒ error unless `--force-statoverride-add`); `--update` applies chown/chmod and
  SELinux context to the existing file immediately (`statoverride/main.c:220-235,326-342`);
  `--remove <path>`; `--list [glob]`. Trailing `/` stripped with a warning.
* DB: `statoverride` written via `atomic_file` with backup + `dir_sync_path(admindir)`
  (`:266-291`). Reads `/etc/passwd`/`group` (honouring `DPKG_PATH_PASSWD/GROUP`, libdpkg
  sysuser). No lock, no package DB access at all.
* Force options limited by `FORCE_STATCMD_MASK` (`:157-159`) but `DPKG_FORCE` inherited from
  dpkg is honoured.

### E.5 dpkg-trigger

* `dpkg-trigger [--no-await|--await] [--by-package=P] <trigger>`: awaiting package defaults
  to `DPKG_MAINTSCRIPT_PACKAGE`+`DPKG_MAINTSCRIPT_ARCH` (else badusage "must be called from a
  maintainer script"); `--no-await` writes `-` (`trigger/main.c:137-167`). Validates the
  trigger name, then rewrites `triggers/Unincorp` under `triggers/Lock` (blocking): copies
  existing lines, appends the awaiter to an existing line for the same trigger or adds
  `<trigger> <pkg>` (`:169-237`); `--no-act` parses but does not write.
* `--check-supported` returns 1 if the triggers dir or `Unincorp` is missing (`:239-261`).
* Note (libdpkg): `Unincorp.new` is renamed into place and the directory synced, but the file
  itself is not fsynced before the rename (`lib/dpkg/trigdeferred.c:265-281`).
* Covered by 1 autotest group (no-act) + chdir tests; real behaviour covered by root `tests/`
  trigger cases (another analyst).

### E.6 dpkg-realpath

* Lexical walk resolving symlinks component by component inside `--root`/`--instdir`
  (`realpath/main.c:111-232`): `..` never escapes the root, absolute link targets restart at
  root, >25 links ⇒ error, link size must match `lstat`; refuses arguments that already
  include the root prefix (`:261-264`); relative arguments are made absolute from `getcwd` and
  the root prefix stripped. `-z` NUL terminator. Note `--root` maps to `set_instdir` (does not
  change admindir, `:237`).
* Used by `dpkg-maintscript-helper` (`symlink_match`, `dir_to_symlink`). 2 autotest groups
  (26 checks).

---------------------------------------------------------------------------------------------

## F. Shell scripts

### F.1 `dpkg-maintscript-helper.sh` (685 lines)

* Called by package maintainer scripts (Debian packages via debhelper's `dh_installdeb`
  maintscript support — (unverified, outside repo)); in-repo callers are the functional tests
  `tests/t-conffile-{obsolete,rename}`, `tests/t-switch-*` (`grep`).
* Commands: `supports <cmd>`; `rm_conffile <conffile> [<prior-version> [<pkg>]] -- "$@"`;
  `mv_conffile <old> <new> …`; `symlink_to_dir <path> <old-target> …`;
  `dir_to_symlink <path> <new-target> …`; internal `_internal_pkg_must_own_file` guarded by
  `DPKG_MAINTSCRIPT_HELPER_INTERNAL_API=$version` (`:560-573`).
* Behaviour is a per-script state machine keyed on `DPKG_MAINTSCRIPT_NAME` and the script
  args, with version gating by `dpkg --compare-versions -- "$2" le-nl "$LASTVERSION"`
  (`:63-93` etc.). It renames to `.dpkg-backup/.dpkg-remove/.dpkg-bak`, compares MD5 with the
  `Conffiles` hash from `dpkg-query -W -f='${Conffiles}'` (`:104-112`), checks ownership with
  `dpkg-query -L | grep -F -x` (`:547-558`), and for `dir_to_symlink` moves the directory to
  `.dpkg-backup`, creates a staging dir marked `.dpkg-staging-dir`, and at postinst moves new
  content to the symlink target and creates the symlink (`:441-531`). Honours `DPKG_ROOT`
  (normalised with `realpath`, `:615-620`).
* Rewrite assessment: it is glue around dpkg/dpkg-query/coreutils whose interface is the CLI
  and env; the logic is intricate (abort paths in postrm) but small. Rewriting it in Rust
  would change nothing visible and add a binary dependency on maintainer-script time; keeping
  it as shell (requires only `/bin/sh` + coreutils, both Essential) is the low-risk option.
  If a Rust dpkg integrated dir↔symlink switching natively, the helper would remain needed for
  compatibility with existing packages.

### F.2 `dpkg-db-backup.sh` (84 lines)

Run daily by `debian/dpkg.dpkg-db-backup.{service,timer}` or `debian/dpkg.cron.daily` (only
when systemd is not running). Copies `arch status diversions statoverride` into
`/var/backups/dpkg.<db>` and rotates with `savelog -c 7` only if any changed (`cmp`); tars
`alternatives/` and rotates if `tar -d` reports changes (`:47-84`). Uses hard-coded
`ADMINDIR` (substituted at build), no locking (unverified whether that matters: it only reads).
Trivial; best left as shell.

### F.3 `dpkg-db-keeper.sh` (56 lines)

Optional `post-invoke` hook (documented in its own comment, `:26-32`) that `git init`s the
admindir and commits after every run; no-op if `DPKG_ROOT` is set or git missing. Trivial; no
tests.

### F.4 `sh/dpkg-error.sh` (136 lines)

Shell reporting library (since 1.20.1) installed to `<pkgdatadir>/sh/`; sourced by the three
scripts above (via `DPKG_DATADIR` override) and by `debian/dpkg.postinst` (`. /usr/share/dpkg/
sh/dpkg-error.sh`). Public functions with "Supported since" annotations are a stable API for
third parties. Must stay shell.

### F.5 Bash completion

`completion/bash/{dpkg,dpkg-deb,dpkg-query}` (438 lines) use bash-completion ≥2.12 APIs; parse
`/var/lib/dpkg/status` with awk for package names, `dpkg --get-selections` for holds, and
`_comp_compgen_help` (i.e. `--help` output) for options. Stays shell; any change in `--help`
format or option names affects it.

---------------------------------------------------------------------------------------------

## G. Process and privilege model

### G.1 Fork/exec sites and pipe topologies

dpkg, per archive with `-i` (verified with `strace -f -e execve`):

```
dpkg ──fork/exec──> dpkg-split -Qao <admindir>/reassemble.deb X.deb      (wait; rc 0/1)
     ──fork/exec──> [debsig-verify -q X.deb]                             (only if in PATH)
     ──fork/exec──> dpkg-deb --control X.deb <admindir>/tmp.ci
                        ├─fork: copy control member ─pipe─> fork: decompress ─pipe─> exec tar -x -f - --warning=no-timestamp
     ──fork/exec──> <admindir>/tmp.ci/preinst ARGS...     (new scripts run from tmp.ci)
     ──fork/exec──> dpkg-deb --fsys-tarfile X.deb ─stdout pipe─> dpkg (libdpkg tar_extractor, in-process)
                        └─fork: copy data member ─pipe─> fork: decompress ─> stdout
     ──fork/exec──> <admindir>/info/<pkg>.postrm ARGS...  (old scripts from info/)
     ──fork/exec──> rm -rf -- <non-empty dir>             (path_remove_tree fallback, path-remove.c:157)
     ──fork/exec──> <admindir>/info/<pkg>.postinst configure ...
```

Other sites: `system()` for invoke hooks (`main.c:622`); status loggers `sh -c` with a pipe as
stdin (`main.c:643-665`); conffile `diff -Nu` into `pager_spawn` and `$SHELL -i`
(`configure.c:201-259`); exec-replace to `dpkg-query`/`dpkg-deb` (`main.c:853-868`). dpkg-deb
build: `tar -cf` child + compressor child connected by pipes, parent feeds filenames
(`build.c:523-597`); dpkg-split: `dpkg-deb --info` with stdout pipe (`split.c:58-89`).
dpkg-query/dpkg: pager processes (`lib/dpkg/pager.c`).

All forks go through `subproc_fork` which, in the child, pushes a fresh error context so that a
failure before `exec` exits 2 without running the parent's cleanups (`lib/dpkg/subproc.c`).
Children are reaped synchronously (`waitpid` loop on EINTR); status checks map exit codes and
signals (`SUBPROC_NOPIPE` ignores SIGPIPE for the decompressor/backend; `SUBPROC_WARN` for the
"old script" first attempt; `SUBPROC_RETERROR` for dpkg-split).

Maintainer script exec (`script.c:175-225`): `chmod 0755` the script if not `0555`-executable
(modifies the file, `:84-93`); push `cu_post_script_tasks` (bombout); fork; child sets env,
chroots (`maintscript_pre_exec`, `:98-156`): if instdir is non-empty and not
`--force-script-chrootless` ⇒ admindir must be under instdir, `chroot(instdir)` (EPERM under
`--force-not-root` gives a hint about chrootless), `chdir("/")`; chrootless ⇒ `chdir(instdir)`;
SELinux `setexecfilecon(script, "dpkg_script_t")`; `execvp(path, [path, args…])`. Parent
**ignores SIGINT and SIGQUIT only while waiting** for the script (`subproc_signals_ignore`,
`:218-220`). After a script: reload diversions and incorporate triggers (`post_script_tasks`).
stdin/stdout/stderr are inherited (debconf/prompts work); there is no `closefrom`; only fds that
libdpkg marked CLOEXEC (locks, log, status-fds, DB files, trigger files) are closed on exec.

### G.2 Signals

No global signal handlers in `src/` or libdpkg (grep `sigaction|signal(`): only the SIGINT/
SIGQUIT ignore window above and SIGPIPE ignored while a pager runs (`lib/dpkg/pager.c:117-156`).
A SIGINT/SIGTERM at any other time kills dpkg without unwinding; recovery relies on the on-disk
state (journal in `updates/`, `reinstreq` + half-* states, `.dpkg-new/.dpkg-tmp` recovery in
`tarobject`, `:845-873`).

### G.3 Root / non-root

* Mutating actions use `msdbrw_needsuperuser` ⇒ "requested operation requires superuser
  privilege" (exit 2) unless `--force-not-root` (verified). `--no-act` opens read-only.
* As root, primary gid forced to 0 (`main.c:1010-1012`).
* EPERM from chown/chmod/lchown is tolerated with `--force-not-root`
  (`forcible_nonroot_error`). fakeroot is the normal way to build with dpkg-deb as non-root;
  dpkg-deb `--root-owner-group` avoids needing it.
* Hooks/status loggers not run as non-root without `--force-not-root` (G.1/C.3).
* dpkg-query never needs root and the man page warns against running it via gain-root because
  of the pager (`man/dpkg-query.pod` SECURITY).

### G.4 chroot / `--root` / `--instdir` / `--admindir`

`--root=D` sets instdir D and admindir `D<compiled ADMINDIR>` (`lib/dpkg/options-dirs.c:50-61`);
`--instdir` changes only the install root; `--admindir` only the DB; env `DPKG_ROOT`,
`DPKG_ADMINDIR` apply when no option is given. All filesystem paths are built by string
concatenation `instdir + "/path"` (`setupfnamevbs`, `archives.c:671-687`; `remove.c:310-311`),
not with `openat`/dirfds. Scripts are chrooted unless chrootless (G.1). Verified: with
`--root` and chrootless, scripts see `DPKG_ROOT=<instdir>` and cwd = instdir. In `src/at`,
`--root/--instdir/--admindir` are exercised by the 33 `chdir.at` groups, but only for option/env
precedence and path normalisation (they check the `root=… admindir=…` debug line).

### G.5 SELinux (`src/common/selinux.c`, `security-mac.h`)

Enabled when built `WITH_LIBSELINUX` and `--force-security-mac` (default on) and
`is_selinux_enabled() > 0`. `dpkg_selabel_load` is called **per archive** (`archives.c:1785`):
opens the status channel once, then reloads the file-context handle when
`selinux_status_updated()` (policy package upgraded mid-run) (`selinux.c:74-111`).
`dpkg_selabel_set_context(matchpath, path, mode)` looks up the context for the final path and
applies it with `lsetfilecon_raw` to the `.dpkg-new`/`.dpkg-tmp` object; ENOTSUP ignored
(`:113-143`). Maintainer scripts get `setexecfilecon(…, "dpkg_script_t")` (`script.c:165-173`).
dpkg-statoverride `--update` applies contexts too.

### G.6 Atomicity and path safety in the unpack path

* New objects are created as `<path>.dpkg-new` (`O_EXCL`, mode 0 until fchmod), old objects
  kept as `<path>.dpkg-tmp` (hardlink for files, copy for symlinks, rename for directories),
  replaced with `rename(2)`; regular files fsynced (batched) before rename unless
  `--force-unsafe-io` (B.5). DB writes are journalled and fsynced by libdpkg (strace shows 36
  of 37 fsyncs on admindir files/dirs for a 2-file package). `tmp.ci` contents fsynced before
  being renamed into `info/` (`unpack.c:1341`, `dir_sync_path` at `:629`).
* Durability gaps observed: conffile `.dpkg-new` never fsynced; no parent-directory fsync after
  renames of installed files (B.5; strace evidence in Appendix).
* Symlink attacks: symlinks are created last by the extractor (`tarfn.c:540-551,598-605`); new
  files are `O_EXCL`; removal uses `lstat` + `secure_unlink` (setid/sticky files neutralised
  with `chmod 0600` before unlink so hardlinked copies are not usable,
  `lib/dpkg/path-remove.c:36-77`). But **existing** symlinks to directories in the target tree
  are deliberately followed (e.g. merged-/usr layouts): the `existingdir` logic and all
  path-string operations traverse them (`archives.c:882-904`).
* Path traversal: `tarobject` only rejects `\n` in names; `fsys_hash_find_node` strips leading
  `/` and `./` but **does not reject `..` components**. Verified (exp): a hand-crafted .deb with
  member `./../ESCAPED-by-dpkg` installed with `--root=<S>/inst/root` wrote
  `<S>/inst/ESCAPED-by-dpkg` outside the instdir and recorded `/../ESCAPED-by-dpkg` in the file
  list; `dpkg-deb -x` of the same file failed because GNU tar refuses `..`. With instdir `/`
  the path cannot leave `/`. This is consistent with upstream's "never install untrusted
  packages" stance (D.5) but differs from dpkg-deb's behaviour.
* `umask(022)` for all programs (`lib/dpkg/program.c:49`); modes from the archive/statoverride
  are applied explicitly with `fchmod/chmod`, so umask only affects files created by dpkg's own
  helpers and inherited by maintainer scripts.

---------------------------------------------------------------------------------------------

## H. Error unwinding in the install path

### H.1 Mechanism (libdpkg `ehandle.c`)

* An **error context** = {handler (longjmp target or function), printer(+data), list of
  cleanup entries, message} (`lib/dpkg/ehandle.c:59-80`). Contexts form a stack.
* `ohshit/ohshite/forcibleerr` format the message and call `run_error_handler`: if
  `onerr_abort` (inside a "fatal errors section") ⇒ print "unrecoverable fatal error,
  aborting" and `exit(2)` **without unwinding**; else longjmp to the context's `jmp_buf` (or
  call its function) (`:451-478,492-537`). 316 `ohshit*` call sites in `src/` (149 in
  `src/main`).
* A **cleanup entry** holds up to two callbacks each with a flag mask, a checkpoint
  mask/value, and `void *argv[]` (`:45-57`). `push_cleanup(fn, mask, n, …)`,
  `push_cleanup_fallback(fn1, mask1, fn2, mask2, n, …)`, `push_checkpoint(mask, value)`,
  `pop_cleanup(flags)` (runs the top entry's callbacks whose mask matches).
* `pop_error_context(flags)` pops the context and `run_cleanups(ctx, flags)` walks the
  entries **newest to oldest**: for each entry runs callbacks whose `mask & flagset`; then
  `flagset = (flagset & cpmask) | cpvalue` — so a checkpoint
  `push_checkpoint(~ehflag_bombout, ehflag_normaltidy)` turns a bomb-out into a "normal tidy"
  for everything pushed before it (`:260-311,322-344`).
* Errors **inside** a cleanup are caught by a temporary context, printed as "error while
  cleaning up", and the walk continues with `ehflag_bombout|ehflag_recursiveerror`; more than 3
  nested recoveries ⇒ flagset 0 (nothing more runs) (`:273-301`).
* Allocation failure while pushing uses a static emergency entry then aborts (`:346-390`).
* `modstatdb_note` and output flushing run inside `push_fatal_errors_section` ⇒ an error while
  journalling the DB ends the process immediately rather than unwinding over a possibly
  half-written journal (`lib/dpkg/dbmodify.c:518,552`; `archives.c:1788-1791`).

Flags: `ehflag_normaltidy` (success path), `ehflag_bombout` (error), `ehflag_recursiveerror`.
Masks used in `src/main`: `~0` (always), `~ehflag_normaltidy` (only on error = undo),
`ehflag_bombout` (only on error, e.g. close fds), `ehflag_normaltidy` (only on success, the
"ok" half of a fallback).

### H.2 Contexts in dpkg

1. Program context (`dpkg_program_init`): handler exits 2.
2. One context per archive (`archivefiles`, `archives.c:1776-1794`), per queued package
   (`process_queue`, `packages.c:288-335`), per deferred trigger package
   (`trigproc.c:160-171`); each uses `setjmp` and therefore `volatile`/`static` variables
   (`archives.c:1678-1679`, `packages.c:186,236`, and the `static` locals of
   `process_archive` whose **addresses** are given to cleanups, `unpack.c:1282-1288`).
3. Child processes get a fresh context in `subproc_fork`.

### H.3 Cleanup entries pushed during `process_archive` (in push order)

Static push sites reachable from `process_archive`: 14 in `unpack.c`
(104, 330, 374*, 567*, 1324, 1439, 1467, 1494, 1514, 1519, 1527, 1625, 1641, 1671), 2 in
`archives.c` (403*, 1106), 1 in `script.c` (191*) + 1 in libdpkg `subproc_signals_ignore`*,
1 in `configure.c` (818*, md5 of on-disk file when refcounting) — `*` = transient, popped by the
same function; plus 2 checkpoints (`unpack.c:1691,1799`) and, via `removal_bulk` for
conflictors, 4 checkpoints + 1 transient push in `remove.c` (297, 408, 424, 507; 623*).
`src/main` totals: 21 `push_cleanup*`, 6 `push_checkpoint`, 10 `pop_cleanup`.

Live entries on the per-archive stack for an upgrade (in order), with what they do on error
(bombout) vs success (normaltidy):

| # | Entry (site) | mask | On error (before checkpoint 1) | On success |
|---|---|---|---|---|
| 1 | `cu_pathname(reassemble.deb)` (`unpack.c:104`) | ~0 | remove file | remove file |
| 2 | `cu_cidir(tmp.ci)` (`:1324`) | ~0 | `path_remove_tree(tmp.ci)` | same |
| 3 | `cu_fileslist` (`:1439`) | ~0 | release tar obstack | same |
| 4 | `cu_prermupgrade(pkg)` (`:1467`, if old ≥ installed-ish) | ~normaltidy | **old postinst `abort-upgrade NEW`**, status via `post_postinst_tasks` | – |
| 5…5+D | `cu_prermdeconfigure` / `ok_prermdeconfigure` per deconfigured pkg (`:330`) | ~normaltidy / normaltidy | **postinst `abort-deconfigure in-favour …`** | enqueue it for configure (`--install` only) |
| … C | `cu_prerminfavour(conflictor, pkg)` per conflictor (`:1494`) | ~normaltidy | **conflictor postinst `abort-remove in-favour …`** | – |
| 6 | one of `cu_preinstverynew/new/upgrade` (`:1514/1519/1527`) | ~normaltidy | **new postrm `abort-install` / `abort-install OLD NEW` / `abort-upgrade OLD NEW`**, restore status (not-installed / config-files / old status), clear reinstreq | – |
| 7 | `cu_closepipe(&p1)` (`:1625`) | bombout | close backend pipe | – |
| 8 | `cu_fileslist` (`:1641`) | ~0 | release pool | same |
| 9…9+N | `cu_installnew(namenode)` per extracted object (`archives.c:1106`) | ~normaltidy | restore `.dpkg-tmp` over the path (removing a new non-atomic object first) or remove the newly placed file; always remove `.dpkg-new` (`cleanup.c:76-135`) | – |
| 10 | `cu_postrmupgrade(pkg)` (`:1671`) | ~normaltidy | **old preinst `abort-upgrade NEW`** | – |
| 11 | checkpoint (`:1691`) | – | everything below now runs as normaltidy | – |
| 12 | checkpoint (`:1799`) | – | | |

So for a package with N extracted objects the stack holds ≈ N + D + C + 9 heap-allocated
entries until the archive's context is popped; unwinding is O(N) and runs in **reverse
extraction order**. After checkpoint 11 an error leaves files in place, the package
half-installed with `reinstreq`, and only the "~0" and "normaltidy" entries run (temp dirs
removed; deconfigured packages re-queued). Verified sequences: scenario A (preinst fail),
E (file conflict), L (postrm and failed-upgrade fail), M (preinst and abort-upgrade fail) in B.4.

Gating counters (`cleanup.c:48-56`): `cu_installnew` increments `cleanup_pkg_failed` and
`cleanup_conflictor_failed` on entry and decrements on exit; every package-level abort
callback does `if (cleanup_pkg_failed++) return;` … `cleanup_pkg_failed--`. Effect: if a file
restore or an abort script fails (ohshit inside a cleanup skips the decrement), **all later
abort scripts for that package are skipped** — verified in L and M where `postinst
abort-upgrade` did not run after `postrm abort-upgrade` failed. `cu_prermdeconfigure` is not
gated. `clear_cleanup_state()` resets the counters per archive (`unpack.c:1304`).

### H.4 Per-package continuation

On longjmp the per-archive/per-package handler pops the context with `ehflag_bombout`
(running H.3), resets `istobe` (process_queue), and `continue`s unless `abort_processing`
(set when `nerrs ≥ --abort-after`, `errors.c:75-79`). The error printer runs **before** the
cleanups (`ehandle.c:270-271`), which is why "error processing archive …" appears before
"executing … for abort-upgrade" (verified). The removal path mirrors this with
`cu_prermremove` before the checkpoint in `removal_bulk_remove_files` (`remove.c:218,297`).

### H.5 Why this is the crux for Rust

* Control flow is non-local: `ohshit` can fire from any depth (libdpkg parsers, I/O helpers,
  `forcibleerr` in conflict checks, `tarobject` called back from `tar_extractor` mid-stream),
  and the undo work is data on a heap stack, not code structure. A Rust port needs (a) a
  `Result`-propagating error type through every layer and (b) an explicit, ordered undo log
  with the same mask/checkpoint/fallback semantics and the gating counters — RAII `Drop` alone
  cannot express "run only on error", "skip if an earlier undo failed", "convert to success
  semantics past a checkpoint", or "errors in undo are reported and unwinding continues".
* Undo actions run maintainer scripts and mutate the DB; their order is user-visible and
  specified by Debian Policy.
* Some state is deliberately kept in `static` variables so cleanup arguments outlive the
  longjmp (`unpack.c:1282-1288`, `archives.c:386-387`, `configure.c:813`); namenodes are arena
  (`nfmalloc`) allocations that are never freed, so cleanup pointers stay valid.
* The fatal-section rule (exit without unwinding while journalling) must be preserved: a Rust
  port that "helpfully" unwinds there could run abort scripts against an inconsistent DB.

---------------------------------------------------------------------------------------------

## I. Tests

`src/at/` autotest (`testsuite.at` includes 10 files; generated `src/at/testsuite`): **80
groups**; the latest run recorded in the build tree reports "All 80 tests were successful"
(`build/src/at/testsuite.log`, a `-j32` run not started by me).

| File | Groups | AT_CHECKs | Covers |
|---|---|---|---|
| `deb-format.at` | 7 | 40 | dpkg-deb option errors; 0.93x format; 2.x core (ar member order/names, truncation, bad headers, premature members, unknown compressors); xz; zstd; bzip2; lzma |
| `deb-content.at` | 2 | 16 | conffiles checks at build; extraction cleanup |
| `deb-fields.at` | 1 | 4 | `-f` field output |
| `deb-streaming.at` | 1 | 6 | reading .deb from stdin/pipes |
| `deb-split.at` | 2 | 15 | dpkg-split options and split format |
| `realpath.at` | 2 | 26 | dpkg-realpath options and resolution |
| `divert.at` | 29 | 159 | dpkg-divert options, queries, add/remove/rename cases, DB error cases (read-only, disk full via `/dev/full`, truncated) |
| `trigger.at` | 1 | 2 | dpkg-trigger `--no-act` |
| `chdir.at` | 33 | 68 | `--root/--instdir/--admindir` + env precedence for dpkg, dpkg-divert, dpkg-statoverride (6 each), dpkg-split, dpkg-query, dpkg-trigger (5 each) — checks only the debug line `root=… admindir=…` |
| `dpkg-arch.at` | 2 | 4 | `--print-architecture`, `--print-foreign-architectures` |

No `src/at` coverage for: dpkg install/unpack/configure/remove/purge/triggers/verify/audit/
selections/avail/compare-versions; dpkg-query output formats; dpkg-statoverride add/remove/list;
dpkg-split `--auto/--listq/--discard/--join` beyond format checks; dpkg-trigger real writes;
`dpkg-maintscript-helper`; `dpkg-db-backup`; `dpkg-db-keeper`; bash completion. The
package-lifecycle paths rely on the root-level `tests/` functional suite: 89 `t-*` directories
(conffile ×22, conflicts/breaks/replaces/provides, disappear ×4, triggers ×10, unpack-*
symlink/hardlink/fifo/device/divert, switch-dir/symlink ×4 using maintscript-helper, filtering,
dry-run, recursive, verify, deb-lfs (5 GiB file), multiarch, db …) — covered by another
analyst. I found no test referencing `dpkg-db-backup`/`dpkg-db-keeper` anywhere (grep).
libdpkg unit tests (`lib/dpkg/t/`) are out of my scope.

---------------------------------------------------------------------------------------------

## J. Rust-port assessment per program

### J.1 Table

| Program | SLOC | Difficulty | Why | Suggested order | Needs from a Rust libdpkg | Specific compatibility risks |
|---|---|---|---|---|---|---|
| dpkg-realpath | 193 | **S** | pure path algorithm, no DB, 26 autotest checks | 1 | options/dirs, error reporting | root clamping of `..`, absolute-link restart, 25-link limit, "includes root prefix" refusal |
| dpkg-trigger | 230 | **S** | one file rewrite under a lock | 2 | trigdeferred parser/writer, `triggers/Lock` (POSIX record lock, blocking), pkg-spec, trigname validation | Unincorp syntax/order; awaiter normalisation (`pnaw_same`); `-` no-await; must interoperate with C dpkg holding the same lock |
| dpkg-split | 1 094 | **S–M** | ar + MD5 + one exec; defensive header parser; no DB | 3 | ar reader/writer, deb-version, parse control (or keep exec of dpkg-deb), MD5 | part file byte format and naming in depot (`%jx.%x.%x`); `--auto` exit 1 for non-parts (dpkg depends on it, `unpack.c:112-131`); preserving or fixing the per-package header bug and the missing MD5 verification is a behaviour decision |
| dpkg-statoverride | 913* | **S** | small, self-contained DB file | 4 | statoverride parse/write (atomic file + backup), sysuser (DPKG_PATH_PASSWD/GROUP), SELinux labelling, force parsing incl. `DPKG_FORCE` | exit 2 for missing override; `#uid` syntax; output order = fs-hash iteration order (unverified whether sorted); no locking |
| dpkg-divert | 715 | **M** | needs read of status + all file lists for ownership/essential checks; rename/copy fallback; best test coverage (29+12 groups) | 5 | full DB read (status, info lists), diversions read/write, fsys hash with diversion links, atomic file | diversions file is a 3-line-record format read by C dpkg; `.distrib` default; `DPKG_MAINTSCRIPT_PACKAGE/ARCH` semantics; "Leaving"/clash messages tested verbatim in `divert.at` |
| dpkg-query | 830 | **M** | thin over libdpkg but output formats are a hard contract | 6 | DB read incl. available, file lists, diversions; `pkg-format` (`--showformat` language and virtual fields), pkg-spec globbing, terminal width/`COLUMNS`, pager, i18n | byte-exact `-l` layout, `-W` default format, `-S` pattern wrapping rules and diversion lines, exit codes 0/1/2, full buffering |
| dpkg-deb | 1 671 | **M** | parsing is straightforward; behaviour depends on GNU tar for build and for all listing/extraction; compression libs | 7 (could be earlier: independent of the DB) | ar, compression (zlib/xz/zstd/bzip2/lzma decompression), tar **writer** (or keep exec of GNU tar), control parsing/validation, pkg-format | byte-for-byte reproducible output (GNU tar `--format=gnu`, `--clamp-mtime`, `--mode`, owner names, order with symlinks last, member names/compressor extension, ar mtime) — any change alters hashes of every built package; `-c` output is GNU `tar -tv` formatting; `--fsys-tarfile` must stream; security boundary for untrusted .debs |
| dpkg read-only actions (`--compare-versions`, `--validate-*`, `--print-*arch*`, `--audit`, `--get-selections`, `--verify`, `--predep-package`, `--assert-*`) | ~2 000 | **S–M** | mostly libdpkg calls + formatting | 8 | version parse/compare, arch list, DB read, md5sums | `--compare-versions` exit codes and warn-but-continue on bad versions (used by every `dpkg-maintscript-helper` call); audit texts |
| dpkg mutating core (`--unpack/--install/--configure/--remove/--purge/--triggers-only`, selections/avail writers, arch add/remove) | ~6 650 | **XL** | 551-line `process_archive`, 492-line `tarobject`, 443-line `depisok`; non-local error unwinding with an explicit undo stack (H); pointer-rich in-memory DB graph mutated across phases; many force-option branches; fsync/rename choreography; trigger and dependtry state machines | last, behind feature flags or as a drop-in tested against the functional suite | essentially all of libdpkg: DB load/journal (`modstatdb_*`), fsys hash with flags/diversions/statoverrides/trigger interests, tar reader, triggers library, dependency structures with reverse links, file locks, atomic files, subprocess helpers, SELinux, error/undo framework | see J.2 |

\* includes the shared `force.c`/`selinux.c`.

### J.2 Compatibility risks for the dpkg core (places a regression could brick a system or corrupt the DB)

1. **Unwind semantics** (H.3): order of abort scripts, which ones are skipped after a failed
   undo, success-time fallbacks (`ok_prermdeconfigure`), checkpoint placement relative to
   `pkg_remove_old_files`/infodb update; a mistake leaves packages half-installed with wrong
   files or runs postinst against new files.
2. **Ordering inside `process_archive`**: new preinst before old postrm; disappear checks after
   file-list writes but before claiming ownership; status `unpacked` only after
   `pkg_remove_files_from_others` (comment `unpack.c:1768-1774`). Moving any of these changes
   which file a later removal deletes.
3. **Shared/overwritten file ownership**: Replaces/Conflicts logic in `tarobject`, M-A:same
   refcounting, `filesavespackage` disappearance, `write_filelist_except` on other packages —
   errors here cause later removals to delete files other packages need (e.g. libc).
4. **Directory/symlink rules**: never replacing dir↔symlink, following existing symlinked
   directories (merged-/usr), `existingdir` tests with `stat` vs `lstat`; a "safer" port that
   refuses to traverse existing symlinks would break merged-/usr systems.
5. **Durability choreography**: write-all → `sync_file_range(WAIT_BEFORE)` → fsync → rename,
   `.dpkg-tmp` backups and recovery-by-rename on the next run (`archives.c:845-873`); DB journal
   fsyncs; fatal-section exits. (Conversely the conffile/parent-dir fsync gaps could be closed.)
6. **Conffile decisions** (B.6 table), dereferencing symlinked conffiles, `.dpkg-old/.dpkg-dist`
   naming, MD5 sentinels `newconffile/nonexistent/-`, and the status-fd prompt line (apt/debconf
   frontends react to it — (unverified which)).
7. **dependtry/trigger heuristics**: thresholds `2·len+2`/`3·len+2`, `progress_bytrigproc`,
   cycle-break preference for packages without postinst, trigger-cycle tortoise/hare — these
   define configuration order and therefore which postinst sees which dependencies configured.
8. **Locking/protocol** with C tools still installed during a transition: same fcntl record
   locks, same file formats, `DPKG_FRONTEND_LOCKED`, `DPKG_FORCE` propagation, diversion reload
   after each script.
9. **CLI/config parsing quirks** (no permutation, config files may set actions, unknown config
   option is fatal, `-G`, `--force-<x>` suffixes) and exit codes 0/1/2.
10. **Environment for maintainer scripts** (C.5) and chroot/chrootless handling of
    `DPKG_ROOT/DPKG_ADMINDIR`.
11. Latent issues a port must decide to keep or fix: `..` members escaping `--root` (G.6),
    unguarded `canfixbytrigaw` (`depcon.c:548`), dpkg-split join without MD5 check,
    `--listq` header bug, "exit code 256" hook message, dead `commandfd`.

### J.3 What makes a port tractable (evidence)

* **Process boundaries already exist**: dpkg talks to dpkg-deb, dpkg-split and dpkg-query only
  through argv/exit codes/pipes (G.1) and never decompresses in-process (only dpkg-deb links
  compression libs). Each program can be replaced independently and verified against the
  other C programs.
* **Small leaf programs** (realpath, trigger, split, statoverride, divert, query: 193–1 094
  SLOC each) with clear file formats and, for divert/realpath/split/deb, autotest suites whose
  expected outputs are literal text.
* **Specs and docs**: `doc/spec/triggers.txt`, `frontend-api.txt`, `protected-field.txt`, man
  pages documenting status-fd, log format, env vars, exit codes, file naming, `deb(5)` limits.
* **Functional suite**: 89 root `tests/` cases exercising unpack/configure/remove/conffiles/
  triggers/diversions/multiarch end to end, plus 80 autotest groups (all passing in this build),
  usable as a differential oracle (run both implementations, compare DB + filesystem).
* **Deterministic, observable behaviour**: status-fd/log/stdout give a precise trace;
  experiments in this note reproduced every maintainer-script sequence non-root in a scratch
  `--root` within seconds, which suggests a cheap differential-testing harness is feasible.
* **Clean separation of decision vs effect in places**: `wanttoinstall`, `depisok`,
  `dependencies_ok`, `conffoptcells` are pure functions over the in-memory DB and could be
  ported and property-tested in isolation.
* **Static linking of libdpkg** into each binary means there is no shared-library ABI to
  preserve for these programs (libdpkg's public API for third parties is a separate question
  for the libdpkg analyst).

---------------------------------------------------------------------------------------------

## Appendix 1 — Experiment log (all non-root, inside the scratchpad)

Setup: test packages built with the built `dpkg-deb --root-owner-group` and
`SOURCE_DATE_EPOCH=1700000000`; maintainer scripts append `"$name-$ver $script $*"` plus env to
`$DPKG_ROOT/../trace.log` and fail when listed in `$FAIL`. dpkg invoked as
`dpkg --root=$E/root --force-not-root --force-script-chrootless --no-debsig …` with
`PATH=$B:/usr/sbin:/sbin:$PATH`.

* Install/upgrade sequences, status-fd and log lines — B.4 table and C.3/C.4. Sample log:
  ```
  2026-10-02 14:11:21 startup archives install
  2026-10-02 14:11:21 install foo:all <none> 1.0
  2026-10-02 14:11:21 status half-installed foo:all 1.0
  2026-10-02 14:11:21 status unpacked foo:all 1.0
  2026-10-02 14:11:21 configure foo:all 1.0 1.0
  2026-10-02 14:11:21 status half-configured foo:all 1.0
  2026-10-02 14:11:21 status installed foo:all 1.0
  ```
* Failure scenarios A–D, L, M (upgrade unwind), E (file conflict), F (Conflicts+Replaces),
  G/H (Breaks with/without `-B`), I (disappear), J (remove/purge), N/O (remove failures), P
  (`--abort-after=1`), K1–K3 (conffile prompt EOF / confold / confnew).
* Triggers: explicit awaited trigger and noawait file trigger (B.9); `triggers/File` contained
  `/usr/share/watched tgt/noawait`; `triggers/mytrig` contained `tgt`.
* strace of `-i foo_1.0` (2 regular files, 1 conffile): execs as in G.1; 37 fsync, 4
  sync_file_range, 30 rename, 13 link, 0 chroot (chrootless); locks fd3 `lock-frontend` and fd4
  `lock` via `F_SETLK`, `triggers/Lock` via `F_SETLKW`. `-y` decoding: `data-1.0.dpkg-new`
  fsynced; `foo.conf.dpkg-new` only sync_file_range'd. `--force-unsafe-io`: 36 fsync, no
  WAIT_BEFORE.
* dpkg-deb: reproducible builds with SOURCE_DATE_EPOCH (identical sha256), non-reproducible
  without; exec'd tar argv captured; `--no-uniform-compression -Zzstd` gives
  `control.tar.gz` + `data.tar.zst`.
* Path traversal: crafted `./../ESCAPED-by-dpkg` member — `dpkg-deb -c` lists it, `dpkg-deb -x`
  fails (GNU tar), `dpkg -i --root` writes outside the root (G.6).
* dpkg-split: `--listq` duplicate header; join of a tampered part succeeds with exit 0.
* Exit codes listed in C.2; config-file action quirk (B.2); hooks and status-logger (C.3/C.5).

## Appendix 2 — Not verified / open questions

* Which dpkg flags and output lines apt (and other frontends) depend on beyond the lock
  protocol and the trigproc comment; whether frontends parse human stdout.
* Whether `depcon.c:548` NULL dereference is reachable in practice.
* Behaviour of the chroot (non-chrootless) path and SELinux labelling — code-read only (tests
  ran non-root, chrootless, SELinux disabled).
* `--path-exclude/--path-include`, `-R/--recursive`, `--record-avail`, `--predep-package`,
  `--verify` output and `--set-selections` — code-read only.
* Old-format (0.939000) archives read from a non-seekable stdin (data length computed from
  `ar->size`, which is 0 for pipes, `extract.c:288`) — not exercised.
* dpkg-statoverride/dpkg-divert listing order (fs-hash iteration) — not checked for
  determinism.
* debhelper's use of dpkg-maintscript-helper (outside the repo).
* Performance characteristics (per-archive `dpkg-split` and two `dpkg-deb` execs; O(N) cleanup
  entries) — not measured.
