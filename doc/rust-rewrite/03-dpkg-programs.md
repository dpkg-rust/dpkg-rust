# 03 — The C programs: dpkg, dpkg-deb and friends

This page describes the programs in `src/` and the two standalone utilities in `utils/`:
what each does, how `dpkg` installs, configures and removes a package, and which
externally visible behaviour any replacement has to keep.

Line-level detail and citations: [reference/02-src-programs.md](reference/02-src-programs.md)
(and section F of [reference/05-build-tests-history.md](reference/05-build-tests-history.md)
for `utils/`).

## 1. Inventory

SLOC means lines without comments and blanks.

| Program | Purpose | Sources | SLOC | Links libdpkg | Runs these programs |
|---|---|---|---|---|---|
| `dpkg` | Install, configure, remove, purge, process triggers; selections, audit, version comparison; forwards query and archive actions | `src/main/` (23 files), `src/common/` | 9,227 | yes | `dpkg-split`, `dpkg-deb` (twice per archive), maintainer scripts, hooks, `diff`, pager, `rm -rf`, optionally `debsig-verify` |
| `dpkg-deb` | Build, inspect and extract `.deb` archives | `src/deb/` | 1,671 | yes | GNU `tar` |
| `dpkg-split` | Split a `.deb` into parts, join them, reassemble parts automatically | `src/split/` | 1,094 | yes | `dpkg-deb --info` |
| `dpkg-query` | Read-only database queries | `src/query/main.c` | 830 | yes | pager |
| `dpkg-divert` | Manage file diversions | `src/divert/main.c` | 715 | yes | — |
| `dpkg-statoverride` | Manage ownership and mode overrides | `src/statoverride/main.c` | 470 (+ shared) | yes | — |
| `dpkg-trigger` | Activate a trigger from a maintainer script | `src/trigger/main.c` | 230 | yes | — |
| `dpkg-realpath` | Resolve a path inside an alternative root | `src/realpath/main.c` | 193 | yes | — |
| `update-alternatives` | Manage the `/etc/alternatives` symlink groups | `utils/update-alternatives.c` | 3,369 lines | **no** | `mv` (fallback) |
| `start-stop-daemon` | Start and stop daemons from init scripts | `utils/start-stop-daemon.c` | 3,148 lines | **no** | the daemon |
| `dpkg-maintscript-helper` | Shell helper for conffile and directory/symlink transitions | `src/dpkg-maintscript-helper.sh` | 533 | — | `dpkg`, `dpkg-query`, `dpkg-realpath`, coreutils |
| `dpkg-db-backup`, `dpkg-db-keeper` | Daily database backup; optional git history of the database | `src/*.sh` | 68 | — | `tar`, `savelog`, `git` |

Within `dpkg`, the code that changes packages (unpack, configure, remove, dependency checks,
triggers, script execution, error unwinding) is 6,655 SLOC. The three longest functions in
the tree are there: `process_archive()` (551 lines, `src/main/unpack.c:1271`), `tarobject()`
(492 lines, `src/main/archives.c:739`) and `depisok()` (443 lines, `src/main/depcon.c:320`).

## 2. The programs talk through processes, not function calls

`dpkg` never opens a `.deb` itself. For each archive it forks and executes:

```
dpkg ── dpkg-split -Qao <admindir>/reassemble.deb X.deb     is this a split part?
     ── [debsig-verify -q X.deb]                             only if installed and not --no-debsig
     ── dpkg-deb --control X.deb <admindir>/tmp.ci           control files (dpkg-deb runs tar)
     ── <admindir>/tmp.ci/preinst ...                        new package's scripts run from tmp.ci
     ── dpkg-deb --fsys-tarfile X.deb ──pipe──> dpkg         dpkg parses the tar stream itself
     ── <admindir>/info/<pkg>.postrm ...                     old package's scripts run from info/
     ── <admindir>/info/<pkg>.postinst configure ...
```

Fourteen `dpkg` actions (`-l`, `-s`, `-L`, `-S`, `-p`, `-b`, `-c`, `-e`, `-I`, `-f`, `-x`, `-X`,
`--ctrl-tarfile`, `--fsys-tarfile`) are not implemented in `dpkg` at all: it replaces itself
with `dpkg-query` or `dpkg-deb` through `exec`. Before doing so it exports `DPKG_ADMINDIR`,
`DPKG_ROOT` and `DPKG_FORCE`, which is how the back-ends and every maintainer script learn
about `--root`, `--admindir` and the force options.

`dpkg-deb` in turn executes GNU `tar` to create both archive members when building and for
every listing or extraction. Only `dpkg-deb` links the compression libraries.

These process boundaries (argument vectors, pipes, exit codes, files on disk) are the
natural seams of the system. Each program can be replaced and tested on its own.

## 3. Package states

```
not-installed < config-files < half-installed < unpacked < half-configured
              < triggers-awaited < triggers-pending < installed
```

Plus an error flag (`reinstreq`, "reinstallation required") and the administrator's
*selection* (`install`, `hold`, `deinstall`, `purge`). Every state change is journalled
through `modstatdb_note()` and reported as `status: <pkg>: <state>` on the status
file descriptors.

Observed sequences (run against a scratch root during this study):

| Operation | States |
|---|---|
| fresh install | half-installed → unpacked → half-configured → installed |
| upgrade | half-configured → unpacked → half-installed → unpacked → half-configured → installed |
| remove | half-configured → half-installed → config-files |
| purge | … → not-installed |

## 4. How dpkg installs a package

`dpkg -i a.deb b.deb` unpacks every archive first and then configures everything that was
unpacked. Each archive and each queued package runs inside its own error context, so a
failure is reported and processing continues with the next one, up to `--abort-after`
(default 50) errors.

### 4.1 Unpack: `process_archive()`

The numbered phases below are in source order (`src/main/unpack.c:1271-1821`).

| # | Phase |
|---|---|
| 1 | Reassemble split parts if needed (`dpkg-split`), optionally verify with `debsig-verify` |
| 2 | Extract control files into `tmp.ci/` (`dpkg-deb --control`), fsync them, parse `control` strictly |
| 3 | Check architecture, selection state, downgrade rules |
| 4 | Check Conflicts, Breaks and Pre-Depends; decide which other packages must be deconfigured or removed |
| 5 | Load all file lists; print "Preparing to unpack"; activate triggers |
| 6 | Read the new `conffiles` list; mark old conffiles |
| 7 | **Old `prerm upgrade`** (on failure: new `prerm failed-upgrade`) |
| 8 | Deconfigure other packages (`prerm deconfigure in-favour …`); `prerm remove in-favour …` for conflicting packages being removed |
| 9 | Set `reinstreq` and half-installed; **new `preinst install` or `upgrade`** |
| 10 | Start `dpkg-deb --fsys-tarfile` and extract each tar member through `tarobject()` (section 4.2) |
| 11 | Batched fsync and rename of all new files |
| 12 | **Old `postrm upgrade`** (on failure: new `postrm failed-upgrade`) |
| 13 | **Checkpoint: the point of no return** |
| 14 | Remove files of the old version that the new one no longer ships, in reverse path order |
| 15 | Write the new `.list`, update trigger interests, move control files into `info/`, write `.md5sums` |
| 16 | Copy "available" metadata to "installed"; record conffiles |
| 17 | Packages whose files have all been taken over "disappear" (`postrm disappear`) |
| 18 | Remove taken-over files from other packages' lists |
| 19 | Status **unpacked**; delete the `.dpkg-tmp` backups; clear `reinstreq` |
| 20 | Finish removal of conflicting packages |

### 4.2 One file at a time: `tarobject()`

For every tar member (`src/main/archives.c:739-1230`):

1. Map the tar user and group names to local ids; reject names containing a newline.
2. Find or create the path node; honour diversions and statoverrides; fire file triggers.
3. If the path exists, decide what to do. A directory is never replaced by a symlink, nor a
   symlink to a directory by a directory (this is why `dpkg-maintscript-helper` has
   `symlink_to_dir` and `dir_to_symlink`). Existing symlinked directories are followed,
   which is what makes merged-`/usr` systems work.
4. If another package owns the path: allow it for co-installable Multi-Arch instances with
   identical content, for `Replaces`, for packages being removed and for directories;
   otherwise fail unless `--force-overwrite`.
5. Skip the member if `--path-exclude` says so.
6. Extract to `<path>.dpkg-new`. Regular files are created exclusively with mode 0,
   preallocated when large, hashed with MD5 while being written, then given their owner,
   mode, timestamp and SELinux context.
7. Conffiles stop here and stay as `.dpkg-new` until configuration.
8. Back up the existing object as `<path>.dpkg-tmp` (a hard link for files).
9. Directories, devices and FIFOs are renamed into place immediately. Regular files,
   hard links and symlinks are queued.

After the whole stream has been read, `tar_deferred_extract()` asks the kernel to start
writeback on every queued file, waits, then fsyncs and renames each one. Batching the I/O
this way is a performance design; `--force-unsafe-io` skips the fsyncs.

The tar reader itself (in libdpkg) creates symlinks only after all other members, so an
archive cannot plant a symlink and then write through it.

### 4.3 Configure

`deferred_configure()` (`src/main/configure.c`) checks dependencies, then for each conffile
compares three MD5 hashes: the one recorded at the last install, the file on disk, and the
new version shipped by the package.

| User edited? | Package changed? | Action |
|---|---|---|
| no | no | keep |
| no | yes | install the new version |
| yes | no | keep |
| yes | yes | prompt (default: keep) |

`--force-confnew`, `--force-confold`, `--force-confdef`, `--force-confmiss` and
`--force-confask` override the table. The loser is kept as `.dpkg-old` or `.dpkg-dist`.
Then dpkg runs **`postinst configure <previous-version>`** and marks the package installed
(or triggers-pending / triggers-awaited).

### 4.4 Ordering and dependency cycles

Packages to configure or remove sit in a FIFO queue (`src/main/packages.c`). A package whose
dependencies are not ready is put back. When the queue stops making progress, dpkg raises
an escalation level called `dependtry`:

| Level | What becomes allowed |
|---|---|
| 1 | normal processing |
| 2 | break dependency cycles (preferring to cut at a package without a postinst) |
| 3 | process triggers of queued packages to unblock dependencies |
| 4 | check for trigger cycles when deferring |
| 5 | ignore version mismatches if `--force-depends-version` |
| 6 | ignore everything if `--force-depends` |

These heuristics decide the order in which postinst scripts run, so they are behaviour, not
implementation detail.

### 4.5 Remove and purge

`prerm remove` → delete files in reverse order, keeping conffiles and shared directories →
`postrm remove` → state config-files. Purge additionally deletes conffiles and their backup
variants (`~`, `.bak`, `.dpkg-old`, `.dpkg-dist`, …), runs `postrm purge`, and removes the
remaining `info/` files.

### 4.6 Triggers

Packages declare interest in named triggers or in paths. Activations are recorded while
other packages are processed and run later as `postinst triggered "<names>"`. dpkg detects
trigger cycles by keeping snapshots of each package's pending set and applying a
tortoise-and-hare comparison; on a cycle the earliest package is marked half-configured
and abandoned. The model is specified in `doc/spec/triggers.txt`.

### 4.7 Maintainer scripts

Scripts are executed directly (not through a shell) after an optional `chroot()` into the
installation root. They receive `DPKG_MAINTSCRIPT_PACKAGE`, `DPKG_MAINTSCRIPT_ARCH`,
`DPKG_MAINTSCRIPT_NAME`, `DPKG_MAINTSCRIPT_PACKAGE_REFCOUNT`, `DPKG_MAINTSCRIPT_DEBUG`,
`DPKG_RUNNING_VERSION`, plus `DPKG_ADMINDIR`, `DPKG_ROOT` and `DPKG_FORCE`. dpkg ignores
SIGINT and SIGQUIT only while waiting for a script. After each script it reloads the
diversions file and incorporates new trigger activations, because the script may have
called `dpkg-divert` or `dpkg-trigger`.

The complete call sequences, verified by running them:

| Scenario | Sequence |
|---|---|
| fresh install | new `preinst install` → unpack → new `postinst configure ""` |
| upgrade | old `prerm upgrade NEW` → new `preinst upgrade OLD NEW` → unpack → old `postrm upgrade NEW` → new `postinst configure OLD` |
| old prerm fails | new `prerm failed-upgrade OLD NEW`, then continue |
| new preinst fails during upgrade | new `postrm abort-upgrade OLD NEW` → old `postinst abort-upgrade NEW` |
| error while unpacking a fresh install | files rolled back → new `postrm abort-install` |
| Conflicts + Replaces | other `prerm remove in-favour PKG VER` → new `preinst install` → unpack → other `postrm remove` → new `postinst configure` |
| Breaks with auto-deconfigure | other `prerm deconfigure in-favour PKG VER` → install → other package re-queued for configuration |
| disappearing package | new `preinst` → unpack → other `postrm disappear PKG VER` → new `postinst configure` |
| remove | `prerm remove` → files removed → `postrm remove` |
| prerm fails on remove | `postinst abort-remove`; package stays installed |

## 5. Error unwinding in the install path

This is the hardest part of dpkg to reason about and the hardest to port.

While `process_archive()` runs, it pushes entries on the cleanup stack of the per-archive
error context (the mechanism is described in [02-libdpkg.md](02-libdpkg.md#3-error-handling-the-defining-design-decision)).
For an upgrade the stack looks like this, oldest first:

| Entry | Runs when | Effect |
|---|---|---|
| remove `reassemble.deb` | always | delete temporary file |
| remove `tmp.ci` | always | delete control staging directory |
| `cu_prermupgrade` | error only | old `postinst abort-upgrade NEW` |
| `cu_prermdeconfigure` / `ok_prermdeconfigure` (one per deconfigured package) | error / success | `postinst abort-deconfigure …` / re-queue for configuration |
| `cu_prerminfavour` (one per conflicting package) | error only | `postinst abort-remove in-favour …` |
| `cu_preinst…` | error only | new `postrm abort-install` or `abort-upgrade`; restore the old status |
| close pipe | error only | close the `dpkg-deb` pipe |
| `cu_installnew` (**one per extracted file**) | error only | restore `.dpkg-tmp` over the path, delete `.dpkg-new` |
| `cu_postrmupgrade` | error only | old `preinst abort-upgrade NEW` |
| **checkpoint** | — | from here on, older entries run as if processing had succeeded |

So a package with N files has about N + 9 heap-allocated undo entries, and an error
anywhere (in a parser, an I/O helper, a conflict check, or inside the tar callback) jumps
back to `archivefiles()`, which runs them newest-first: files are restored in reverse
extraction order and the abort scripts run in the order Debian Policy prescribes.

Details that matter:

- **Gating.** Each undo step increments a counter on entry and decrements it on exit. If a
  file restore or an abort script itself fails, the counter stays raised and all later
  abort scripts for that package are skipped.
- **The checkpoint.** After it, an error leaves the new files in place and the package
  half-installed with `reinstreq` set. Nothing is rolled back.
- **Fatal sections.** Journal writes run in a mode where any error terminates the process
  immediately. A port that unwinds there instead would run abort scripts against a database
  in an unknown state.
- **Static storage.** Several locals in `process_archive()` are `static` because their
  addresses are handed to cleanup entries that outlive the `longjmp`.
- **Signals.** There are no signal handlers. A SIGINT or SIGTERM outside a maintainer
  script kills dpkg without unwinding; recovery relies on the journal, the `reinstreq`
  flag, and the `.dpkg-new`/`.dpkg-tmp` files that the next run knows how to pick up.

## 6. dpkg-deb

**Build** (`src/deb/build.c`). Validates `DEBIAN/control`, script permissions and the
conffiles list; walks the tree in sorted order with symlinks moved to the end; pipes the
file list to GNU `tar -cf - --format=gnu --mtime @<ts> --clamp-mtime --null --no-recursion
-T -`; compresses in a forked child; writes the `ar` container with members
`debian-binary`, `control.tar.<ext>`, `data.tar.<ext>`. With `SOURCE_DATE_EPOCH` set the
output is byte-for-byte reproducible (verified). Default compressor: xz.

**Extract** (`src/deb/extract.c`). One function reads the `ar` container defensively
(member order, names, sizes, format version), then builds a pipeline of three children:
copy the member → decompress → GNU `tar`. For `--fsys-tarfile` and `--ctrl-tarfile` the tar
stage is omitted and the raw tar stream goes to stdout.

**Formats.** `.deb` 2.0 and the ancient 0.939000 format. `ar` member sizes are ten decimal
digits, so a member is limited to just under 10 GB.

**Security stance.** Upstream documents `dpkg-deb` as a boundary for *examining* untrusted
packages, and says *installing* untrusted packages "must never be done", since maintainer
scripts run as root anyway.

## 7. The smaller programs

- **dpkg-query**: `-l` (table sized to the terminal), `-W` with `--showformat`, `-s`, `-p`,
  `-L`, `-S` (glob search over every known path), `--control-path/-list/-show`. Takes no
  lock. Exit status 1 when anything was not found.
- **dpkg-divert**: `--add` (default action), `--remove`, `--list`, `--listpackage`,
  `--truename`; optional `--rename` with a cross-filesystem copy fallback. Best tested of
  the small tools (29 autotest groups).
- **dpkg-statoverride**: `--add`, `--remove`, `--list`, optional `--update` to apply the
  override immediately.
- **dpkg-trigger**: appends an activation to `triggers/Unincorp` under `triggers/Lock`.
- **dpkg-split**: 450 KiB parts by default; `--auto` keeps parts in `<admindir>/parts` until
  a set is complete. `dpkg` calls it for every archive it installs.
- **dpkg-realpath**: symlink resolution clamped to a root directory; used by
  `dpkg-maintscript-helper`.

`dpkg-divert` and `dpkg-statoverride` rewrite database files **without taking the dpkg
lock**. They rely on being called from maintainer scripts while `dpkg` holds it.

- **update-alternatives** keeps one text file per link group under
  `<admindir>/alternatives/` and maintains the two-level symlinks
  `<link>` → `/etc/alternatives/<name>` → `<choice>`. It has its own error handling and a
  712-assertion black-box test suite that has already carried it through two rewrites
  (a complete rewrite in Perl in 2009, then Perl → C in 2010).
- **start-stop-daemon** is mostly process inspection with a variant per operating system
  (Linux `/proc`, Solaris and AIX `psinfo`, Hurd libps, macOS libproc, HP-UX pstat, FreeBSD
  sysctl, generic kvm): 104 conditional blocks across 10 OS families. It has no tests at
  all and is the most frequently changed C file of the last five years.

## 8. Contracts a replacement must keep

| Contract | Summary |
|---|---|
| Command line | dpkg has 44 actions and 32 options, plus 29 force names, 13 debug flags, 16 comparison operators. The parser stops at the first non-option, accepts `--opt=value` and `--opt value`, bundles short options, and has the `--force-<x>` suffix form |
| Configuration | `/etc/dpkg/dpkg.cfg.d/*`, `/etc/dpkg/dpkg.cfg`, `~/.dpkg.cfg`; an unknown option is fatal |
| Exit codes | 0 success; 1 some package failed, or a check returned false; 2 fatal or usage error |
| `--status-fd` | `status: <pkg>: <state>`, `processing: <stage>: <pkg>`, `status: <pkg> : error : <msg>`, `status: <file> : conffile-prompt : '<old>' '<new>' <useredited> <distedited>` |
| Log | `YYYY-MM-DD HH:MM:SS <startup|status|install|upgrade|configure|trigproc|remove|purge|conffile> …` |
| Locks | `lock-frontend` then `lock`, `fcntl` record locks; `DPKG_FRONTEND_LOCKED` (see `doc/spec/frontend-api.txt`) |
| Hooks | `--pre-invoke`, `--post-invoke` (with `DPKG_HOOK_ACTION`), `--status-logger` |
| Script environment | the `DPKG_MAINTSCRIPT_*` variables, `DPKG_RUNNING_VERSION`, chroot or chrootless behaviour |
| File names | `.dpkg-new`, `.dpkg-tmp`, `.dpkg-old`, `.dpkg-dist`, `.dpkg-divert.tmp` |
| Output formats | `dpkg-query -l/-W/-S/-L`, `dpkg --get-selections`, `dpkg --verify`, `dpkg-deb -c/-I/-f` |
| `.deb` bytes | reproducible output depends on the exact tar invocation, member order and compressor settings |

Which of these apt and other front-ends actually depend on, beyond the documented lock
protocol, was not verified from apt's source.

## 9. Test coverage of the programs

- `src/at/` (autotest): 80 groups covering `dpkg-deb` formats and errors, `dpkg-split`,
  `dpkg-realpath`, `dpkg-divert`, and the `--root`/`--admindir` option handling of all
  programs. Nothing there installs a package.
- `tests/` (functional suite): 89 scenario directories covering the package life cycle,
  conffiles, triggers, diversions, Multi-Arch, file conflicts and directory/symlink
  switches. This is what actually tests `dpkg -i`.
- `utils/t/`: 712 assertions for `update-alternatives`.
- No tests: `start-stop-daemon`, `dpkg-db-backup`, `dpkg-db-keeper`, bash completion, and
  `dpkg-statoverride` beyond option handling.

See [05-build-test-release.md](05-build-test-release.md) for how the suites are run.
