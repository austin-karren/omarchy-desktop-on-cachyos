---
status: proposed
---

# Shokupan owns the install path; the bridge is retired

ADR-0001 installed this machine with [mroboff/omarchy-on-cachyos][bridge], a
third-party script that patches Omarchy's installer to leave CachyOS's decisions
alone. It worked, and it left nine uncommitted edits inside
`~/.local/share/omarchy` that `loaf doctor` has been watching ever since.

Under quattro the bridge stops being usable, and the reason is not that it broke —
it is that most of what it patched no longer exists. Rather than wait for it to
catch up 5,913 commits, Shokupan takes the install path over. The bridge remains
the credit for working the problem out first; ADR-0034 already states the contract
it was enforcing.

[bridge]: https://github.com/mroboff/omarchy-on-cachyos

## What the bridge does, and what survives

Measured against a `git worktree` of `origin/quattro` at `~/quattro-lab/omarchy`.
The bridge is ten concerns expressed as `sed` patches over Omarchy's installer:

| # | Bridge concern | Under quattro |
|---|---|---|
| 1 | Add `[omarchy]` to `pacman.conf`, trust key `F0134EE680CAC571` | **Still needed.** Not a patch — a setup step |
| 2 | Strip `tldr` from the base package list (CachyOS ships `tealdeer`) | **Still needed.** `install/omarchy-base.packages:121` |
| 3 | Drop `pacman.sh` from `preflight/all.sh` and `post-install/all.sh` | **Half moot, half critical.** `install/preflight/` is gone; `install/post-install/all.sh` still runs it, and it is the one that matters |
| 4 | Replace `nvidia.sh` with CachyOS 580xx driver logic | **Path moved** to `install/hardware/nvidia.sh`. Not exercised on this AMD machine |
| 5 | Drop `plymouth.sh` from `login/all.sh` | **Moot** |
| 6 | Drop `limine-snapper.sh` from `login/all.sh` | **Moot** |
| 7 | Drop `alt-bootloaders.sh` from `login/all.sh` | **Moot** |
| 8 | Remove `/etc/sddm.conf` | **Needs re-deciding.** Quattro's `login/sddm.sh` only strips keyring lines from `/etc/pam.d/sddm` |
| 9 | Disable `wpa_supplicant`, set `wifi.backend=iwd` | **Inverted.** See below |
| 10 | Pin `walker` with `IgnorePkg` | **Moot.** Quattro has no Walker at all |

`login/all.sh` on quattro is one line — `run_logged "$OMARCHY_INSTALL/login/sddm.sh"` —
which is why three of the ten evaporate together.

## The one that still matters

`install/post-install/pacman.sh` opens with:

    cp -f "$OMARCHY_PATH/default/pacman/pacman-${OMARCHY_MIRROR:-stable}.conf" /etc/pacman.conf
    cp -f "$OMARCHY_PATH/default/pacman/mirrorlist-${OMARCHY_MIRROR:-stable}" /etc/pacman.d/mirrorlist

That is the same overwrite ADR-0034 identified in `omarchy-channel-set`, except it
is unconditional and sits in the install path rather than behind a command nobody
runs. Its own comment explains why it exists — offline ISO installs use the live
medium's `pacman.conf` until this final restore — which is exactly the assumption
that does not hold when the base was installed by somebody else. It replaces the
repo list with Omarchy's vanilla-Arch one; `[cachyos*]` does not survive it.

So bridge patch #3 is not an optimisation. **It is the install path.** Everything
else on the list is tidying by comparison.

## Two concerns the bridge never had

**`install/config/snapper.sh` overwrites the root snapper config.** It runs
`install -m 0644 "$template" /etc/snapper/configs/root` and rewrites
`/etc/conf.d/snapper` to `SNAPPER_CONFIGS="root"`, then disables
`snapper-timeline.timer`. On a CachyOS install snapper is already configured and
`limine-snapper-sync` already enabled, so this silently replaces the base's
retention policy with Omarchy's. New patch needed.

**Quattro reversed the WiFi backend.** `install/hardware/network.sh` now runs
`systemctl disable iwd.service`, `iwd` is gone from `omarchy-base.packages`, and
NetworkManager is left on its default `wpa_supplicant` backend. The bridge's
patch #9 did the opposite. This machine still carries `wifi.backend=iwd` in
`/etc/NetworkManager/NetworkManager.conf`, and `loaf doctor`'s `wifi backend`
check (ADR-0034) currently warns when it is *missing* — under quattro that check
has to invert or be dropped. It is the first of ADR-0034's four base checks to
expire, and a reminder that they assert bridge side effects, not CachyOS truths.

## The shape of the replacement

Not "a better `sed` script". The bridge's fragility is that it edits Omarchy's
installer in place, so every upstream refactor breaks it silently — five of its
nine live patches now target files that do not exist, and nothing said so.

What the install path should be instead:

1. **Install CachyOS normally.** Base, kernel, bootloader, snapper — untouched by
   anything downstream.
2. **Add the `[omarchy]` repo** above nothing; it coexists with `[cachyos*]`.
3. **Run Omarchy's installer with the destructive steps disabled**, preferring
   environment variables and skip-lists over `sed` where quattro offers them —
   `install/config/snapper.sh` already reads `OMARCHY_SNAPPER_CONFIG_PATH`, which
   suggests upstream is willing to be overridden.
4. **Assert the base afterwards with `loaf doctor`**, which is where the real
   guard lives: detective, not preventive (ADR-0034).

Step 3 is the open question, and it cannot be answered by reading — it needs an
install to fail against. That is what the fresh-quattro-elsewhere build is for.
This ADR stays `proposed` until one has actually been run.

## Consequences

- ADR-0001 stays `accepted` as the record of how *this* machine was built. It is
  history, not instructions.
- The bridge's nine dirty patches in `~/.local/share/omarchy` are not carried
  forward. They are captured as a diff in `~/snapshots/pre-omarchy-update-*`.
- `loaf doctor`'s base checks are versioned against Omarchy like everything else.
  The `wifi backend` check is already known-stale for quattro.
- Anyone installing from this repo gets Shokupan's path, not the bridge's — which
  means the repo now owes them one that works.
