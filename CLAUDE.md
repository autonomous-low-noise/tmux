# tmux — the dotfiles integration fork

This is `autonomous-low-noise/tmux`: the integration fork of [tmux/tmux](https://github.com/tmux/tmux) that `dfenerski/dotfiles` builds its mux binary from (`nix/mux.nix`, `make install-mux`). It exists for one reason — dotfiles needs upstream *master* plus a small number of patches upstream has blessed but not yet merged — and it carries nothing else. Upstream owns the project; this fork owns an ordering.

## Branch law

- **`master` is the untouched upstream mirror.** It is only ever fast-forwarded to `tmux/tmux` master. Nothing is committed to it, nothing is rebased onto it, no tag lives on it.
- **`dotfiles` = upstream master + a LINEAR queue of blessed commits.** Every commit above the upstream base is one queue entry (see § Queue). The branch is always **rebased** onto upstream master, never merged — a merge commit would fold upstream history into the queue and make an entry's retirement a surgery instead of a `drop`.
- **Force-push of `dotfiles` is expected** and is the normal way the branch moves. Its history is not a record; the tags are (§ Tag law).
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

Base: upstream master `578e07fcbc66dc60822b55b88ba12f518df57374` (2026-09-06). Entries in branch order, oldest first.

| # | entry | upstream ref | admitted | retirement trigger |
| --- | --- | --- | --- | --- |
| 1 | `Add pane-border-type (joined\|separate\|separate-active)` — window option `pane-border-type`; `separate-active` insets every tiled pane and frames only the active one, which is what `tmux.conf` consumes to escape the two-pane shared-divider degeneracy (`redraw_mark_two_pane_colours` paints one divider half in each pane's style) | [tmux/tmux#5433](https://github.com/tmux/tmux/pull/5433) at `fe8f9ff526ab` (`redesigndavid:feat/per-window-border`, 28 commits, squashed) | 2026-09-06 | upstream merges #5433 (maintainer: a 3.9 feature) — drop the entry at the next rebase and re-pin |
| 2 | `Add CLAUDE.md: integration-branch laws and queue ledger` — this file | authored here | 2026-09-06 | permanent — it is the fork's own contract, retired only with the fork |

Every rebase that drops or adds an entry edits this table in the same push; the table and the branch are one state.

## Rebase procedure

Run from the `dotfiles` worktree (`/Users/dimitar/Projects/tmux/dotfiles` of the bare checkout `/Users/dimitar/Projects/tmux`; remote `upstream` = `tmux/tmux`, `origin` = this fork). Every git push acts as `autonomous-low-noise`; never as the personal account.

1. **Fetch upstream** — `git fetch upstream master`, and fast-forward the mirror: `git push origin upstream/master:master` (a plain push; the mirror must be a fast-forward or upstream rewrote history, which is a stop-and-look event).
2. **Rebase the queue** — `git rebase upstream/master dotfiles`. A conflict in a queue entry is resolved *in that entry* (the resolution stays one commit); a conflict too large to resolve honestly means the entry is retired early and dotfiles loses the feature until upstream lands it — say so in the queue table. When an entry's retirement trigger has fired, `git rebase -i` and `drop` it; if upstream merged a *different* shape of the change, the dotfiles consumer (`tmux.conf`, a script) moves with it in the dotfiles PR that re-pins.
3. **Build and run the regress tests of every queue entry** — `sh autogen.sh && ./configure && make`, then `cd regress && sh pane-border-type.sh` (its `TEST_TMUX` defaults to `../tmux`, i.e. the binary just built; each script runs on its own `-L test…` socket, so no running server is touched). A red test blocks the pin.
4. **Tag** — `git tag -a pin/$(date +%F) -m '<base sha>; queue: <entries>'` on the rebased head.
5. **Push** — `git push --force-with-lease origin dotfiles` and `git push origin pin/<date>`. The tag push is not optional: until it lands, the sha the next step pins is one force-push from orphaned.
6. **Re-pin dotfiles** — in `dfenerski/dotfiles` `nix/mux.nix`: `rev` = the tagged sha, `hash` = the new `fetchFromGitHub` hash (`nix-prefetch-github autonomous-low-noise tmux --rev <sha>` or the build's mismatch message), and the two marker comments above the block: `# upstream-base: <upstream master sha the queue now sits on>` and `# pin-tag: pin/<date>`. Then `make install-mux` and cut over per dotfiles' `.claude/rules/mux.md` — a tmux client and its server must be the same binary, so the server is drained and restarted, never hot-swapped.

## How dotfiles screens this fork

`make mux-upstream` in dotfiles (`scripts/mux-upstream`, git + curl only) is the re-pin **screen**: it reads the module's `rev`, `# upstream-base:` and `# pin-tag:` markers and reports, read-only, (a) whether `tmux/tmux` master has moved past the recorded upstream base and by how many commits, (b) whether the release the pin anticipates has been tagged upstream — the queue's `pane-border-type` entry retires with the 3.9 tag, at which point the fork's override can shrink or retire entirely, (c) whether the fork's `dotfiles` head still equals the pinned `rev` and the `pin/*` tag still resolves (an orphaned pin is loud), and (d) whether the unreleased CHANGES section grew since the last screen. The screen never edits the module, the profile or a running mux; a re-pin is the procedure above, run deliberately at a screen, never on a release's schedule.
