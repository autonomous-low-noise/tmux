# tmux — the dotfiles integration fork

This is `autonomous-low-noise/tmux`: the integration fork of [tmux/tmux](https://github.com/tmux/tmux) that `dfenerski/dotfiles` builds its mux binary from (`nix/mux.nix`, `make install-mux`). It exists for one reason — dotfiles needs upstream *master* plus a small number of patches upstream has blessed but not yet merged — and it carries nothing else. Upstream owns the project; this fork owns an ordering.

## Branch law

- **`master` is the untouched upstream mirror.** It is only ever fast-forwarded to `tmux/tmux` master. Nothing is committed to it, nothing is rebased onto it, no tag lives on it.
- **`dotfiles` = upstream master + a LINEAR queue of blessed commits.** Every commit above the upstream base is one queue entry (see § Queue). The branch is always **rebased** onto upstream master, never merged — a merge commit would fold upstream history into the queue and make an entry's retirement a surgery instead of a `drop`.
- **The queue rides upstream master, never a release branch.** Since 3.7c upstream cuts its release tags on `release_X.Y` branches forked off master (3.7c sits on `release_3.7d`, 3.8-rc2 heads `release_3.8`), so a release tag is not an ancestor of master and is never a base. A release is an event the screen reports, not a base to move to.
- **Force-push of `dotfiles` is expected** and is the normal way the branch moves. Its history is not a record; the tags are (§ Tag law).
- **Staging**: a staged rebase may sit on an untagged `stage/<base8>` branch (`<base8>` = the first 8 hex of its upstream base) until its pin is cut. Nothing pins it, and it is deleted when the pin is cut. Create it as its own worktree — `git worktree add <checkout>/stage -b stage/<base8> dotfiles` — because `dotfiles` stays checked out in the `dotfiles` worktree; in steps 3–7 read `<queue branch>` as `stage/<base8>` there, `dotfiles` otherwise.
- The default branch is `dotfiles`, so a checkout, a `gh` view and a nix `fetchFromGitHub` with a bare ref all land on the consumed state, never on the mirror.

## Queue admission law

A commit enters the queue only if **both** hold:

1. it is **maintainer-blessed upstream** (an open PR the maintainer has said will be merged — nicm's "this is OK" — or a commit already on a maintainer branch) **or authored here** for dotfiles' own need; and
2. it is **actually used by dotfiles** — a `tmux.conf` option, a script contract or a build verdict that reads it. Nothing rides along for completeness.

One entry = **one commit** (a multi-commit PR is squashed) whose message cites its upstream PR and head sha, the upstream base it was applied onto, the date, and its **retirement trigger** — the upstream event that removes it from the queue. A queue entry is authored as its upstream author and committed by this account, so `git log --format='%an | %cn'` reads the provenance without opening the message.

## Tag law

- **Every pinned state is an immutable tag `pin/YYYY-MM-DD` on the `dotfiles` branch.** `nix/mux.nix` pins a *sha*, and a force-push orphans every sha the branch no longer contains — the tag is what keeps that sha reachable, fetchable and buildable on every machine for as long as any dotfiles checkout names it.
- Tags are **never deleted or moved.** Two pins on one day is a second tag with a suffix (`pin/2026-09-06b`), never a reassignment.
- Enforced by repository rulesets: `refs/tags/pin/*` refuses deletion, update and non-fast-forward; `refs/heads/master` and `refs/heads/dotfiles` refuse deletion (force-push on `dotfiles` stays allowed — § Branch law).

## Queue

Base: upstream master `94796f6b1182507efac8a272fc309a79e22e58a5` (2026-09-26). Entries in branch order, oldest first.

| # | entry | upstream ref | admitted | retirement trigger |
| --- | --- | --- | --- | --- |
| 1 | `Add pane-border-type (joined\|separate\|separate-active)` — window option `pane-border-type`; `separate-active` insets every tiled pane and frames only the active one, which is what `tmux.conf` consumes to escape the two-pane shared-divider degeneracy (`redraw_mark_two_pane_colours` paints one divider half in each pane's style) | [tmux/tmux#5433](https://github.com/tmux/tmux/pull/5433) at `1b370a63e016` (`redesigndavid:feat/per-window-border`, 28 commits, squashed) | 2026-09-06; re-squashed 2026-09-26 | #5433 merged upstream — drop the entry at the next rebase and re-pin (a release tag is not the trigger) |
| 2 | `Add AGENTS.md: integration-branch laws and queue ledger` — this file. Tracked although upstream's `.gitignore` lists `AGENTS.md`: added with `git add -f` (edits stage normally once tracked), and upstream's `.gitignore` is never edited for it | authored here | 2026-09-06 as `CLAUDE.md`; renamed to `AGENTS.md` 2026-09-28 | permanent — it is the fork's own contract, retired only with the fork |

Every rebase that drops or adds an entry edits this table in the same push; the table and the branch are one state.

## Rebase procedure

Run from the `dotfiles` worktree (`/Users/dimitar/Projects/tmux/dotfiles` of the bare checkout `/Users/dimitar/Projects/tmux`; remote `upstream` = `tmux/tmux`, `origin` = this fork), or from a staging worktree of the same checkout (§ Branch law). Every git push acts as `autonomous-low-noise`, never as the personal account.

1. **Pre-flight** — the identity: `git var GIT_COMMITTER_IDENT` prints `autonomous-low-noise <272819295+autonomous-low-noise@users.noreply.github.com>` (the bare checkout's `user.name`/`user.email`; a machine's global identity is not the fork's, and a credential router changes push auth only). rerere: `rerere.enabled` and `rerere.autoUpdate` are `true` in the bare checkout, so a conflict resolved once is replayed at the next rebase; a replayed resolution is still reviewed before `rebase --continue`.
2. **Fetch upstream** — `git fetch upstream master`, and fast-forward the mirror: `git push origin upstream/master:master` (a plain push; the mirror must be a fast-forward or upstream rewrote history, which is a stop-and-look event). `<old base>` is the Base line above, `<new base>` is `upstream/master`.
3. **Rebase the queue** — `git rebase --onto <new base> <old base> <queue branch>`. A conflict in a queue entry is resolved *in that entry* (the resolution stays one commit); a conflict too large to resolve honestly means the entry is retired early and dotfiles loses the feature until upstream lands it — say so in the queue table. When an entry's retirement trigger has fired, `git rebase -i` and `drop` it; if upstream merged a *different* shape of the change, the dotfiles consumer (`tmux.conf`, a script) moves with it in the dotfiles PR that re-pins. An upstream PR whose head moved is **re-squashed** at its new head rather than rebased: `git merge --squash <PR head>` on the new base, committed `--author` as the upstream author, its message naming the squash it replaces.
4. **Review** — `git range-diff <old base>..pin/<old date> <new base>..<queue branch>`: every entry reads unchanged, modified (read the inner diff), added or dropped, and the verdict matches the queue table.
5. **Build and run the regress tests** exactly as dotfiles builds: in a `nix-shell` on the pinned nixpkgs snapshot's `tmux` derivation (`nix/mux.nix`), `sh autogen.sh && ./configure --sysconfdir=/etc --localstatedir=/var --enable-jemalloc --enable-sixel --enable-utf8proc && make` — the derivation's own `configureFlags` (`--enable-jemalloc` is its darwin-only verdict; on Linux drop it). Zero warnings, and `./tmux -V` prints `tmux ` + the base's `AC_INIT` version. Then run, from `regress/`, every queue entry's test and the tests of what it touches — today `pane-border-type.sh` and `display-panes.sh` — the way `regress/Makefile` runs each one (`env -i LC_CTYPE=C.UTF-8 MallocNanoZone=0 sh -x ./<test>`; `TEST_TMUX` defaults to `../tmux`, the binary just built, and each script runs on its own `-L test…` or `-S` socket, so no running server is touched). A red test blocks the pin.
6. **Tag** — `git tag -a pin/$(date +%F) -m '<new base sha>; queue: <entries>' <new head>`.
7. **Push the tag, then the branch** — `git push origin pin/<date>` first, so the new head is reachable from an immutable ref before the branch leaves the old one. Then `git push --force-with-lease=dotfiles:<old queue sha> origin <new head>:refs/heads/dotfiles`. The lease names its expected value explicitly: this bare checkout keeps no remote-tracking refs for `origin`, so a bare `--force-with-lease` is refused "stale info". After a staged rebase, once the pin is cut: move the `dotfiles` worktree onto the new head (`git -C <checkout>/dotfiles reset --hard <new head>`), then delete the stage — `git worktree remove <checkout>/stage`, `git branch -D stage/<base8>` (`-D`: the bare checkout's HEAD is its stale local `master`, which never contains the stage; the tag and `dotfiles` hold the sha), `git push origin --delete stage/<base8>`.
8. **Re-pin dotfiles** — in `dfenerski/dotfiles` `nix/mux.nix`, one commit moving together: `rev` = the tagged sha; `hash` = the new `fetchFromGitHub` hash (`nix-prefetch-url --unpack https://github.com/autonomous-low-noise/tmux/archive/<sha>.tar.gz`, then `nix hash convert --hash-algo sha256 --to sri <hash>`, or the build's mismatch message); the three consumer markers above the block, `# fork-rev: <sha>`, `# upstream-base: <new base sha>` and `# pin-tag: pin/<date>`; the `version` attribute, whose prefix is the base's `AC_INIT` version (`next-X.Y-dotfiles-<date>-g<sha7>`); and `meta.changelog`'s rev. Then `make install-mux` and cut over per dotfiles' `.claude/rules/mux.md` — a tmux client and its server must be the same binary, so the server is drained and restarted, never hot-swapped.

## How dotfiles screens this fork

Two read-only screens in dotfiles, both git + curl only, both reading the consumer's three markers (`.claude/rules/forks.md` there owns the grades). Neither edits the module, the profile or a running mux, and nothing runs them on a clock: a re-pin is the procedure above, run deliberately at a screen, never on a release's schedule.

- **`make fork-upstream`** (`scripts/fork-upstream`) screens every fork in dotfiles' `forks.tsv`, this one included: the consumer's markers cross-checked against `rev`, the pinned state (the `dotfiles` head equals `fork-rev`, the `pin/*` tag peels to it; an orphaned pin is loud), the queue (`upstream-base..fork-rev`), upstream drift past the base with a hint when a queue subject reappears upstream, a rebase dry-run (`git merge-tree` of the queue onto upstream's head), and the range-diff between the two newest `pin/*` tags.
- **`make mux-upstream`** (`scripts/mux-upstream`) screens the tmux row only, through the same grades, interleaved with its tmux-only questions: whether upstream tagged a release (information — the queue does not rebase onto it), whether the unreleased CHANGES section grew since the last screen, whether master's man page still documents the surface contract dotfiles' config rides, whether it now documents `pane-border-type` (the #5433 merge sensor — entry 1's retirement trigger fired), and zellij's session contract.
