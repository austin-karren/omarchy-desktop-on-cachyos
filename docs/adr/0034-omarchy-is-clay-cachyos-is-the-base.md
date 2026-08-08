---
status: accepted
---

# Omarchy is clay; CachyOS is the base

Shokupan is its own desktop, built from Omarchy. Omarchy is source material, not an
authority: upstream changes are adopted when they are worth adopting, and the rice's
own releases lag upstream's by however long the porting takes. Mirroring closely is
the expected default, not an obligation.

CachyOS is the base — the kernel and the znver4-optimised packages are the reason this
machine is a CachyOS box with Omarchy layered on rather than an Omarchy install
(ADR-0001). **The base is not Omarchy's to manage.**

Two consequences follow, and both need enforcing rather than remembering.

## Updating the base and updating the desktop are separate operations

Omarchy does not offer that separation. `omarchy update` runs, in order:

    omarchy-snapshot create
    omarchy-update-git             # Omarchy's checkout
    omarchy-update-keyring
    omarchy-update-system-pkgs     # sudo pacman -Syyu --noconfirm
    omarchy-migrate
    omarchy-update-aur-pkgs
    omarchy-update-orphan-pkgs     # removes orphans
    omarchy-hook post-update

So it is a full, non-interactive system upgrade with a desktop update attached. It does
not endanger the CachyOS mix — `pacman -Syyu` reads this machine's own `pacman.conf`,
where the CachyOS repos sit above `core`/`extra`, so `linux-cachyos` upgrades from
CachyOS. What it destroys is **attribution**: after one combined `--noconfirm` run,
a broken desktop cannot be traced to either the base or the desktop.

Hence two commands, and two palette entries:

| Entry | Runs | Scope |
|---|---|---|
| **Update System** | `~/.local/bin/system-update` | pacman, AUR, Flatpaks — interactive, prompts visible. Never touches Omarchy |
| **Update Omarchy Desktop** | `omarchy-menu update` | Omarchy's checkout, migrations, hooks |

`system-update` is interactive on purpose. Omarchy's updater passes `--noconfirm`;
on a bridged install the pacman prompts are where replacements and `.pacnew` warnings
appear, and those are the early warning that an update is about to overwrite something
the bridge put in `/etc`.

## `omarchy-channel-set` must never run on this machine

This is the one command that would actually destroy the base. `omarchy-channel-set`
calls `omarchy-refresh-pacman`, which does:

    sudo cp -f "$OMARCHY_PATH/default/pacman/pacman-$channel.conf" /etc/pacman.conf
    sudo cp -f "$OMARCHY_PATH/default/pacman/mirrorlist-$channel" /etc/pacman.d/mirrorlist
    sudo pacman -Syyuu --noconfirm

That replaces the repo list with Omarchy's vanilla-Arch one, and the second `u` permits
**downgrades** — every znver4 package rolls back to stock Arch and `linux-cachyos` loses
the repo it comes from. It backs up to `/etc/pacman.conf.bak` first, which is thin cover
for a `-Syyuu`.

It nearly ran here on 2026-08-08: `omarchy-channel-set stable` was invoked to get the
checkout onto a branch, and only failed because `omarchy-branch-set master` errored first
and short-circuited the `&&`. The packages still updated, because `omarchy-update -y`
runs unconditionally afterwards. That was luck.

**A PATH shim cannot guard it.** Omarchy's `bin` is at PATH index 2 and `~/.local/bin`
at index 14, so a same-named script here is never reached. The guard is therefore
detective, in `loaf doctor`:

- `repos` — fails if `/etc/pacman.conf` has no `[cachyos*]` section
- `walker hold` — warns if `IgnorePkg = walker` is gone (CachyOS ships its own walker,
  which would otherwise replace Omarchy's)
- `wifi backend` — warns if `wifi.backend=iwd` is gone from NetworkManager
- `kernel` — warns if the running kernel is not a CachyOS one

The last three are bridge side effects that live in `/etc` rather than in Omarchy's
checkout, so the existing `cachyos patch` check cannot see them.

## Consequences

- The rice's releases are tagged against the Omarchy they were verified on
  (`omarchy-vX.Y.Z`), and lag upstream deliberately. `loaf doctor` reports the drift.
- A desktop-only update can leave the machine half-updated if a new Omarchy expects a
  package that is not installed — quattro needs Quickshell, for instance. The fix is
  running `system-update` afterwards, which installs from the CachyOS repos. The
  separation buys attribution, not independence.
- Anyone using this repo — a person or an agent — should read `omarchy update` as
  "update everything", and `omarchy-channel-set` as "destroy the base".
