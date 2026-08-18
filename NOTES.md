# Engineering notes

Working notes for anyone changing this script — including me, next time. The
README says what the tool promises; this file says what the code is shaped around
and what has actually been exercised. Add to it rather than rediscovering it.

## Invariants

Break one of these and the tool stops being safe to run on a live host.

1. **Nothing irreversible happens outside a resolved root.** `may_remove()` is the
   single gate, and it is asked before *every* irreversible act — the removal and
   the signal that precedes it. Adding a third kind of act means adding a third
   call, not trusting the caller.
2. **`is_safe_target()` checks three ways.** Literal path under a resolved prefix;
   the same path with every symlinked parent resolved; and a flat refusal of any
   `.` / `..` / `//` component. The second check is what a planted symlink inside
   the tree runs into; the third is what a name read out of a file on disk runs
   into.
3. **`safe_prefixes` only ever gains a path that passed a structural or a name
   screen.** Server roots pass `looks_like_server_root()`; temp directories pass
   the screen in `prune_tmp()`. Anything else added there widens every removal in
   the run.
4. **A guard that cannot answer must fail closed.** `path_in_use()` returns "in
   use" when it has neither `/proc` nor `lsof`; `newest_install()` is the one
   deliberate exception — it fails *open* on purpose, because "keep nothing" would
   mark every install in the root stale. `is_safe_target()` resolves the parent
   itself rather than through `abs_dir()` for the same reason: `abs_dir()` hands
   back its own argument when resolution fails, which is a *report* helper's
   answer and would let an unresolvable parent pass the check unresolved.
5. **A guard has to be tested against both answers.** The in-use check passed
   its "this is removed" half for five versions while the half nobody had
   written — "this is kept" — was false on every macOS host. Any check of the
   form "we only do X when Y" needs a case where Y holds and X does not happen,
   not just the reverse.
6. **A tunable is validated by name, up front.** `[ abc -eq 1 ]` is merely false
   and `find -mmin -abc` merely matches nothing, so a typo silently disables a
   guard instead of failing.
7. **`--dry-run` writes nothing into any tree it operates on**, including the
   `lru.json` rewrite and the timer. It does take the lock, which is the one
   thing it creates — deliberately, so its report describes a tree no concurrent
   run is changing underneath it.
8. **Replacing something that already works has to be reversible.** `--install`
   on a host that already has a timer takes the working registration apart
   before it puts the new one together, and the step that puts it together is
   the one that fails on a headless Mac. Anything added on that path — a second
   service manager, another platform — keeps a copy of what it displaced and
   puts it back before it calls `die()`.

## Shell traps this code is shaped around

The script runs under `set -euo pipefail` with `nullglob`. That combination has
sharp edges, and most of the bugs found in the 2.3 review were one of these:

- **`pipefail` + `set -e` + `grep` = a silent exit.** `x=$(grep … | wc -l)` ends
  the run when grep matches nothing, because the pipeline's status becomes the
  assignment's. Any command substitution containing a pipeline needs `|| true` on
  the pipeline, or `{ … || true; } | …` when the failure is in the middle. When
  auditing, grep for `=\$\(` on a line containing `|` and check every hit.
  Note this bites *assignments*; a command substitution used as an argument
  (`say "$(…)"`) takes its status from the surrounding command instead.
- **A redirect is the same trap without a pipeline.** `x=$(tr … <"$file")` ends
  the run just as dead when the file cannot be read, and the `|` audit below
  does not see it because there is no `|` on the line. `prune_lru()` reached 2.5
  that way: it asked whether `lru.json` was writable and never whether it was
  readable, so a file in mode 200 aborted the sweep before its summary. Guard
  the substitution *and* ask `-r` — the audit greps for `<"` for this reason.
- **`set -e` ignores a failing command that is not the last in an `&&`/`||`
  list.** `[ -d "$p" ] && do_thing` is safe as a statement; that is why the file
  uses that idiom freely. Do not "tidy" one of those into an `if` that returns 1
  as a function's last statement — that *does* trigger the exit.
- **A function whose last statement is a loop returns the loop's last status.**
  Several helpers end in `return 0` for exactly this reason. Keep them.
- **`nullglob` makes an unquoted `$VAR` in a `for` glob against the current
  directory** and silently drop every word that matches nothing. Hence
  `split_tmp_patterns()` toggling `set -f`.
- **`die` inside `$( )` exits the subshell, not the script.** It still works —
  `roots=$(resolve_roots)` is a plain assignment, so `set -e` takes the non-zero
  status — but the mechanism is indirect. Do not wrap that assignment in `local`
  or an `if`, either of which would swallow it.
- **`awk -v` expands escape sequences in the assignment**, which corrupts any
  path containing a backslash. Needles travel through `ENVIRON` instead.
- **GNU `stat` reads `-f` as `--file-system`** and answers `%m` with `?` at exit
  0, so the BSD spelling must be tried *second* and every answer checked for being
  a number.
- **busybox `sort` accepts `-V` and ignores it**, which is worse than refusing it —
  hence `version_sort_works()` testing the answer rather than the flag.
- **`lsof`'s exit status is not an answer.** `lsof +D DIR` exits non-zero after a
  walk it completed without trouble, and uses that same status for "nothing is
  open" — so `lsof … >/dev/null && in_use` reads as "free" for every directory.
  Read the output instead; `-F` gives one machine-readable line per hit and
  nothing at all when there are none. Cost us five versions of a temp sweep that
  believed it was checking something.
- **`du -s` prints one line per argument, except when the name contains a
  newline**, which breaks that line in two. `tail -n1` on that output measures
  the tail of the *name*. Take the first line.

## Platform facts worth not re-measuring

- `/tmp` is a symlink to `/private/tmp` on macOS, and `find` does not descend into
  a symlinked starting point. Temp roots are resolved with `pwd -P` first, and the
  prefix check compares resolved against resolved.
- macOS `TMPDIR` is `/var/folders/…`, resolving to `/private/var/folders/…`. The
  system-directory refusal list is exact-match for that reason — `/var` is
  refused, `/private/var/folders/…` is not.
- **Nothing in `prune_tmp()` refuses `/tmp` by name.** It is not on the
  system-directory list, and four characters clear the length test — the only
  screen that turns it down is the `$HOME` comparison. That is why the check for
  the duplicate refusal hands the script `HOME=/tmp`: it is the one arrangement
  where `TMPDIR` and `/tmp` resolve to the same directory *and* that directory is
  refused, which is the case the ordering inside the loop decides.
- The `lsof` branch costs one process per question, and since 2.5 the question is
  asked of every session log directory as well as every temp candidate. Measured
  on macOS 26.6, arm64: 40 log directories in 2.65 s, so roughly 60 ms per
  `lsof -F n +D`. Fine for a nightly job at `Nice=10`; see "Still open" if it
  ever stops being fine.
- The `/proc` fast path is worth having. Measured on Linux 6.x, GNU findutils
  4.9.0, a host with 262 fd links: one `find /proc -maxdepth 3 … -printf '%l\n'`
  covered 1238 links in 0.12 s, while 500 `readlink` forks took 0.50 s — roughly
  20× per descriptor, and `path_in_use()` is asked once per candidate. The
  portable `readlink` walk stays as the busybox fallback.
- Stock macOS `/bin/bash` is 3.2.57. No associative arrays, so sets are `|a|b|`
  strings. Everything else the script uses (`${!name}`, `${var//…}`, `<<<`,
  process substitution, `${var:0:8}`) works in 3.2.

## Deliberate choices that look like bugs

Do not "fix" these without reading the reasoning first.

- **An interactive run prunes live servers** (`IDE_PROTECT_LIVE=auto` resolves to
  `0` with a terminal attached). A hung server would otherwise protect its own
  outdated version forever, which is the situation the tool exists to fix. The
  unattended case resolves the other way because a timer is not a descendant of
  the server it prunes and cannot see a colleague's session.
- **Sockets in `/tmp` are never removed**, however old. `/proc` lists an open unix
  socket by inode and never by path, so a live `vscode-ipc-*.sock` cannot be told
  from an abandoned one.
- **Workspace storage stays when the folder's *parent* is missing too.** That
  reads as an unmounted volume or a moved home, not a deleted project — and the
  storage is the only place that project's local history lives.
- **`percent_decode()` refuses rather than guesses.** A `%` not introducing two
  hex digits means the file is not the format we know; decoding it by hand turns
  `100%done` into a path that cannot exist, which is exactly the answer that makes
  the caller delete something.
- **The unit describes the call that wrote it, and nothing else.** A second
  `--install` with a barer command line schedules a barer run: the settings the
  first one carried are gone from the regenerated unit. It is tempting to "fix"
  this by merging the settings already installed into the new unit, and that
  trade is worse than it looks — once a unit remembers a setting there is no
  spelling left that means "go back to the default", and `--install` stops being
  a description of a run you just tried by hand. What 2.7 added instead is
  `report_config_drift()`: read the old unit before overwriting it and name
  every setting that this call drops or changes. Announce the revert, do not
  prevent it.
- **Exit status 1 after a refusal is intentional.** `had_error` makes a timer log
  a failed unit, which is the point.
- **`freed_kb` counts in `--dry-run` too.** It is a projection, not a claim about
  what happened.

## Verified

Everything below was exercised against 2.7, not reasoned about.

| | |
| --- | --- |
| `selftest` | 119 checks, all passing, no skips |
| Shells | bash 5.2.21 and stock `/bin/bash` 3.2.57 |
| Host | macOS 26.6, arm64 |
| Linux | full suite green on 6.8, x86_64 (bash 5.2.21, GNU coreutils 9.4, findutils 4.9.0) — **last run against 2.6**; 2.7 has not been on a Linux host yet |

The Linux run is what turned up the two bugs 2.6 fixes, and one bug in the
harness: the stub standing in for a working GNU `stat` delegated to the BSD
spelling, which answers nothing on GNU coreutils, so that check failed on every
Linux host for reasons that had nothing to do with the tool. A stub that can
only be right on one platform is worse than no stub — it reports a failure
where there is none and trains you to ignore the suite.

Checked by hand on Linux beyond the suite, all refused or contained: a symlink
to `$HOME` planted inside `cli/servers` under a commit name (the link goes, its
target does not); a symlink to `$HOME` *inside* an install being removed
(`rm -rf` does not follow it); `--root` at `/`, `/home`, `$HOME`, `/usr`,
`/etc`, `/usr/local`, `/usr/share`, `/var/tmp`; `TMPDIR` at `/`, `/var`,
`$HOME`; `workspace.json` pointing at `/`, `/etc`, `/etc/passwd` and a
percent-encoded `../../etc`; absolute paths and `..` in `.obsolete`; a `ps`
table of 155 processes narrowed to the 26 belonging to the invoking user. The
generated `systemd` service and timer pass `systemd-analyze verify` with a
`Environment=` value carrying both a double quote and a backslash.

Adversarial cases run by hand and covered by `selftest`: `cli/servers`, `data/`
and `extensions/` each replaced by a symlink pointing outside the tree (all
refused, targets intact); `../../../outside` in `.obsolete`; a process behind a
symlinked parent (unsignalled since 2.4); `TMPDIR` set to `$HOME`, to a parent of
`$HOME`, and to `/usr`; unset and non-existent `HOME`; `--install` over a symlink;
a cron hint containing an apostrophe and a path with a space.

Run by hand for 2.7: a re-install under stock `/bin/bash` 3.2.57 reading back a
`systemd` value carrying a double quote and a `plist` value carrying `&` and
`<` — the two escapings `systemd_unquote()` and `xml_unescape()` have to undo,
and the two that a naive two-pass substitution gets wrong. Both branches were
driven from macOS through a stub `uname`, which is now how `selftest` reaches
them as well.

Run by hand for 2.5 and not (yet) in `selftest`: roots whose path carries a
space, an apostrophe, a backslash, `..` inside a component, and each of
`( ) + [ ] *` — the run behaves identically and exits 0, which is what
`quote_re()` is there for. `--root /`, `IDE_SERVER_ROOTS=/`, `--root $HOME` and a
root symlinked to `$HOME` are all refused by resolved path.

The in-use check is worth re-testing by hand whenever it changes, because the
two platforms take entirely different branches — `/proc` on Linux, `lsof` on
macOS — and only one of them is exercised on any given host. The macOS branch is
the one that was wrong for five versions.

Static sweep: the only `cd` in the file is inside the subshell in `abs_dir()`, and
`CDPATH` is unset at the top — there is no way for the working directory of the
main shell to move. Exactly one `rm -rf` acts on pruning targets
(`discard()`); the others take deterministic paths — the lock directory, the
`lru.json` temporary, unit files.

## Still open

- The systemd branch of `--install` is still only exercised through stubs — on
  either platform now, since 2.7 reaches it through a stub `uname`. The unit it
  writes is checked by `systemd-analyze verify` on a real Linux host, so the
  file is known to be valid, but `systemctl --user daemon-reload`,
  `systemctl --user enable --now` and `loginctl enable-linger` have never been
  observed to succeed against a live service manager — only to be called. The
  same now goes for the launchd restore path: the stub `launchctl` reports
  success, so what is verified is that the previous plist is back on disk and
  that a load is attempted, not that launchd accepts it.
- `report_config_drift()` compares against the unit on disk, so a unit edited by
  hand is read as "what was installed" — which is right for the systemd
  `Environment=` lines and the `--root` arguments it knows, and blind to
  anything else somebody added there. A hand-edited unit is silently discarded
  by the next `--install`, exactly as before; only the settings we ourselves
  write are named on the way out.
- `IDE_SERVER_ROOTS` is colon-separated, so a root containing a `:` cannot be
  expressed. `-r` has no such limit; the variable would need a different
  separator, which is a compatibility break.
- The `lsof` branch of `path_in_use()` forks once per question, where the
  `/proc` branch loads every open path once and answers from memory. The
  symmetrical fix is a single `lsof -w -F n -u $UID` cached the same way. It was
  deliberately *not* done in 2.5: the whole release is a repair of that guard,
  and a cache that comes back empty for any reason — a failed invocation, a
  refused read — reads as "nothing is open", which is the answer that deletes
  things. Doing it properly means a fail-closed fallback to the per-path call,
  and that deserves its own change rather than riding along with this one. At
  60 ms a question it is not yet urgent.
- `pids_matching()` matches a needle anywhere in a process's arguments. A
  process of the same user that merely *mentions* an install being removed —
  a `tail -f` on one of its logs, say — is a candidate. Narrowing this means
  parsing argv positionally, which is its own kind of fragile.

## Quick checks when changing anything

```sh
bash -n ide-server-prune && /bin/bash -n ide-server-prune
./selftest
grep -nE '^[[:space:]]*[a-z_]+=\$\(' ide-server-prune | grep -E '\||<"' | grep -v '|| true'
```

The last one is the `set -e` audit: every hit is an assignment whose status ends
the run, so each needs its pipeline or its redirect guarded, or a reason why it
cannot fail. It looks for `<"` as well as `|` because a redirect from an
unreadable file is the same trap with no pipeline in sight.
