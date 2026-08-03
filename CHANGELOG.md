# Changelog

All notable changes to this project are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); the project uses a
plain `major.minor` version printed by `--version`.

## [2.6] — 2026-08-03

The first run of the full suite on Linux, and the review that went with it. Two
of the three things it turned up were real; the third was the harness lying
about the tool.

### Fixed

- **The orphan sweep killed live servers.** A process is an orphan when the
  install it runs out of is gone, but only the *first* server path in its
  arguments was ever looked at. A process whose argv happens to name a tree an
  earlier run removed before the tree it is actually running out of was
  therefore declared an orphan and sent `SIGTERM`, then `SIGKILL` — a live
  session taken down by the pass that exists to clean up after dead ones. Every
  path in the arguments is examined now, and one that still exists spares the
  process. Reached only with `IDE_PROTECT_LIVE=0`, which is the interactive
  default; unattended runs never ran this pass at all.
- **An unreadable `lru.json` ended the run before its summary.** The file was
  checked for being writable and never for being readable, and the read that
  followed is a plain assignment — so under `set -e` a file in mode 200 stopped
  the sweep where it stood, with a raw shell error and every later pass
  skipped. The same failure mode as the unguarded `grep` fixed in 2.5, arriving
  through a redirect instead of a pipeline, which is why the audit for it did
  not see it. Readability is asked about, the read is guarded, and an answer
  that does not arrive fails the shape test into "left untouched".

### Changed

- `--version` reports 2.6.
- A temp directory that `TMPDIR` and `/tmp` both resolve to is now named once
  when it is refused, not twice: the duplicate check runs before the screen
  rather than after it.

### Tests

`selftest` grew from 104 to 108 checks, and the suite now runs green on Linux —
Linux 6.8, x86_64, bash 5.2.21, GNU coreutils 9.4, findutils 4.9.0 — with no
skips. New coverage for all three changes above, the duplicate temp directory
included: `HOME` and `TMPDIR` both at `/tmp` is the one arrangement in which the
two spellings resolve to a directory that is also refused, and the check counts
the refusals rather than looking for one. The stub standing in for a working
GNU `stat` is portable at last: it delegated to the BSD spelling, which GNU
coreutils answers with an error rather than an mtime, so that check had been
failing on every Linux host for a reason that had nothing to do with the tool.

## [2.5] — 2026-08-03

A second review pass before the first public release, this one adversarial about
the guards themselves rather than about what they are pointed at. One of them
turned out never to have worked on macOS.

### Fixed

- **`path_in_use()` answered "free" for every directory on any host without
  `/proc`.** It read `lsof`'s exit status, and `lsof +D` exits non-zero after a
  walk it completed perfectly well — the same status it uses for "nothing is
  open". So the promise that a temp directory is removed only when no process
  holds it was not kept for directories at all, on every macOS host and every
  Linux one where `/proc` was unavailable. Individual files were unaffected. The
  output is now the answer: `-F` prints one machine-readable line per hit and
  nothing when there are none.
- **`--protect-live` now covers an unfinished `.staging` download.** The flag
  spares a running server by keeping its commit, and no commit is ever recorded
  for a version nobody could start yet — so a staging directory had nothing to
  be held by, and a transfer stalled past `IDE_STAGING_MIN_AGE_HOURS` was
  terminated and removed even under the flag, including on the unattended path
  where that flag is the default. The path itself is now asked about.
- **Session log directories are asked whether anything still writes to them.**
  `data/logs/*` was dropped on age alone, while the CLI logs beside it had
  always been checked. The retention window is short enough for a session to
  outlive it.
- **`sweep_orphan_processes()` honours `IDE_KILL_GRACE_SECONDS`.** It signalled
  inside its loop with a hard one-second pause, which both ignored the setting
  and served the pause once per orphan. Orphans are now collected first and the
  grace period served once for the whole set, as `terminate_matching()` does.
- **A short but legal temp path is no longer refused as being outside the
  roots.** `is_safe_target()` required more than ten characters, and
  `/tmp/code-` under a widened `IDE_TMP_PATTERNS` is exactly ten — refused with
  a message about roots that had nothing to do with it, and counted as an error
  the run then exited on. The floor is now six, the shortest path the shortest
  acceptable temp directory can carry.
- **`is_safe_target()` no longer resolves the parent through `abs_dir()`**,
  which answers with its own argument when it cannot resolve one. That fallback
  is right for reporting and wrong for a guard: an unresolvable parent was
  compared literally and so passed the check it was meant to fail.
- **`size_kb()` measures a path whose name contains a newline.** `du -s` breaks
  its line on such a name, and reading the last line of that output measured the
  tail of the *name*, so every such path counted as 0 KB in the freed total.

### Changed

- `--version` reports 2.5.
- `--help` and the README now agree on where the parenthesis in the `--keep`
  description closes.

### Tests

`selftest` grew from 96 to 104 checks. New coverage: a directory held open in
`/tmp` and under `data/logs`, each paired with an identical one nobody holds —
the pairing is the point, since the old in-use test passed the "removed" half
and failed only the half that was never written; a stalled `.staging` download
with a live process under `--protect-live` and under `--no-protect-live`; an
orphan process actually being signalled rather than only reported; and a temp
name containing a newline being measured rather than counted as nothing.

## [2.4] — 2026-08-02

A safety and correctness pass over 2.3, prompted by a full review before the
first public release. Nothing about what the tool decides to remove has changed;
what changed is when it stops, what it refuses to adopt, and what it does before
it signals anything.

### Fixed

- **A run no longer ends silently on an empty `lru.json`.** `grep` reports a file
  with no matches with a non-zero status, and under `set -o pipefail` that status
  became the surrounding assignment's, which `set -e` then turned into an exit —
  mid-sweep, with no message, no summary line and every later pass skipped. The
  trigger was self-inflicted: the tool writes `[]` itself once the last install
  is gone, so any host that had been swept clean once (`--keep 0`, or simply an
  eviction of everything) failed this way on every run afterwards, reporting only
  exit status 1. Both counting pipelines now tolerate "no match".
- **Nothing is signalled for a target the containment check refuses.** A stale
  install is terminated on the way to being deleted, but only the removal
  consulted `is_safe_target`. A path reached through a symlinked parent inside
  the tree was therefore sent `SIGTERM`, and `SIGKILL` after the grace period,
  before the removal that followed was refused. The gate is now asked first, by
  both callers.
- **`bin` plus `data` alone no longer identifies a server root.** Auto-detection
  reaches every `~/.*-server` and `~/.*-server-*` by name, and that pair of
  directory names is far too ordinary a shape: an unrelated tree matching it was
  adopted, after which the unconditional cache and log passes swept its
  `data/logs` and `data/CachedProfilesData`. The legacy layout is now recognised
  by a mark the editor actually leaves — an install named after a commit, or
  `data/User` / `data/Machine`. `cli/servers` still qualifies on its own.
- **`--install` after `-r` no longer widens the scheduled run.** `-r` used to be
  written into the unit as `IDE_SERVER_ROOTS`, which *adds* to the auto-detected
  set where `-r` *replaces* it — so the timer swept trees the run tried by hand
  never looked at. Explicit roots now travel as `--root` arguments in
  `ExecStart` / `ProgramArguments`, quoted so a path with a space survives.
- **An unusable `HOME` is named rather than stumbled over.** Every root, guard
  and the lock itself hang off `$HOME`; without one the script failed with a bare
  `HOME: unbound variable` from the middle of the file. It now refuses up front,
  before anything else runs.
- **A system directory in `TMPDIR` is refused.** `TMPDIR` becomes a removal root
  for the rest of the run, and the previous screening — not `/`, not `$HOME`, not
  a parent of `$HOME`, at least four characters — let `/usr`, `/etc`, `/var` and
  their kind straight through. They are now refused by name. Paths resolving
  below them are unaffected: macOS hands out `/private/var/folders/…`, and that
  is the normal answer.
- **`--install` no longer copies itself through a symlink** left on
  `~/.local/bin/ide-server-prune`. `cp` writes through a link rather than
  replacing it; the copy is now staged under a name of its own and moved into
  place.
- **The printed cron line survives a shell.** Values were wrapped in single
  quotes without escaping, so an apostrophe inside one closed the quoting and
  handed the rest of the line to cron as something else.

### Changed

- `--version` reports 2.4.
- `--install` now lists any `--root` arguments it carried over alongside the
  tunables it carried over.

### Tests

`selftest` grew from 71 to 96 checks. New coverage: an empty `lru.json` and the
`--keep 0` round trip that produces one; a process behind a symlinked parent
going unsignalled; a foreign `~/.*-server` not being adopted, and a legacy tree
with `data/User` still being adopted; `-r` reaching the unit as `--root`;
`TMPDIR=/usr`; unset and missing `HOME`; `--install` over a symlink; and the cron
hint parsed back into the arguments it was built from.

## [2.3] — 2026-08-01

Initial public-facing revision.
