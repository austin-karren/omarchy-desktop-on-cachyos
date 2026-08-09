---
status: accepted
---

# The rice re-asserts itself after upstream updates

This machine is three layers deep. CachyOS is the base. Omarchy is the desktop —
originally layered on with a third-party bridge repo (ADR-0001), distro packages
at `/usr/share/omarchy` since quattro (ADR-0035). The rice is everything this
repo tracks, on top of both. The two lower layers update on their own schedule,
and each can quietly undo the one above it.

Stow alone does not survive that. `stow` installs symlinks once; it has no
opinion about what happens to them afterwards, and nothing notices when the
answer is "Omarchy replaced them".

So the repo carries a small CLI — `loaf` — that dispatches to `loaf-*` scripts
the same way `omarchy` dispatches to `omarchy-*`, including the
`# loaf:summary=` convention so `loaf` with no arguments lists what exists.
Two commands matter:

- **`loaf doctor`** asserts the invariants of all three layers and changes
  nothing. Read-only by construction: no writes, no sudo, no network.
- **`loaf heal`** re-asserts the rice on top of a base that moved, and applies
  pending migrations. Wired into `~/.config/omarchy/hooks/post-update.d`, so it
  runs after every `omarchy update` without being remembered.

## The failure this exists for

Omarchy's migrations rewrite files under `~/.config`. On a stowed machine the
thing being rewritten is one of our symlinks, so Omarchy replaces the link with a
fresh default and the tracked config silently stops being live — `git status`
stays clean the whole time, because the repo file was never touched.

This has already happened here. The `.bak.<epoch>` files Omarchy leaves behind
sit next to `waybar/config.jsonc`, `hypr/bindings.conf`, `walker/config.toml` and
`uwsm/env` — all tracked paths.

`heal` never deletes. A file that displaced one of our symlinks is moved aside as
`.displaced.<epoch>`, because it is the new upstream default and usually worth
reading before discarding.

## Migrations, on a machine that cannot be rebuilt

Omarchy has 330 timestamped migrations because it ships to thousands of machines
it cannot inspect. That reasoning does not transfer to one machine you are
sitting on, and migrations were nearly rejected on those grounds.

They earn their place for a different reason: this machine cannot be rebuilt to
receive a fix. It is the machine. A change to a tracked config takes effect by
being stowed, but a change to something *outside* the repo — a stale symlink in
`$HOME`, a state file, a setting Omarchy wrote once — has nowhere to live except
a script that runs once and records that it did.

Each migration is named for the epoch second it was written, must be idempotent,
and is recorded in `~/.local/state/loaf/applied` after it succeeds. A failed
migration is not recorded, so it retries.

## Checking the base, not just ourselves

As first written, this section described `doctor` watching the bridge's CachyOS
adaptations — `sed`-ed into Omarchy's checkout, uncommitted, revertible by any
`omarchy update` git pull. Under quattro there is no checkout and no bridge
(ADR-0035), and every check built on them is deleted. What the base checks
assert now is recorded in ADR-0034: the `[cachyos*]` repos, the mirrorlist, the
wifi backend's existence, the kernel.

The lesson the old section carried is the part that survives, because the
quattro upgrade proved it twice. **A sentinel can outlive its reason** — the
bridge patched `omarchy-update-restart`, upstream rewrote that script until the
patch was obsolete, and asserting it would have failed forever on a healthy
machine. Checks get deleted when upstream makes them obsolete. And the sharper
converse, measured on this machine: **a sentinel can keep passing after its
subject dies.** The `wifi backend` check went on reporting ✓ for hours after
quattro uninstalled `iwd`, because it asserted the *setting* rather than the
invariant — a green tick on a NetworkManager with no backend at all. An expired
check is worse than a missing one; it reports health it cannot see.

## One repo, not two

The tool and the configs stay in the same repo. `doctor` reads this repo as its
source of truth — `git ls-files` for what should be symlinked,
`.stow-local-ignore` for what should not, `packages/chosen.packages` for what
should be installed. Splitting them would mean teaching the tool where the repo
is and keeping two versions in lockstep.

Migrations settle it: a migration is a statement about a specific change in this
repo, so separate repos would let the ledger and the thing it reconciles sit at
different versions.

This would flip if `loaf` were ever meant for someone else's machine, which needs
its own version and release cadence. The scripts read `LOAF_ROOT` from the
environment rather than hard-wiring a path, so that extraction stays cheap.

## Consequences

`doctor` is only as good as its invariants, and invariants encode assumptions
that expire. It reports; it does not fix, so a wrong check is noise rather than
damage — but noise that is tolerated stops being read. Delete checks that no
longer describe a real requirement.

The `post-update.d` hook deliberately never fails the update. An update that
succeeded should not be reported as broken because the rice wants attention.
