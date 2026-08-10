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

**Superseded by the section after this one, 2026-08-09.** Under quattro the desktop *is*
base packages, so this separation is no longer purchasable. What follows is the reasoning
as it stood on 3.8.4, kept because the two commands survive for a different reason and
that reason only makes sense against this one.

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

## The ownership ladder, 2026-08-10

The user's stance, near-verbatim: *the previous approach was scared to take
ownership over Omarchy. We can take Omarchy and take ownership over it, as long
as it stays compatible with future updates.*

"Omarchy is clay" was always the claim; the pre-quattro reality was more timid,
and partly for good reason — the checkout era made ownership genuinely hostile
(PATH order meant a same-named script never won; edits inside Omarchy's tree
vanished on `git pull`; ADR-0026 fought its value war with a poll-and-repair
loop because nothing better existed). Quattro changed the terms: config is Lua
that composes, the shell takes user plugins and QML modules, and the package
boundary makes "theirs" unambiguous. Ownership is now taken at whichever rung
does the job, and **update-compatibility is the admission test** for every
rung — never edit a file the omarchy package owns; the next upgrade reverts it
and doctor cannot even see it happened.

1. **Our value, their mechanism** — declare the deviation in a file we track,
   let upstream's machinery apply it. The zen ratio is the type specimen: 6:5
   lives in our `hyprland.lua`, gated on upstream's own flag file; the toggle
   is a straight delegation to `omarchy-hyprland-toggle`.
2. **Sanctioned extension points** — hooks, `extensions/omarchy-menu.jsonc`,
   `themed/*.tpl`, `bar/modules/*.qml`. Upstream declares the surface; we fill
   it. Costs a re-diff when upstream moves the surface.
3. **Hosted wrappers** — load an upstream component whole and adjust it from
   outside (`microphone.qml`, `audio.qml`). Upstream still ships every update
   into the wrapped part; the wrapper carries only the delta. Fragile exactly
   at the seams it reaches through, so each wrapper documents its crawl.
4. **Fork as overlay** — take the whole component and carry it
   (`plugins/shokupan-launcher/`, upstream's launcher forked). Full control,
   full re-diff burden after every upstream release. The rung of last resort,
   and the proof it is climbable.

Climb the ladder no higher than the change needs, but climb it without
apology — the lower rungs are preferred for their update cost, not out of
deference. What is *never* acceptable is rung zero: patching Omarchy's own
files in place, which is the bridge's old model and fails the admission test
by construction.

## Under quattro the axis is interactive vs unattended, 2026-08-09

Measured on the upgraded machine. **`omarchy-update-git` does not exist any more.** The
pipeline in `/usr/share/omarchy/bin/omarchy-update` is:

    omarchy-update-keyring
    omarchy-update-system-pkgs     # sudo env OMARCHY_UPDATE_PACMAN=1 pacman -Syu --noconfirm --overwrite ...
    omarchy-migrate
    omarchy-hook post-update
    omarchy-update-aur-pkgs        # yay -Sua --noconfirm
    omarchy-update-mise
    omarchy-update-orphan-pkgs
    omarchy-update-restart

Omarchy is no longer *pulled*. It is upgraded **by** `pacman -Syu`, because
`omarchy-dev` and `omarchy-settings-dev` are ordinary packages from the `[omarchy]` repo
(ADR-0035). So both rows of the table above are now false: "Update Omarchy Desktop" is a
full unattended system upgrade, and "Update System … never touches Omarchy" is impossible
— `pacman -Syu` upgrades `omarchy-dev` like anything else.

The attribution this ADR was buying cannot be bought. You cannot update the desktop
without updating the base, because the desktop *is* base packages.

Both commands still deserve to exist, but the axis has rotated:

| | `system-update` | `omarchy update` |
|---|---|---|
| pacman | **interactive** — replacements and `.pacnew` warnings visible | `--noconfirm`, plus 25 `--overwrite` paths |
| migrations / hooks | no — calls `loaf heal` directly | `omarchy-migrate`, `omarchy-hook post-update` |
| snapshot | no | `omarchy-snapshot create` |

That `--overwrite` list is the sharpest argument for keeping `system-update`. It silently
overwrites 25 files under `/etc`, including `sddm.conf.d`, `sudoers.d`,
`mkinitcpio.conf.d` and `systemd/*.conf.d` — precisely the class of change this ADR wants
a human watching, and precisely what `--noconfirm` hides.

Quattro also installs a `PreTransaction` pacman hook, `00-omarchy-update-guard.hook`,
with `AbortOnFail`: a direct `pacman -Syu` is refused unless `OMARCHY_ALLOW_DIRECT_PACMAN=1`
or `OMARCHY_UPDATE_PACMAN=1` is set. `system-update` sets the former, so it is now
load-bearing rather than forward-looking.

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
- `mirrorlist` — fails if `/etc/pacman.d/mirrorlist` points at an Omarchy mirror
- `wifi backend` — fails if NetworkManager's `wifi.backend` names a backend that is not
  installed
- `kernel` — warns if the running kernel is not a CachyOS one

These live in `/etc` rather than in Omarchy's own files, which is why the `cachyos patch`
check could never see them.

**Revised 2026-08-09.** As first written, the last three asserted *bridge side effects*
rather than truths about CachyOS, and two expired the moment the bridge did (ADR-0035).
`walker hold` is gone — quattro ships no Walker for an `IgnorePkg` to protect. `wifi
backend` is rewritten: it warned when `wifi.backend=iwd` was *missing*, and so went on
reporting ✓ after quattro removed `iwd` while the stanza naming it stayed behind — a green
tick on a NetworkManager with no backend at all. Inverting it would have swapped one
imported preference for another, so it now asserts the invariant that survives either
resolution: the configured backend exists.

`mirrorlist` is new, and takes the freed slot. ADR-0035 measured Omarchy's frozen Arch
mirror as the root cause of the entire aquamarine failure, and nothing was watching for
it. `[cachyos*]` surviving in `pacman.conf` does not imply the mirrorlist survived: the
upgrade's `pacman.conf` edit is a surgical `awk`, while the mirrorlist is replaced whole.

The pattern is the point. An expired check is worse than a missing one — it reports health
it cannot see.

## Consequences

- The rice records the Omarchy it was verified against in `packages/omarchy.pin`, and lags
  upstream deliberately. `loaf doctor` reports the drift. (Originally a `omarchy-vX.Y.Z`
  tag; see the README for why that stopped working under package versions.)
- A desktop-only update can leave the machine half-updated if a new Omarchy expects a
  package that is not installed — quattro needs Quickshell, for instance. Under
  package-backing this mostly resolves itself, because the dependency is a package
  dependency and pacman pulls it.
- Anyone using this repo — a person or an agent — should read `omarchy update` as
  "update everything, unattended", `system-update` as "update everything, watching", and
  `omarchy-channel-set` as "destroy the base".
