# ide-server-prune

Keep remote VS Code / Cursor / Windsurf server trees from growing forever.

One shell script, no dependencies, no network access. It removes the server
versions your editor no longer wants, the processes those versions left behind,
and the caches, logs, superseded extensions and temp files that pile up around
them — while you stay connected.

## The problem

Every time your editor updates, Remote-SSH unpacks a fresh server on the host.
The CLI keeps the five most recent ones (`KEEP_LRU = 5` in
[`cli/src/download_cache.rs`](https://github.com/microsoft/vscode/blob/main/cli/src/download_cache.rs))
and evicts the sixth. At roughly 550 MB per version that is a standing 2.7 GB,
which is a lot on a small VPS and unpleasant on a shared box.

Worse, eviction deletes the directory without stopping the server running out of
it, so a host accumulates node processes whose install no longer exists. There
is still no built-in cleanup: [microsoft/vscode#246885](https://github.com/microsoft/vscode/issues/246885)
is open, and the only shipped remedy is "Uninstall VS Code Server from Host",
which removes everything and re-downloads on the next connect.

## What it does

- **Server installs** — keeps the head of the CLI's own LRU list (2 entries by
  default) and removes the rest, in both the current `cli/servers/Stable-<commit>`
  layout and the legacy `bin/<commit>` one, together with the matching
  `code-<commit>` binaries and `.cli.<commit>.log` / `.<commit>.pid` leftovers.
- **Processes** — a version that is being removed gets `SIGTERM`, then `SIGKILL`
  after a grace period; a download in flight for the version that replaces it is
  spared, even though `Stable-<commit>` is a prefix of `Stable-<commit>.staging`.
  Processes whose install an earlier run already deleted are swept separately, in
  both layouts.
- **`lru.json`** — entries whose directory is gone are dropped, order preserved.
- **Caches and logs** — `CachedExtensionVSIXs`, `CachedProfilesData`, `clp`,
  per-commit `CachedData`, session logs and CLI logs past a retention window.
- **Extensions** — interrupted `.vsctmp` installs, versions the editor itself
  marked in `.obsolete`, and versions superseded by a newer one. Deciding which
  version is newer needs a version-aware sort; where the host has none, the
  superseded pass says so and stops rather than trusting an ordering that reads
  1.10.0 as older than 1.9.0.
- **Workspace storage** — storage directories for workspaces whose folder no
  longer exists, and empty ones.
- **Temp files** — `vscode-*`, `cursor-*`, `windsurf-*`, `vscodium-*` and
  `node-compile-cache*` in `$TMPDIR` and `/tmp`. The list is deliberately narrow;
  widen it with `IDE_TMP_PATTERNS` if your setup leaves other litter behind.

## Safety

Deleting things on a live host deserves a short list of what this will *not* do.

- **`--dry-run` prints every removal and changes nothing.** Start there. It
  refuses to be combined with `--install` or `--uninstall` rather than quietly
  ignoring one of them, since those write a binary, a unit and a registration
  with the service manager. Nothing under any root or temp directory is touched;
  the one thing a dry run does write is the tool's own lock directory, which it
  takes so that its report describes a tree no other run is changing underneath
  it.
- **It never prunes the version hosting your own session.** The process tree
  above the script is inspected, and any server referenced by an ancestor is
  protected — so running it from a terminal inside the remote editor is safe even
  if that version has fallen out of the LRU head.
- **It never signals its own process group or the invoking user's ancestors**,
  and it only ever matches processes belonging to the invoking user.
- **Unattended runs leave every running server alone.** A timer is not a
  descendant of the server it prunes, so the rule above cannot see a colleague's
  live session. With no terminal attached the script therefore behaves as if
  `IDE_PROTECT_LIVE=1`; a human at a prompt gets the full sweep. Override either
  way with `--protect-live` / `--no-protect-live`. That protection covers an
  unfinished `.staging` download as well, which needs asking about by path: no
  commit is ever recorded for a version nobody could start yet, so there is
  nothing for the usual "keep what is running" rule to hold it by.
- **A tunable that is not a number aborts the run.** `IDE_FRESH_GUARD_MINUTES=abc`
  would otherwise make `find -mmin -abc` match nothing and quietly disable the
  freshness guard, so every setting is validated up front, by name. A value
  carrying a newline is refused for the same reason: `--install` writes settings
  into a unit file one per line, and the half after the break would silently
  stop belonging to the setting it was typed into.
- **Installs younger than an hour are left alone**, and an unfinished `.staging`
  download for half a day — the transfer writes *inside* that directory, so the
  age of the newest file anywhere under it is what counts, not the directory's
  own timestamp. Half a gigabyte over a thin link takes longer than the hour that
  guards a finished install.
- **The last fallback always keeps something.** With no `lru.json` and nothing
  running, the most recently installed tree is kept. Should the host answer no
  timestamps at all, that rule keeps the first install it finds rather than none
  — a guard that fails open would mark every version in the root stale. An
  `lru.json` entry whose directory is already gone does not occupy one of the
  kept slots either: it would leave the set non-empty, so the fallback would
  never fire, and every install actually on disk would then look stale.
- **It refuses to run through `sudo`.** Being root is fine where the server tree
  belongs to root; arriving as root from another account is not, because the
  tree, the processes and the timer all belong to whoever called it.
- **Every removal must resolve to a path under a root the script resolved
  itself.** Paths containing `.` or `..` components are refused outright, and the
  check is repeated after resolving symlinked parents, so neither a planted
  symlink nor a name read out of a file on disk can redirect a deletion outside
  the tree. `/` and `$HOME` are rejected as roots — by their resolved path, so a
  symlinked home is rejected too — and so is any directory that does not carry
  the structure of a server tree, including one named through `-r` or
  `IDE_SERVER_ROOTS`. Nothing is signalled for a target the check refuses either:
  a stale install is terminated on the way to being deleted, so the gate is asked
  before the signal and not only before the `rm`. `TMPDIR` becomes a removal root
  for the rest of the run and is screened before the temp pass — it has no
  structure to be recognised by, so it is screened by name instead: `/` and your
  home are rejected, so is any directory *containing* your home, which
  `TMPDIR=/home` or `TMPDIR=/Users` otherwise would be, and so is a system
  directory named outright (`/usr`, `/etc`, `/var`, `/Library`, `/System` and
  their kind), which a length test alone lets straight through. Anything
  resolving *below* those is still fine — macOS hands out
  `/private/var/folders/…` and that is the normal answer.
- **`extensions.json` is treated as authoritative.** A version directory it
  references is never removed, so the manifest cannot end up pointing at
  something that is gone. This also makes profiles, which legitimately keep
  several versions of one extension, safe. Without a readable manifest nothing is
  removed for being superseded at all — "looks like an older version number" is a
  guess, and guessing wrong uninstalls an extension that was in use.
- **Workspace storage goes only when the folder is demonstrably gone.** The path
  is read out of `workspace.json`, and a `%` that does not introduce two hex
  digits means the file is not the format we know — the entry is kept rather than
  decoded by hand into a path that could not exist anyway. If the folder's
  *parent* is missing too, that reads as an unmounted volume or a home that
  moved, not as a deleted project, and the storage — the only place that
  project's local history lives — stays.
- **`lru.json` is only rewritten when it matches the format we understand**, and
  the write is atomic. Anything else is left untouched with a warning.
- **Temp files are only removed when they are yours, older than a day, not a
  socket, pipe or symlink, and not open by any process.** The same question is
  asked of a log directory before it is dropped for age, since a session can
  simply have been up longer than the retention window. If neither `/proc` nor
  `lsof` is available to determine that, the temp pass is skipped rather than
  guessed at, and logs are kept. Sockets are excluded on purpose: `/proc` reports
  an open unix socket by inode and never by path, so a live `vscode-ipc-*.sock`
  ([#7926](https://github.com/microsoft/vscode-remote-release/issues/7926)) cannot
  be told from an abandoned one, and removing a live one drops your connection.
- **No network access, no telemetry, no auto-update.** The script makes no
  outbound connection of any kind.

Killing a stale server disconnects any editor window still attached to it; the
editor reconnects and keeps unsaved buffers. Still, the first run on a host is
best done from a plain SSH session rather than from inside the editor.

## Install

The tool is a single self-contained script with nothing to build. Clone the
repository, then pick one of the two routes below — they are alternatives, not
steps of one sequence.

Either way, start by looking before you leap:

```sh
./ide-server-prune --dry-run
```

### Route 1 — install and schedule in one step (recommended)

```sh
./ide-server-prune --install
```

`--install` does the copying for you: it places the script at
`~/.local/bin/ide-server-prune` and enables the daily job described under
[Scheduling](#scheduling). There is no separate copy step to run first, and
running it a second time is harmless: the unit is rewritten rather than
duplicated, and no prune is triggered. Upgrading later is the same command —
[Updating](#updating) covers what to repeat along with it.

### Route 2 — just the command, no job

If you would rather run it by hand or wire it into your own scheduler:

```sh
mkdir -p ~/.local/bin
cp ide-server-prune ~/.local/bin/
chmod 755 ~/.local/bin/ide-server-prune
```

You can still add the job later: `ide-server-prune --install` run from the copy
on your `PATH` sees that it is already in place and only sets up the timer.

Make sure `~/.local/bin` is on your `PATH`; most distributions add it when it
exists, otherwise add `export PATH="$HOME/.local/bin:$PATH"` to your shell rc.

## Usage

```
ide-server-prune [options]

  -n, --dry-run       report what would be removed, change nothing
  -q, --quiet         only report the summary line
  -k, --keep N        keep N server versions from the LRU head (default 2;
                      0 removes every version that is not hosting a session)
  -r, --root PATH     operate on PATH instead of auto-detected roots (repeatable)
  -p, --protect-live  never touch a server that is currently running
      --no-protect-live   prune stale servers even while they are running
      --install       install a daily timer (systemd --user on Linux, launchd on macOS)
      --uninstall     remove that timer
  -V, --version       print version
  -h, --help          this text
```

Roots are detected by structure, not by name: any `~/.*-server` or
`~/.*-server-*` directory holding a `cli/servers` qualifies, which covers VS
Code, Insiders, Cursor, Windsurf, VSCodium and anything else built on the same
CLI. The legacy layout has no such directory to be recognised by, so `bin` plus
`data` qualifies as well — but only alongside a mark the editor itself leaves: an
install named after a commit, or the `data/User` / `data/Machine` state directory
it keeps beside one. `bin` and `data` alone are far too ordinary a shape to adopt
a directory on, and a stray `~/.something-server` carrying them is left alone.
`~/.codeium/windsurf-server` is checked too. Use `-r` for a custom
`remote.SSH.serverInstallPath`, or set `IDE_SERVER_ROOTS` to a colon-separated
list. A root you name that way has to pass the same structural test — a path
that is not a server tree aborts the run instead of being pruned, so a typo in
`-r` or a stale `IDE_SERVER_ROOTS` cannot aim the tool at an unrelated
directory.

The default of two kept versions exists so a second client one release behind
still finds its server. Set `--keep 1` if only one machine connects. `--keep 0`
is meaningful too — it empties the keep set outright, so nothing survives but the
version hosting your own session, and the "keep the newest install" fallback does
not fire either. That is a clean slate before a reinstall, not a daily setting.

## Scheduling

`--install` (Route 1 above) is the whole setup. If you took Route 2, this is the
command that adds the job:

```sh
ide-server-prune --install
```

It enables a daily 04:30 job: a
`systemd --user` timer on Linux (with `Persistent=true`, so a host that was off
catches up) or a launchd agent on macOS logging to
`~/Library/Logs/ide-server-prune.log`. Running it twice adds nothing twice —
the unit is rewritten, not duplicated, and the timer is enabled rather than
started, so a re-install never triggers a prune. `--uninstall` removes the job.
Where neither init system is available the script prints a ready-made cron line
instead.

The job starts from an empty environment, so `--install` writes whatever settings
the invocation carries into the unit it generates and prints them back to you.
Tune the run by hand first, then install exactly that:

```sh
IDE_KEEP_SERVERS=1 ide-server-prune --dry-run
IDE_KEEP_SERVERS=1 ide-server-prune --install
```

The unit therefore describes *the call that wrote it*, and nothing else: a
second `ide-server-prune --install` with a barer command line schedules a barer
run, and the `IDE_KEEP_SERVERS=1` above is gone from it. That is deliberate —
a unit that remembered its settings could never be talked out of one — so what
the tool does instead is say so:

```
  carrying over: nothing - the scheduled run uses the built-in defaults and auto-detected roots
ide-server-prune: IDE_KEEP_SERVERS=1 was in the installed timer; this run does not set it, so
  the scheduled run falls back to the built-in default
ide-server-prune: the timer now matches this invocation - re-run --install with the earlier
  settings to put them back
```

Every setting the new unit drops or changes is named, on stderr, read back from
the unit that was there a moment ago; repeating the *same* `--install` says
nothing, because nothing moved. To see what is scheduled right now without
touching it: `systemctl --user cat ide-server-prune.service`, or
`cat ~/Library/LaunchAgents/local.ide-server-prune.plist` on macOS.

`--install` also copies the script it is being run from over
`~/.local/bin/ide-server-prune`. Run it out of an older checkout and the
installed copy goes back a version — it says which version it is replacing when
the two differ.

The generated cron line, on hosts without either init system, is prefixed the
same way, and every value in it is quoted so the line survives being pasted into
a crontab.

`-r` travels into the unit as a `--root` argument rather than as
`IDE_SERVER_ROOTS`. The two are not interchangeable: `-r` names the roots
outright, while `IDE_SERVER_ROOTS` only *adds* to the auto-detected set, so a
timer configured the second way would sweep trees the run you tried by hand never
looked at.

Note for macOS: `--install` registers the agent with `launchctl bootstrap
gui/<uid>`, which needs a GUI session for that user — over SSH to a headless Mac
that usually means somebody has to be logged in at the console. Where it fails
the script tries the older `launchctl load -w`, and if that fails too it stops
and prints a cron line to use instead. Nothing is left half-registered:
registering a new agent means unloading the one already there, so the plist that
was working is kept aside first and put back — and loaded again — if neither
call succeeds. A re-install that fails leaves you the timer you already had.

Note for Linux: the installer calls `loginctl enable-linger` for your user, which
is what lets the timer fire while you are not logged in. It also means your other
user services keep running after logout.

That call goes through polkit, and polkit says yes only to a session it
considers active — a minimal VPS image often ships no `polkitd` at all, and then
it is refused outright with nobody to authenticate to. The installer checks the
resulting *state* rather than the status of that call, so it stays quiet where
linger is already on and says this where it is not:

```
ide-server-prune: linger is not enabled for you: the timer only fires while you are logged in
ide-server-prune: enable it with: sudo loginctl enable-linger you
```

Run that one command and the timer is unattended for good; the script does not
take the privilege itself. Check the result with:

```sh
systemctl --user list-timers ide-server-prune.timer
journalctl --user -u ide-server-prune -n 20
```

## Updating

Updating is the install again, run out of a fresh copy of the repository:

```sh
git clone https://github.com/banochkin/ide-server-prune.git ~/ide-server-prune
~/ide-server-prune/ide-server-prune --install
```

Where that clone already exists, `git -C ~/ide-server-prune pull` first and run
the same second line. `--install` copies the script it is being run from over
`~/.local/bin/ide-server-prune`, rewrites the unit and re-enables the job.
Nothing has to be uninstalled first, no prune is triggered, and repeating it
changes nothing — so it is safe on a host somebody is working on. Confirm with
`~/.local/bin/ide-server-prune --version`.

It reports two things on stderr, and both are worth reading:

```
ide-server-prune: replacing /home/you/.local/bin/ide-server-prune (version 2.6) with version 2.7 - the copy being run
ide-server-prune: IDE_KEEP_SERVERS=1 was in the installed timer; this run does not set it, so the scheduled run falls back to the built-in default
```

The first line is the update itself. The second means this host had settings:
the unit describes the call that wrote it, so they have to be repeated —
`IDE_KEEP_SERVERS=1 ~/ide-server-prune/ide-server-prune --install` puts the
timer back to what it was. Silence there means there was nothing to carry over.

Several hosts at once:

```sh
for host in web-1 web-2 build-1; do
    ssh "$host" 'git -C ~/ide-server-prune pull ||
                 git clone https://github.com/banochkin/ide-server-prune.git ~/ide-server-prune
                 ~/ide-server-prune/ide-server-prune --install'
done
```

macOS hosts are the exception: registering a launchd agent needs a GUI session
for that user, which over SSH to a headless Mac usually means somebody has to be
logged in at the console. Where it fails the update stops and puts the agent that
was running back, so the host keeps the timer it already had.

## Tuning

All defaults can be overridden through the environment.

| Variable | Default | Meaning |
| --- | --- | --- |
| `IDE_KEEP_SERVERS` | `2` | server versions kept from the LRU head (`0` keeps none) |
| `IDE_LOG_RETENTION_DAYS` | `7` | logs older than this are dropped |
| `IDE_TMP_MIN_AGE_DAYS` | `1` | minimum age for temp artefacts |
| `IDE_WORKSPACE_MIN_AGE_DAYS` | `30` | minimum age for orphaned workspace storage |
| `IDE_FRESH_GUARD_MINUTES` | `60` | never touch installs younger than this |
| `IDE_STAGING_MIN_AGE_HOURS` | `12` | keep an unfinished `.staging` download this long |
| `IDE_KILL_GRACE_SECONDS` | `10` | seconds between `TERM` and `KILL` |
| `IDE_PROTECT_LIVE` | `auto` | `1` = never kill a live server, `0` = prune stale ones anyway, `auto` = `1` when no terminal is attached |
| `IDE_CLEAN_TMP` | `1` | clean own temp artefacts |
| `IDE_CLEAN_EXTENSIONS` | `1` | drop superseded extension versions |
| `IDE_CLEAN_WORKSPACE_STORAGE` | `1` | storage of workspaces whose folder is gone |
| `IDE_TMP_PATTERNS` | `vscode-* cursor-* windsurf-* vscodium-* node-compile-cache*` | temp names to sweep, space separated |
| `IDE_SERVER_ROOTS` | — | extra roots, colon-separated |

Counters take whole numbers and switches take `0` or `1`. Anything else aborts
the run with the offending variable named, rather than being silently ignored —
a mistyped guard that fails open is worse than no guard. An empty value means
"use the default".

`IDE_PROTECT_LIVE` defaults to `auto` because the right answer differs by
context. Interactively you want stale servers gone even if something is still
running in them — a hung server would otherwise protect its own outdated version
forever, which is the situation this tool exists to fix. Unattended, on a host
reached from several machines, you want the opposite: nobody is watching, and a
disconnect nobody asked for is worse than a version kept one day longer. Pin it
to `1` or `0` if your host has one clear answer.

Every pattern in `IDE_TMP_PATTERNS` must start with a literal alphanumeric
character and contain no `/`, so the list cannot be widened into something that
sweeps a shared `/tmp` wholesale. Bare `code-*` and `mcp-*` are not in the
default set for the same reason: they match plenty of things an IDE never wrote.

## Requirements

`bash` 3.2 or newer — the stock `/bin/bash` on macOS qualifies — plus the usual
POSIX tools (`find`, `ps`, `du`, `awk`, `sed`, `grep`, `sort`). No `jq`, no
`flock`, no `lsof`, no Python, no package manager. Linux and macOS.

One thing outside POSIX is used where it is available and skipped where it is
not: `sort -V`. Comparing extension versions has no correct answer without it —
busybox even accepts the flag and ignores it — so the superseded-extension pass
checks that version ordering really works and stands down with a warning if it
does not. Everything else runs unchanged.

Run it as the user who owns the server tree. On a host where that user is root
this is fine; reaching root from another account is not, and `sudo` is refused
outright rather than left to do the right thing by accident.

## Tests

```sh
./selftest
```

Builds a throwaway server tree in a temporary directory and asserts what
survives, what does not, and that the guards hold — dry-run inertness and its
refusal to install, refusal to delete through a symlinked parent or a `..` in a
file on disk, lock behaviour, format-preserving `lru.json` handling and the
keep slot an entry with no install must not occupy, rejection of malformed and
multi-line tunables, the freshness guard, a `.staging` download still being
written, workspace storage behind a missing parent or an undecodable path,
refusal under `sudo` and of a `TMPDIR` that is unsuitable or contains your home,
extension handling without a manifest, and what `--install` writes into the
generated unit. A directory something still holds a descriptor open in — in
`/tmp` and under `data/logs` alike — is checked to survive alongside an
identical one nobody holds, which is what distinguishes a working in-use test
from one that always answers "free". Five checks put real processes in their own
process group: that a download in flight outlives the removal of the install it
replaces, that a stalled one survives `--protect-live` while `--no-protect-live`
still sweeps it, that an orphan from the legacy `bin/<commit>` layout is seen at
all, that it is then actually signalled, and that a process naming a *live*
install alongside a removed one is left alone. They report `skip` where the shell
has no job control; a further check covers that a process behind a symlinked
parent is not signalled for a removal that is then refused. The
fallback that picks the newest
install is checked against a stubbed `stat` that answers the way GNU coreutils
does, and against one that answers nothing at all; a stubbed `sort` without `-V`
checks that extension versions are then left alone. It touches nothing outside
its own temporary directory: the scheduling check runs against an isolated
`HOME` with a stub `launchctl`/`systemctl` on `PATH`, so no real service manager
is involved — which also covers what `-r` writes into the unit, that `--install`
does not copy itself through a symlink left on its name, that a second
`--install` names every setting it drops and stays quiet when it drops none,
that replacing the installed binary with a differently versioned copy is
announced, that a launchd registration which fails puts the previous agent back,
and that the cron line printed where neither init system exists parses back into
the arguments it was built from. The suite runs green on both platforms it
targets: macOS 26.6 on arm64 and Linux 6.8 on x86_64, with no skipped checks on
either.

## Changelog

See [CHANGELOG.md](CHANGELOG.md). Engineering notes — invariants, the traps this
script is shaped around, and what has actually been exercised on which platform —
are in [NOTES.md](NOTES.md).

## Licence

MIT, see [LICENSE](LICENSE). Maintained by banochkin.com DAO <mail@banochkin.com>.
