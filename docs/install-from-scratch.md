# Installing from scratch

The README's Install section assumes the machine already exists: `stow --adopt`
captures a live system into the repo. This document is the other direction —
blank disk to a booted, verified rice. It targets a stranger or Austin on new
hardware, not the first-machine case.

## 1. What this is layering

Three layers, bottom to top. **CachyOS is the base** — kernel, znver4 repos,
bootloader, snapper — and it is not Omarchy's to manage (ADR-0034). **Omarchy is
the desktop**, installed as ordinary distro packages (`omarchy-dev`,
`omarchy-settings-dev`) from the `[omarchy]` repo, not from Omarchy's own ISO
(ADR-0035). **Shokupan (this repo) is the rice** — the deliberate deviations from
a stock Omarchy install, stowed on top and kept in sync by `loaf`. Each layer is
installed by a different mechanism, and section 4 below exists because one of
those mechanisms reaches into a layer it does not own.

## 2. CachyOS installer choices that matter

Install CachyOS normally, before anything Omarchy-related touches the disk
(ADR-0035, "The shape of the replacement", step 1). Four choices matter enough
to call out. Where the installer's exact wording could not be verified against
a live install, the choice is described rather than a specific menu label —
see "Unverified" below.

- **No desktop environment.** Omarchy supplies Hyprland and Quickshell as its
  own packages; installing a DE first only gives Omarchy something to fight
  during layering. Pick the profile that leaves the desktop empty.
- **Encryption (LUKS) — recommended, with a caveat.** This is the one choice
  with a known sharp edge on this port. It does not break at install time — it
  breaks at the *next initramfs rebuild* after Omarchy is layered on, because
  Omarchy's hooks file assumes a different unlock mechanism than the CachyOS
  installer's cmdline provides. See section 4; do not skip it because
  encryption "worked" on first boot.
- **Limine bootloader — recommended.** ADR-0047's guard (the
  `cryptdevice=` drop-in) is a `/etc/limine-entry-tool.d/` fragment, and this
  machine's snapshot-boot story runs through `limine-snapper-sync`
  (ADR-0035, "What the first real install established"). Both assume Limine
  specifically.
- **Btrfs + snapper.** This is the rollback story the upgrade path leans on —
  `omarchy-snapshot create` and `limine-snapper-sync` both assume it, and
  ADR-0047 notes explicitly that a snapper snapshot cannot save you from the
  boot-contract failure because the ESP is not inside the btrfs snapshot.

What this machine actually has, confirmed read-only (`lsblk`, `/etc/fstab`,
`findmnt -t btrfs`, `snapper list-configs`): a LUKS-encrypted `nvme0n1p2`
unlocked to `/dev/mapper/luks-<uuid>`, btrfs subvolumes `@`, `@home`, `@root`,
`@srv`, `@cache`, `@tmp`, `@log` each mounted separately, a separate vfat
`/boot` (not `/boot/efi` — ADR-0035 records this as deliberate: the live ISO's
text installer cannot express this layout and forces `/boot/efi` instead; use
Calamares, not the TUI), and one snapper config named `root` on `/`. Swap is a
btrfs-native swapfile at `/swap/swapfile`, not a dedicated subvolume.

## 3. Layering Omarchy onto that base

Once CachyOS is installed and booted, add the `[omarchy]` repo and its trust
key, then run Omarchy's installer or `lab/omarchy-upgrade-to-quattro.patched`
against it (ADR-0035 documents the four patches this port needs relative to
stock: don't overwrite the mirrorlist, reconcile with `pacman -Syu` first,
drop the fish stack, and skip the now-unnecessary `tldr` patch). Two standing
hazards apply for the life of the machine, not just during this install
(`.local/bin/loaf-doctor`'s Base section comments are the fullest account of
both):

- **Never run `omarchy-channel-set`.** It calls `omarchy-refresh-pacman`,
  which does `cp -f .../pacman-<channel>.conf /etc/pacman.conf` followed by
  `pacman -Syyuu --noconfirm` — the second `u` permits downgrades, so every
  znver4-optimized package rolls back to stock Arch and `linux-cachyos` loses
  the repo it comes from. This nearly destroyed this machine's base on
  2026-08-08 and was stopped only by an unrelated command short-circuiting
  before it ran (ADR-0034).
- **Never let anything overwrite `/etc/pacman.d/mirrorlist` with an
  omarchy.org mirror.** Omarchy's `configure_pacman_channel` writes a single
  frozen Arch snapshot mirror. CachyOS tracks Arch rolling, so pinning
  core/extra to that snapshot puts the two halves of the system into
  permanent version skew — this is the actual root cause of what looked like
  an unrelated `libaquamarine` ABI conflict during the quattro upgrade
  (ADR-0035). `[cachyos*]` surviving in `pacman.conf` does not imply the
  mirrorlist survived; the pacman.conf edit is a surgical `awk`, the
  mirrorlist replacement is wholesale.

`loaf doctor`'s Base section checks both continuously after install (`repos`
and `mirrorlist` checks) — it is detective, not preventive, per ADR-0034.

## 4. The boot contract (ADR-0047)

This is the sharp edge in this whole document. Read it before rebooting after
any kernel or initramfs change on an encrypted install.

Omarchy writes `/etc/mkinitcpio.conf.d/omarchy_hooks.conf`, which **replaces**
the initramfs `HOOKS` array with the udev/busybox `encrypt` hook. The CachyOS
installer, by contrast, writes an *sd-encrypt*-style kernel cmdline —
`rd.luks.uuid=<uuid>` — which the `encrypt` hook does not read; it reads
`cryptdevice=UUID=<uuid>:<mapper>` instead. With the hooks swapped and the
cmdline unchanged, root never unlocks.

The trap has a delay built in: editing the hooks file changes nothing by
itself, because the running initramfs still works. The break lands at the
**next initramfs rebuild** — typically the next kernel upgrade, hours or days
later — so the update that appears to break the machine is rarely the one that
actually did. A snapper snapshot cannot save you either: the ESP holding the
kernel and initramfs is not inside the btrfs snapshot.

This machine hit exactly this failure during the r1744 upgrade (2026-08-14):
`failed to load /dev/mapper/luks-cf6de841-…`, recovered from a live USB. The
recovery runbook is linked from ADR-0047.

The fix an installer needs is a drop-in,
`/etc/limine-entry-tool.d/luks-cryptdevice.conf`, appending
`cryptdevice=UUID=<uuid>:<mapper>` to the default kernel cmdline while leaving
`rd.luks.uuid=` in place (so the machine survives a swap back to
`sd-encrypt`). `loaf install`'s "Boot contract" step emits this automatically —
it detects the `encrypt` hook plus a LUKS cmdline, reads the UUID and mapper
name from the *running* `/proc/cmdline` rather than assuming a naming
convention, writes the drop-in, and re-renders the boot menu with
`limine-update` before any kernel work can trigger a rebuild
(`.local/bin/loaf-install`, step 3). `loaf doctor`'s boot-contract check
verifies the same pre-detonation window afterwards, without sudo, on every
run.

On this machine the drop-in is already present:
`/etc/limine-entry-tool.d/luks-cryptdevice.conf` exists alongside
`omarchy-defaults.conf`, `omarchy-uki.conf`, `resume.conf`, and
`rtc-alarm.conf`, and the running cmdline carries both `rd.luks.uuid=` and
`cryptdevice=UUID=...:luks-...` — confirmed via `ls /etc/limine-entry-tool.d/`
and `/proc/cmdline`.

## 5. Installing the rice

With CachyOS and Omarchy both in place, clone the repo and run the installer
(`.local/bin/loaf-install`):

```bash
cd ~
gh repo clone austin-karren/shokupan
cd shokupan
.local/bin/loaf-install
```

In order, it:

1. **Confirms the base is CachyOS** — checks for a `[cachyos*]` section in
   `/etc/pacman.conf` and installs `linux-cachyos` if it is missing (a reboot
   into it is left to the user).
2. **Confirms Omarchy matches the pin** — compares `pacman -Q omarchy` against
   `packages/omarchy.pin` (currently `4.0.0.r1744.gf002044-1`), since the rice
   reaches into upstream files whose shape is only verified against one
   version. A mismatch is a hard stop unless `--force` accepts the drift.
3. **Runs the boot-contract step** described in section 4.
4. **Installs chosen packages and flatpaks** from `packages/chosen.packages`
   and `packages/chosen.flatpaks`.
5. **Runs `loaf heal`** — stows the rice, applies debloat, runs pending
   migrations. This is the same reconciliation pass the post-update hook runs
   after every `omarchy update`; installing from nothing is just healing from
   nothing.
6. **Reconciles the Widevine donation** for Helium (ADR-0038) — a failure here
   does not stop the install; DRM is severable from a working rice.
7. **Runs `loaf doctor`** as its own final step.

Every step is idempotent, so a failed run resumes by running `loaf-install`
again.

`loaf` defaults to `~/shokupan` (`LOAF_ROOT`); clone anywhere else and export
that variable first.

Before `loaf doctor` reports clean, three machine-side identity files need
creating — the README's "Required: git identity", "Required: compose
identity", and "Optional: shell identity" sections already document their
exact contents; this is a pointer, not a duplicate:

- `~/.gitconfig.local` — required, or commits are silently rejected.
- `~/.XCompose.local` — required, or the whole compose table fails to parse.
- `~/.bashrc.local` — optional, guarded, machine-specific shell exports.

## 6. Verifying the result

```bash
loaf doctor           # all three layers agree; read-only, no sudo
bash test/loaf-test.sh
```

`loaf doctor` green means: CachyOS repos and mirrorlist intact, Omarchy at the
pinned version (or an accepted `--force` drift), the boot contract's
pre-detonation window closed, packages and flatpaks matching their manifests,
no displaced symlinks, and the wallpaper pool and Widevine donation in their
expected states. `test/loaf-test.sh` runs the CLI's own test suite —
framework-free by design, since `loaf heal` runs from Omarchy's
`post-update.d` hook and cannot depend on anything only a test framework would
pull in.

Boot-contract spot check, worth running once by hand after the first
encrypted boot and again after any kernel upgrade:

```bash
grep -qE '^HOOKS=.*[( ]encrypt[ )]' /etc/mkinitcpio.conf.d/omarchy_hooks.conf && \
  grep -q 'cryptdevice=' /proc/cmdline && \
  echo "cryptdevice= is live" || echo "check loaf doctor's boot-contract line"
```

If `loaf doctor` ever reports the boot contract red, do **not** reboot before
resolving it — follow the recovery runbook linked from ADR-0047.

## Unverified

- **ADR-0047's "Machine reference" table**, as named in the originating brief,
  does not exist in `docs/adr/0047-the-boot-contract-is-guarded-not-assumed.md`
  or elsewhere in `docs/`. The machine facts in section 2 were gathered
  directly from `lsblk`, `/etc/fstab`, `findmnt -t btrfs`,
  `ls /etc/limine-entry-tool.d/`, `snapper list-configs`, and `/proc/cmdline`
  instead.
- **Exact CachyOS installer screen wording** (Calamares vs. the live ISO's
  text installer, exact labels for the DE-selection, encryption, bootloader,
  and filesystem screens) was not verified against a live install in this
  session. Section 2 describes the choice, not the screen. ADR-0035 records
  that the live ISO's text installer cannot express this disk layout at all
  (its UEFI-mountpoint radio group rejects input and forces `/boot/efi`) and
  that Calamares is the component that actually produced this layout — that
  much is sourced, but a fresh walkthrough of Calamares' screens was not done
  here.
- **`snapper list-configs` output was read from the live machine**, not a
  freshly-installed one; whether a stock CachyOS installer's default snapper
  config exactly matches what ADR-0035 describes Omarchy's
  `install/config/snapper.sh` overwriting (`SNAPPER_CONFIGS="root"`,
  `snapper-timeline.timer` disabled) was not independently re-verified during
  this session — ADR-0035 is the only source for that claim.
- **Whether `install/config/snapper.sh`'s overwrite still applies unmodified
  under the current Omarchy pin** (`4.0.0.r1744.gf002044-1`) was not checked
  against `/usr/share/omarchy` in this session; ADR-0035 measured it against
  an earlier `quattro` worktree.
