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
(ADR-0035). **`dotfiles-arch` is the rice** — the deliberate deviations from
a stock Omarchy install, stowed on top and kept in sync by `loaf`. It carries
`loaf` itself, the package manifests, and the test suite. A fourth piece,
**`shokupan`**, holds the Quickshell plugins and the Tokyo Night bar override;
it is fetched and linked by `loaf plugins` rather than stowed, because it is
publishable on its own and upstream's `omarchy plugin add` is the consumption
path for exactly that shape. Each layer is installed by a different mechanism,
and section 4 below exists because one of those mechanisms reaches into a layer
it does not own.

> The three repos were split apart on 2026-08-18. If you are reading an older
> copy of this guide that tells you to clone `shokupan` and run `loaf-install`
> from it, that is the pre-split layout — `loaf` lives in `dotfiles-arch` now.

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

  If you are building a **VM or test base** rather than a daily machine, choose
  encryption anyway. Section 4's guard only engages when root is actually on
  LUKS, so an unencrypted base silently cannot exercise the one failure this
  whole document exists to prevent — every check reports "not applicable" and
  passes vacuously. The 2026-08-18 lab base was built without encryption and
  was useless for exactly this reason
  (`docs/vm-validation-2026-08-18.md`). An encrypted one was built the same day
  and did exercise it, in both directions
  (`docs/vm-validation-luks-2026-08-18.md`); `lab/quattro-vm luks` records how
  it was made.

  The Calamares choices that produce this shape are, in the installer's own
  terms: `useSystemdHook: true` in `initcpiocfg.conf` — which is why the
  cmdline it writes is `rd.luks.uuid=`-flavoured — and a LUKS root, which makes
  `bootloader/main.py` emit `rd.luks.uuid=<uuid>` plus
  `root=/dev/mapper/luks-<uuid>` into `/etc/default/limine`. That is the
  installer half of section 4's collision, read from source rather than
  inferred from the damage.
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

### What layering the packages does *not* do

Installing `omarchy-dev` and `omarchy-settings-dev` gets you Omarchy's code and
defaults. It does **not** get you the machine the Omarchy ISO would have built,
and the difference is large. Four things the ISO does that package layering
does not — all four were measured on a from-blank-disk VM
(`docs/vm-validation-luks-2026-08-18.md`) and all four are needed before the
desktop works:

1. **`omarchy-base.packages` is never installed.** Omarchy keeps its own list of
   147 packages at `/usr/share/omarchy/install/omarchy-base.packages`, consumed
   by its ISO installer. It is not a dependency of `omarchy-dev`. On a freshly
   layered machine, **116 of the 147 were missing** — including `bluez`,
   `brightnessctl`, `grim`, `slurp`, `hyprpicker`, `hyprsunset`, `foot`, `btop`,
   `udiskie`, `wl-clipboard`, `yay`, `starship`, `nvim`, `ripgrep`, `zoxide` and
   `ufw`. 115 of those 116 are on the reference machine. Install them:

   ```bash
   sudo pacman -S --needed $(grep -vE '^\s*#|^\s*$' \
     /usr/share/omarchy/install/omarchy-base.packages)
   ```

   `loaf doctor` will **not** warn about this. Its `manifest` check covers
   `packages/chosen.packages`, which records this rice's *deviations from a
   stock Omarchy install* — and a package-layered CachyOS box is not one.

2. **Quickshell resolves to the wrong package.** `omarchy-dev` depends on the
   virtual `quickshell`. On a CachyOS base, pacman satisfies that from
   `[cachyos*]` with `quickshell`, not from `[omarchy]` with `quickshell-git` —
   which is what the reference machine runs, and what `omarchy-base.packages`
   names. The two conflict, so the list above will not install until you swap:

   ```bash
   sudo pacman -S quickshell-git    # answer y to removing quickshell
   ```

   The bar is Quickshell, so this is not cosmetic.

3. **Nothing enables a login session.** `sddm` arrives as a dependency but stays
   disabled; `/usr/share/omarchy/install/login/sddm.sh` only edits
   `/etc/pam.d/sddm`, and says in its own comment that "the ISO owns
   autologin/session state". Without `systemctl enable sddm`, the machine boots
   to a text console with no way into Hyprland.

4. **Omarchy's own migrations are pending.** A first session raises
   `Pending Omarchy Migrations — Click to run N pending migrations` (80, at
   r1773). `loaf doctor`'s Migrations section says `✓ pending none` and is
   right to: it tracks the *rice's* migrations in `dotfiles-arch/migrations`,
   not Omarchy's in `/usr/share/omarchy/migrations`.

### Create the user account *after* installing `omarchy-settings-dev`

`omarchy-settings-dev` populates `/etc/skel`. `useradd` copies `/etc/skel` at
account-creation time and never again, so an account created before the package
lands never receives it — and the rice depends on one of those files.

`.config/hypr/hyprland.lua` (tracked here) does `require("hypr.input")`.
Omarchy's `bootstrap.lua` searches `~/.local/state/?.lua`, `~/.config/?.lua`
and `$OMARCHY_PATH/?.lua`; `.conf` files are not searched. This repo tracks
`input.conf`, **not** `input.lua` — the reference machine's `input.lua` is an
untracked `/etc/skel` copy. Without it Hyprland comes up inside its red config
error box:

```
/home/<user>/.config/hypr/hyprland.lua:12: module 'hypr.input' not found
```

If the account already exists, seed it without clobbering the stow tree:

```bash
cp -rn /etc/skel/.config/. ~/.config/
cp -rn /etc/skel/.local/.  ~/.local/
```

`-n` (no-clobber) is what keeps the rice's symlinks intact; verify with
`loaf doctor`'s `symlinks` and `leaks` lines afterwards.

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

It has since been reproduced on demand in a LUKS VM
(`docs/vm-validation-luks-2026-08-18.md`) — a rebuild on an unguarded encrypted
guest, then a reboot:

```
ERROR: device '/dev/mapper/luks-<uuid>' not found. Skipping fsck.
mount: /new_root: special device /dev/mapper/luks-<uuid> does not exist.
ERROR: Failed to mount '/dev/mapper/luks-<uuid>' on real root
You are now being dropped into an emergency shell.
```

So this is no longer a single incident recalled after the fact. It is a
repeatable failure with a guard that has been watched both to hold and, with
the drop-in removed, to fail.

One detail the ADR does not mention, and which makes the trap harder rather
than easier: Omarchy ships `/etc/limine-entry-tool.d/omarchy-uki.conf` with
`ENABLE_UKI=yes`, so from the moment Omarchy is layered the boot object is a
**unified kernel image** with the cmdline baked in and hash-verified. There is
no bootloader line to edit at the menu when it goes wrong.

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

Two things to expect on a *fresh* encrypted install, both correct behaviour
that reads like a fault:

- **`loaf install` ends with the boot contract red.** The check is about the
  **running** boot, and `loaf install` writes the drop-in to a machine that is
  still running the old cmdline. It stays red until you reboot, and turns green
  on the first boot that went through the fixed cmdline. Rebooting at that
  point is safe — that reboot *is* the test.
- **`system-update` may claim the menu is broken when it is not.** ADR-0047's
  step 3 says every cmdline in `/boot/limine.conf` must carry `cryptdevice=`,
  and `.local/bin/system-update` enforces exactly that. But
  `limine-snapper-sync` records each snapper snapshot's kernel, initramfs *and
  cmdline* in `/boot/<machine-id>/limine_history/snapshots.json` at
  snapshot-creation time and replays them verbatim, so any snapshot older than
  the drop-in keeps its original cmdline. On a correctly guarded machine with
  one such snapshot, `system-update` prints

  ```
  1 of 2 boot menu cmdlines lack cryptdevice= — do NOT reboot until fixed (ADR-0047)
  ```

  The stored initramfs and the stored cmdline are a matched pair, so that entry
  is not actually broken. Trust `loaf doctor`'s boot-contract line, which looks
  at the entry you are about to boot. The `system-update` check needs narrowing
  to the non-snapshot entries; it lives in `dotfiles-arch`.

## 5. Installing the rice

With CachyOS and Omarchy both in place, clone the rice and run the installer
(`.local/bin/loaf-install`):

```bash
cd ~
git clone https://github.com/austin-karren/dotfiles-arch.git
cd dotfiles-arch
.local/bin/loaf-install
```

`git` is on a stock CachyOS base; `gh` is not, and nothing in
`packages/chosen.packages` installs it — so use `git clone` here, not
`gh repo clone`. You do not need to clone `shokupan` yourself: step 4b below
fetches it.

In order, it:

1. **Confirms the base is CachyOS** — checks for a `[cachyos*]` section in
   `/etc/pacman.conf` and installs `linux-cachyos` if it is missing (a reboot
   into it is left to the user).
2. **Confirms Omarchy matches the pin** — compares `pacman -Q omarchy` against
   `packages/omarchy.pin` (currently `4.0.0.r1744.gf002044-1`), since the rice
   reaches into upstream files whose shape is only verified against one
   version. A mismatch is a hard stop unless `--force` accepts the drift.

   > On a fresh install `--force` is not optional. The `[omarchy]` repo serves
   > only the branch tip, so the pinned build is not installable from it and
   > "install the pinned version" is not an available answer. Expect
   > `.local/bin/loaf-install --force`, and expect `loaf doctor` to carry a
   > `! version pin` warning afterwards until someone re-verifies and re-pins.
3. **Runs the boot-contract step** described in section 4.
4. **Installs chosen packages and flatpaks** from `packages/chosen.packages`
   and `packages/chosen.flatpaks`. This is also where `stow` arrives — a stock
   CachyOS base has no `stow`, and step 5 cannot do anything without it.

   > This step fails on a fresh CachyOS base. It runs
   > `pacman -S --needed --noconfirm`, and `zed` depends on the virtual
   > `vulkan-driver`. With the znver4 repos in play the first provider pacman
   > offers is `mesa-git` from `[cachyos-znver4]`; `--noconfirm` takes it, and
   > it conflicts with the already-installed `mesa`, aborting the whole
   > transaction. Install a concrete provider first —
   > `sudo pacman -S --needed vulkan-radeon` on AMD hardware,
   > `vulkan-virtio` in a VM — then re-run `loaf-install`; every step resumes.
   > The same shape will bite any virtual dependency where CachyOS ships a
   > `-git` build that sorts first.
4b. **Fetches and links the plugins repo** (`loaf plugins`) — clones
   `austin-karren/shokupan` to `~/.local/share/shokupan`, symlinks the
   Quickshell plugins, bar modules, indicators, `bin/` helpers and theme-set
   hooks into `~/.config/omarchy` and `~/.local/bin`, and *copies* (never
   symlinks) the theme override, because `omarchy-theme-set` stages user themes
   with `cp -r` and would leave a symlink dangling. Idempotent, and
   `--offline` re-asserts the links without touching the network.
5. **Runs `loaf heal`** — stows the rice, applies debloat, runs pending
   migrations. This is the same reconciliation pass the post-update hook runs
   after every `omarchy update`; installing from nothing is just healing from
   nothing.
6. **Reconciles the Widevine donation** for Helium (ADR-0038) — a failure here
   does not stop the install; DRM is severable from a working rice.
7. **Runs `loaf doctor`** as its own final step.

Every step is idempotent, so a failed run resumes by running `loaf-install`
again.

`loaf` defaults to `~/dotfiles-arch` (`LOAF_ROOT`); clone anywhere else and
export that variable first. The plugins checkout is separate and defaults to
`~/.local/share/shokupan` (`PLUGINS_ROOT`).

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

`loaf doctor` **exits non-zero when it reports problems**, so it is usable in a
script or a CI gate, not just as something to read. `test/loaf-test.sh` expects
`stow` and `shellcheck` to be installed: without `stow` a dozen tests fail on a
base that has not finished step 4, and the shellcheck test self-skips, which is
why the count is 122 on a complete machine and 121 on one missing shellcheck.

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
if ! grep -qE '^HOOKS=.*[( ]encrypt[ )]' /etc/mkinitcpio.conf.d/omarchy_hooks.conf 2>/dev/null; then
  echo "n/a — hooks do not use the busybox encrypt flavour"
elif ! grep -qE 'rd\.luks\.uuid=|root=/dev/mapper/' /proc/cmdline; then
  echo "n/a — encrypt hook present, but root is not on LUKS"
elif grep -q 'cryptdevice=' /proc/cmdline; then
  echo "cryptdevice= is live"
else
  echo "DANGER: encrypt hook + LUKS root + no cryptdevice= — do not reboot"
fi
```

The three-way split matters. An earlier version of this check tested only the
hook and `cryptdevice=`, so on a machine with the `encrypt` hook and an
*unencrypted* root — a perfectly healthy state — it reported a problem and sent
the reader off to investigate nothing. Only the last branch is the failure that
strands the machine, and it is the only one that should alarm you.

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
  here. The 2026-08-18 LUKS lab run did not close this either: it read
  Calamares' *configuration and Python source* off the ISO and reproduced the
  result with a scripted install, deliberately avoiding the GUI. So what the
  installer produces is now verified; what its screens say is still not.
- **Nothing here is verified against a machine that ran Omarchy's own ISO.**
  Section 3's "what layering the packages does not do" list was measured by
  diffing a package-layered VM against the reference machine, which is itself a
  package-layered machine of long standing. If the ISO does more than
  `omarchy-base.packages`, `sddm`, `/etc/skel` and its migrations, this
  document still does not know about it.
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
