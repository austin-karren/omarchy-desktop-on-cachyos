# VM validation on an encrypted base, 2026-08-18

The 2026-08-18 run (`docs/vm-validation-2026-08-18.md`) could not test this
repo's central claim, because its lab base had no LUKS anywhere on the disk.
This run built a **new, LUKS-encrypted** lab VM from the CachyOS ISO and used it
to exercise ADR-0047 in both directions.

**Headline: ADR-0047 is proven, both ways.** With the drop-in in place, an
initramfs rebuild followed by a reboot comes back. With the drop-in removed and
the initramfs rebuilt, the same machine drops to a busybox emergency shell with
root unopened. Both were observed on the serial console, not inferred.

The three blockers the previous run stopped on were all solved inside the guest,
without a host change and without asking the user.

Everything below was observed unless it says otherwise.

## Summary

| | Result |
|---|---|
| A LUKS-encrypted lab base exists | **Yes** — `shokupan-luks-lab`, built from the ISO |
| Guest sudo | **Solved** — known password *and* a NOPASSWD drop-in |
| Guest internet egress | **Solved** — qemu user-mode (SLIRP), no `virbr0`, no host firewall |
| Guest access | **Solved** — SSH over a SLIRP host-forward, no host firewall |
| ADR-0047 pre-detonation state reproduced | **Yes** — encrypt hook + `rd.luks.uuid=`-only cmdline |
| ADR-0047 failure reproduced (unguarded) | **Yes** — emergency shell, `/dev/mapper` empty |
| `loaf install` emits `luks-cryptdevice.conf` | **Yes** (observed) |
| Default boot entry carries `cryptdevice=` | **Yes** (observed) |
| *Every* rendered cmdline carries `cryptdevice=` | **No — see finding 3.** Snapshot entries do not |
| `loaf doctor` boot contract green after reboot | **Yes** (observed) |
| Detonation test — rebuild + reboot | **Passed** — machine came back |
| Reverse-guard test — drop-in removed, rebuild | **Passed** — machine failed to boot, as required |
| `bash test/loaf-test.sh` | **122/122** (observed, in the guest, with shellcheck) |
| `loaf plugins` fetch + link | **Passed** (observed, via `loaf install` step 4b) |
| `loaf doctor` fully green | **No** — see the residual-problems section |
| Desktop session / bar widgets | see the desktop section |

## The lab VM

`shokupan-luks-lab`, a **new** domain under `qemu:///system` with its own qcow2.
`shokupan-quattro-lab` and its three snapshots were not touched.

Mandatory shape, per `lab/quattro-vm`: **host-passthrough CPU** (the host is an
AMD Ryzen AI MAX+ 395, so the guest reports `x86-64-v4` and the CachyOS znver4
repos are genuinely in play) and **UEFI** (OVMF, `q35`).

### How the three old blockers were solved

**Egress — qemu user-mode networking.** The domain has no libvirt network at
all. The NIC is attached with a `<qemu:commandline>` override:

```xml
<qemu:commandline>
  <qemu:arg value='-netdev'/>
  <qemu:arg value='user,id=slirp0,hostfwd=tcp:127.0.0.1:2222-:22'/>
  <qemu:arg value='-device'/>
  <qemu:arg value='virtio-net-pci,netdev=slirp0,id=slirpnic0,bus=pcie.0,addr=0x1e'/>
</qemu:commandline>
```

This is entirely inside qemu — no `virbr0`, no NAT rule, nothing for `ufw` to
drop. Verified from inside the guest before any package work:

```
$ ip -4 addr show scope global
    inet 10.0.2.15/24 ... enp0s30
$ curl -sI https://archlinux.org | head -1
HTTP/2 200
```

`virt-install --network` does not expose SLIRP; writing the XML and
`virsh define` does. Two notes for anyone repeating it:

- The `-device` needs an explicit free PCI address, or it collides with the
  video device libvirt places at `pcie.0` slot 1.
- The `hostfwd` host port must not fall inside libvirt's VNC autoport range, or
  the domain fails to start with `Failed to find an available port`.

**Guest access — the same host-forward.** `hostfwd=tcp:127.0.0.1:2222-:22`
gives `ssh -p 2222 austin@127.0.0.1` from the host. After the ISO's live
environment was reached over the serial console, the host public key was
installed and `sshd` started there too, so the *install itself* ran over SSH
rather than through a console.

**Sudo — set at creation time.** The guest was built by this run, so the
credential is not a mystery: user `austin`, password `labpass` (root likewise),
plus `/etc/sudoers.d/10-lab-nopasswd`. LUKS passphrase `labluks`.

### Reaching a root shell on the ISO

OVMF renders the firmware and GRUB on the serial port, so
`virsh console` shows the CachyOS ISO's GRUB menu. Driven with `tui-use`:
arrow keys to stop the 2-second countdown, `e` to edit, `console=ttyS0,115200
console=tty0` appended to the `linux` line, `Ctrl+X` to boot. That yields
`CachyOS login:` on `ttyS0`; `root` logs in with no password.

Two mechanics worth recording, both cost time:

- GRUB's editor reports the cursor one screen line above where it actually is,
  so `End`/`Ctrl+E` lands on the previous line. Type a throwaway marker and
  snapshot before committing to an edit position.
- `tui-use`'s daemon is shared across sessions on this machine and has no
  per-command session flag, so every call must be preceded by
  `tui-use use <id>` or it drives whatever session another agent last touched.

## How the base was built

Calamares is a GUI and was not used. The install is a scripted Arch-style one,
run over SSH from the ISO's live environment. It deliberately reproduces what
the CachyOS Calamares *would* have produced, read out of the installer's own
source on the ISO rather than guessed:

- `/usr/lib/calamares/modules/initcpiocfg.conf` has `useSystemdHook: true`, so
  the base gets the `systemd`/`sd-encrypt` flavour.
- `/usr/lib/calamares/modules/bootloader/main.py` `get_kernel_params()` emits,
  in this order: `kernelParams` (`quiet nowatchdog`), `splash` when plymouth is
  installed, `rw`, `rootflags=subvol=<subvol>`, then — because `HOOKS` matches
  `^HOOKS.*systemd` — `rd.luks.uuid=<luksUuid>` and
  `root=/dev/mapper/<luksMapperName>`. It writes that into
  `/etc/default/limine` as `KERNEL_CMDLINE[default]+=`.

So the base's cmdline carries `rd.luks.uuid=` and **no** `cryptdevice=`, which
is exactly the premise ADR-0047 rests on. That premise is now confirmed from
the installer's source, not only from the damaged machine.

Resulting shape, confirmed in the booted guest:

```
$ lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINTS
vda                                                         60G
|-vda1                                        vfat           1G /boot
`-vda2                                        crypto_LUKS   59G
  `-luks-9f5a658b-0f66-46ce-9da5-b99ef7f0b141 btrfs         59G /var/log
                                                                /var/cache
                                                                /srv
                                                                /var/tmp
                                                                /root
                                                                /home
                                                                /
$ cat /proc/cmdline
quiet nowatchdog splash rw rootflags=subvol=/@ rd.luks.uuid=9f5a658b-... root=/dev/mapper/luks-9f5a658b-... console=tty0 console=ttyS0,115200
```

Subvolumes `@ @home @root @srv @cache @tmp @log`, a separate vfat `/boot` (not
`/boot/efi`), Limine on UEFI, `limine-entry-tool` + `limine-mkinitcpio-hook` +
`limine-snapper-sync` + snapper. That matches §2's description of the real
machine.

Two deliberate deviations from the real machine, both lab-only:

1. `console=tty0 console=ttyS0,115200` is appended so the guest is drivable
   headlessly — including the LUKS passphrase prompt, which is the only way to
   boot it at all.
2. `/etc/crypttab` is left empty. `sd-encrypt` unlocks root from
   `rd.luks.uuid=`, and a crypttab entry for an already-open root only produces
   a second prompt.

The CachyOS repos are **not** in the target after `pacstrap` — pacstrap uses the
*host* pacman.conf for the transaction and installs a stock Arch one into the
target. They were added afterwards with upstream's own
`https://mirror.cachyos.org/cachyos-repo.tar.xz`, which produced
`[cachyos-znver4] [cachyos-core-znver4] [cachyos-extra-znver4] [cachyos]`.

### Snapshots

| Snapshot | State |
|---|---|
| `luks-base` | CachyOS on LUKS, znver4 repos, nothing layered |
| **`encrypted-base`** | identical to `luks-base` — **the clone point for other lanes** |
| `omarchy-armed` | Omarchy layered, encrypt hook live, cmdline still `rd.luks.uuid=`-only, initramfs *not* yet rebuilt |
| `guarded` | `loaf install` has written the drop-in and rebuilt, not yet rebooted |
| `guarded-booted` | rebooted through the guard, boot contract green |

`encrypted-base` exists as requested. Note that libvirt refuses internal
snapshots on a pflash domain whose nvram is `raw`; the domain therefore uses an
explicit `<loader>`/`<nvram format='qcow2'>` pair with a qcow2 `OVMF_VARS`
template staged into the `images` pool, instead of `firmware='efi'`
autoselection. Swapping the nvram loses the `Limine` UEFI boot entry; the guest
still boots via `\EFI\BOOT\BOOTX64.EFI` and `limine-install` re-creates the
entry.

## ADR-0047, proven

### 1. The pre-detonation window reproduces exactly

After `pacman -S omarchy-dev omarchy-settings-dev` and nothing else:

```
$ cat /etc/mkinitcpio.conf.d/omarchy_hooks.conf
HOOKS=(base udev plymouth keyboard autodetect microcode modconf kms keymap
       consolefont block encrypt filesystems fsck btrfs-overlayfs)

$ sudo bash -c 'source /etc/mkinitcpio.conf; for f in /etc/mkinitcpio.conf.d/*.conf; do source "$f"; done; echo "${HOOKS[*]}"'
base udev plymouth keyboard autodetect microcode modconf kms keymap consolefont block encrypt filesystems fsck btrfs-overlayfs

cond A (encrypt hook present)  : TRUE
cond B (LUKS-style cmdline)    : TRUE
cryptdevice= on cmdline        : ABSENT  -> ARMED
```

`omarchy_hooks.conf` sorts after the base's own drop-in and wins outright: the
base's `systemd`/`sd-encrypt` array is gone from the effective `HOOKS`.
`/etc/limine-entry-tool.d/` is created by Omarchy at this point, containing
`omarchy-defaults.conf` and `omarchy-uki.conf` and **no** `luks-cryptdevice.conf`.

Note that `omarchy-uki.conf` sets `ENABLE_UKI=yes`, so from here on the boot
object is a unified kernel image with the cmdline **baked in**
(`/boot/EFI/Linux/omarchy_linux-cachyos.efi`). That makes the delayed-detonation
property sharper than the ADR describes: the live cmdline is not a bootloader
setting that can be edited at the menu, it is a signed-hash-verified blob
regenerated at rebuild time.

### 2. The failure, unguarded

From the `omarchy-armed` snapshot, `sudo limine-mkinitcpio` (a stand-in for the
next kernel upgrade) followed by a reboot:

```
ERROR: device '/dev/mapper/luks-9f5a658b-0f66-46ce-9da5-b99ef7f0b141' not found. Skipping fsck.
mount: /new_root: special device /dev/mapper/luks-9f5a658b-0f66-46ce-9da5-b99ef7f0b141 does not exist.
ERROR: Failed to mount '/dev/mapper/luks-9f5a658b-0f66-46ce-9da5-b99ef7f0b141' on real root
You are now being dropped into an emergency shell.
[rootfs ~]#
```

That is ADR-0047's `failed to load /dev/mapper/luks-cf6de841-…`, reproduced on
demand. It is the failure that stranded the real machine on 2026-08-14 and it
had never been seen in a lab before.

### 3. The guard, and what it does not cover

`loaf install` step 3 fired on this machine:

```
Boot contract
  writing /etc/limine-entry-tool.d/luks-cryptdevice.conf for LUKS root 9f5a658b-0f66-46ce-9da5-b99ef7f0b141

$ cat /etc/limine-entry-tool.d/luks-cryptdevice.conf
KERNEL_CMDLINE[default]+=" cryptdevice=UUID=9f5a658b-...:luks-9f5a658b-..."
```

The mechanism it relies on is real on this exact package version:
`/usr/lib/limine/limine-common-functions` sources
`/usr/share/limine-entry-tool.d/*.conf`, then `/etc/limine-entry-tool.d/*.conf`,
then `/etc/default/limine`. The drop-in is additive and `rd.luks.uuid=` survives.

**The default entry is fixed. The snapshot entries are not.** The rendered
menu after `loaf install`:

```
$ sudo grep -c 'cmdline:' /boot/limine.conf         # 2
$ sudo grep 'cmdline:' /boot/limine.conf | grep -c cryptdevice=   # 1
```

The entry that lacks it is `limine-snapper-sync`'s snapshot entry:

```
cmdline: quiet nowatchdog splash rw rootflags=subvol=/@/.snapshots/1/snapshot rd.luks.uuid=9f5a658b-... root=/dev/mapper/luks-9f5a658b-... console=tty0 console=ttyS0,115200
```

It carries none of the drop-ins' additions — not `cryptdevice=`, and not
Omarchy's own `initramfs_async=0 loglevel=0 …` either. The reason is in
`/boot/<machine-id>/limine_history/snapshots.json`: `limine-snapper-sync`
records each snapshot's kernel, initramfs **and cmdline** at snapshot-creation
time and replays them verbatim. That is defensible in itself — the stored
initramfs and the stored cmdline are a matched pair, and this particular pair
(a pre-Omarchy `sd-encrypt` initramfs with an `rd.luks.uuid=` cmdline) would
boot fine.

But it means **ADR-0047's step 3 check is wrong as written**.
`.local/bin/system-update` does:

```bash
cmdlines_total=$(sudo grep -c '^\s*cmdline:' /boot/limine.conf)
cmdlines_ok=$(sudo grep '^\s*cmdline:' /boot/limine.conf | grep -c 'cryptdevice=')
if ((cmdlines_total > 0 && cmdlines_ok == cmdlines_total)); then ... else
  "N of M boot menu cmdlines lack cryptdevice= — do NOT reboot until fixed"
```

On this machine — freshly installed, correctly guarded, `loaf doctor` green on
the boot contract — that prints **"1 of 2 boot menu cmdlines lack cryptdevice=
— do NOT reboot until fixed"**. It is a false alarm of exactly the shape the
§6 shell snippet was patched for in the previous run, and it will fire on any
encrypted machine that has a snapper snapshot older than the drop-in. Since the
message says "do NOT reboot", the cost of the false positive is high.

This is reported here rather than fixed: `system-update` lives in
`dotfiles-arch`, not in this repo.

### 4. The detonation test — the machine comes back

From `guarded` (drop-in written, UKI rebuilt with the `encrypt` hook, not yet
rebooted), `sudo systemctl reboot`:

```
BdsDxe: starting Boot0004 "Limine" ...
A password is required to access the luks-9f5a658b-0f66-46ce-9da5-b99ef7f0b141 volume:
...
Omarchy 7.1.8-1-cachyos (ttyS0)
cachy-luks-lab login:
```

The prompt wording is itself evidence: `A password is required to access the …
volume` is the **busybox `encrypt` hook**'s prompt, not systemd's `Please enter
passphrase for disk …`. The initramfs really did change flavour, and it
unlocked anyway, because `cryptdevice=` was there for it to read.

After the reboot:

```
$ cat /proc/cmdline
... rd.luks.uuid=9f5a658b-... root=/dev/mapper/luks-9f5a658b-... ... cryptdevice=UUID=9f5a658b-...:luks-9f5a658b-...

$ loaf doctor
  ✓ boot contract  encrypt hook + cryptdevice= cmdline + drop-in agree
```

Before that reboot, `loaf doctor` on the same machine correctly reported the
contract **red** — the drop-in was on disk but the *running* cmdline was still
the old one. The check is about the running boot, so it stays red until the
machine has actually been through the fixed cmdline. That is the right
behaviour and it is worth knowing, because `loaf install` finishes by running
`loaf doctor` and therefore always ends a fresh encrypted install with a red
boot-contract line.

### 5. The reverse-guard test — a guard seen to fail

From `guarded-booted`, the drop-in was removed and the initramfs rebuilt:

```
$ sudo rm /etc/limine-entry-tool.d/luks-cryptdevice.conf
$ sudo limine-mkinitcpio
$ sudo grep -c cryptdevice= /boot/limine.conf
0
```

Reboot:

```
[rootfs ~]# cat /proc/cmdline
quiet nowatchdog splash rw rootflags=subvol=/@ rd.luks.uuid=9f5a658b-... root=/dev/mapper/luks-9f5a658b-... console=tty0 console=ttyS0,115200 initramfs_async=0 loglevel=0 ...
[rootfs ~]# ls /dev/mapper
control
```

Busybox emergency shell, `/dev/mapper` holding nothing but `control` — root was
never unlocked. The guard is not merely never-seen-to-fire; removing it
reproduces the failure it exists to prevent, on the same machine, in the same
session. The domain was then reverted to `guarded-booted`.

## Running `docs/install-from-scratch.md` as a stranger

Followed as written, from the two public repos, on the fresh encrypted base.

### What worked

§5's clone-and-run is correct post-split:

```
$ cd ~ && git clone https://github.com/austin-karren/dotfiles-arch.git
$ cd dotfiles-arch && .local/bin/loaf-install
```

`git` is on the base, `.local/bin/loaf-install` exists, `LOAF_ROOT` defaults to
`~/dotfiles-arch`, and step 4b fetched and linked shokupan without a separate
command. All four defects the previous run patched are gone.

### Finding 1 — the pin cannot be satisfied, and the guide does not say so

`loaf-install` stops hard:

```
Desktop (Omarchy)
  ✗ omarchy 4.0.0.r1773.gec77971-1 is not the version this rice was verified against (4.0.0.r1744.gf002044-1)
    install the pinned version, or re-verify the rice and re-pin
    or accept the mismatch with: loaf install --force
```

The `[omarchy]` repo serves only the branch tip; `4.0.0.r1744.gf002044-1` is not
installable from it. So on any fresh install, "install the pinned version" is
not an available option, and `--force` is the *only* way forward. §5 step 2
mentions `--force` as an escape hatch for drift; it does not warn that a
from-scratch install always needs it.

### Finding 2 — step 4 fails on a fresh CachyOS base (`mesa-git` conflict)

`loaf-install` step 4 runs `pacman -S --needed --noconfirm` over
`packages/chosen.packages`. `zed` depends on the virtual `vulkan-driver`. On a
CachyOS base with the znver4 repos, the first provider pacman offers is
`mesa-git` from `[cachyos-znver4]`, `--noconfirm` takes it, and it conflicts
with the already-installed `mesa`:

```
:: There are 45 providers available for vulkan-driver:
:: Repository cachyos-znver4
   1) mesa-git  2) nvidia-utils  ...
Enter a number (default=1):
:: mesa-git-... and mesa-3:26.2.0-1 are in conflict
error: failed to prepare transaction (conflicting dependencies)
  ✗ package install failed
```

This is not specific to `zed`: it is the shape of every virtual dependency where
CachyOS's repos offer a `-git` build that sorts first. Installing a concrete
provider first (`vulkan-virtio` in a VM, `vulkan-radeon` on the real hardware)
clears it and `loaf install` then runs to completion. Nothing in the guide or in
the manifests anticipates this.

### Finding 3 — the biggest one: `omarchy-base.packages` is never installed

§3 says to layer Omarchy as packages (`omarchy-dev`, `omarchy-settings-dev`),
which is ADR-0035's whole point. But `omarchy-base.packages` — Omarchy's own
list of 147 packages, consumed by the *Omarchy ISO installer* — is not a
dependency of `omarchy-dev`. Layering the packages does not install it.

Measured on the guest immediately after §3 and §5 completed:

```
total in /usr/share/omarchy/install/omarchy-base.packages : 147
missing on the layered guest                              : 116
```

Of those 116, **115 are installed on the real machine**; only `mise-bin` is not.
The list includes things the rice itself depends on at runtime —
`bluez`, `brightnessctl`, `grim`, `slurp`, `hyprpicker`, `hyprsunset`, `foot`,
`btop`, `fastfetch`, `udiskie`, `wl-clipboard`, `yay`, `starship`, `nvim`,
`ripgrep`, `zoxide`, `ufw`.

`loaf doctor` does **not** catch it: it reported `✓ manifest  all chosen
packages installed`, because `chosen.packages` records the rice's *deviations
from a stock Omarchy install*, and a package-layered CachyOS box is not one.
The first visible symptom in the session is an Omarchy toast reading
`App failure — Error: Command not found: "udiskie"`.

This is the single largest gap between "followed `install-from-scratch.md`" and
"Austin's machine".

### Finding 4 — Omarchy layering silently picks a different Quickshell

`omarchy-dev` depends on the virtual `quickshell`. On the real machine that is
satisfied by `quickshell-git 0.3.0.r20.g28771c7-1` from `[omarchy]`. On the
guest, pacman resolved it to CachyOS's `quickshell 0.3.0-2.1` from
`[cachyos-znver4]`, and the two conflict, so `omarchy-base.packages` (which
names `quickshell-git` explicitly) cannot be installed until the swap is made
by hand. The bar is Quickshell, so this is not cosmetic: a from-scratch machine
runs a different Quickshell build than the one the rice was verified against,
and nothing warns about it.

### Finding 5 — the user account must be created after Omarchy, or Hyprland breaks

`hyprland.lua` (tracked in the rice) does `require("hypr.input")`. Omarchy's
`bootstrap.lua` searches `~/.local/state/?.lua`, `~/.config/?.lua` and
`$OMARCHY_PATH/?.lua` — `.conf` files are not searched. The rice tracks
`input.conf`; it does **not** track `input.lua`.

On the real machine the require resolves because there is an **untracked**
`~/.config/hypr/input.lua`, dated 2026-08-09, which came from Omarchy's
`/etc/skel`. On the guest the user was created before `omarchy-settings-dev`
was installed, so `/etc/skel` was empty at `useradd` time and the file never
appeared. Result, on the first session:

```
Your config has errors:
/home/austin/.config/hypr/hyprland.lua:12: module 'hypr.input' not found:
        no field package.preload['hypr.input']
        no file '/home/austin/.local/state/hypr/input.lua'
        no file '/home/austin/.config/hypr/input.lua'
(27 more...)
```

`loaf doctor` reports `✓ symlinks 86 live` and `✓ leaks no repo-only paths in
$LOAF_HOME` throughout — the rice is intact; the *seed* is missing. `input.lua`
is the only name in `/etc/skel/.config/hypr` absent from the guest's home.

The guide gives no ordering rule for user creation, and there is no step that
seeds Omarchy's `/etc/skel` into an existing account.

### Finding 6 — nothing enables a login session

§3 installs Omarchy's packages; `sddm` arrives as a dependency but stays
`disabled`, and `/usr/share/omarchy/install/login/sddm.sh` only edits
`/etc/pam.d/sddm` — the comment in it says outright that "the ISO owns
autologin/session state". A machine built by following this guide boots to a
text console with no way into Hyprland. Enabling `sddm` (and, for a lab,
autologin into `hyprland-uwsm`) is an undocumented manual step.

## Verification battery

**`bash test/loaf-test.sh` — 122/122, in the guest.**

```
1..122
All 122 passed.
```

Run before `shellcheck` was installed it reports `1..121 / All 121 passed` with
a `# skip - shellcheck not installed` line — which confirms §6's 121-vs-122
explanation exactly. Unlike the previous run, there were no `stow` failures:
`loaf install` step 4 had completed, so the suite was meaningful.

**`loaf plugins` — passed.** Run by `loaf install` step 4b; a later
`loaf plugins --offline` was silent and exited 0 (idempotent), and
`find ~/.config/omarchy ~/.local/bin -xtype l` found no broken links.
`loaf doctor` reported `✓ plugins 6 linked from .local/share/shokupan` and
`✓ bar widgets no duplicate ids in bar.layout`.

**`loaf doctor` — not green.** Residual problems on the guest, all understood:

| Line | Cause |
|---|---|
| `! version pin` | r1773 vs the r1744 pin — finding 1 |
| `✗ forks` | upstream moved under a fork or watch, at r1773 |
| `✗ wallpapers` | 15 manifest entries not on disk; `loaf wallpapers` downloads them |
| `✗ flatpaks` | no flatpak remote configured on the lab base |

None of these are boot-path or layering defects. `loaf doctor` exits non-zero
when it reports problems, as §6 now says.

## The desktop, observed

The previous run could not see a desktop at all. This one can: the host can read
the guest's framebuffer with `virsh screenshot` and drive it with
`virsh send-key`, so the session below was watched rather than assumed.

Getting there needed the four repairs above — install `omarchy-base.packages`,
swap `quickshell` for `quickshell-git`, seed `/etc/skel`'s `input.lua`, and
enable `sddm` (with a lab autologin into `hyprland-uwsm`). Every one of those is
a step `docs/install-from-scratch.md` does not have.

**Hyprland starts clean.** Before the `/etc/skel` seed, the session came up
inside Hyprland's red config-error box (`module 'hypr.input' not found`,
"27 more"). After it, no error box.

**The bar renders, and the rice's own widgets are the ones rendering.**
Compared against `~/dotfiles-arch/.config/omarchy/shell.json` on the real
machine:

| shell.json | In the guest |
|---|---|
| left: `shokupan.omenu` | **present** — the power glyph |
| left: `omarchy.workspaces` | **present** — 1–5, 1 active |
| center: `indicators` (Dictation, ScreenRecording, Reminder, NightLight, Dnd, StayAwake, Ratio) | none showing — all seven are conditional and none was active |
| center: `austinkarren.clock` | **present** — `Tuesday 10:29 PM`, i.e. the configured `"dddd h:mm AP"` |
| center: `omarchy.weather` | **present** |
| center: `omarchy.system-update` | not distinguishable in the bar; the update toast fired, so the module is live |
| right: `omarchy.tray` | empty — no tray application was running |
| right: `shokupan.notifications` | not showing |
| right: `shokupan.apexshot` | **present** — camera glyph |
| right: `omarchy.bluetooth` | absent — the VM has no bluetooth adapter (`/sys/class/bluetooth` empty) |
| right: `austinkarren.network` | **present** — globe glyph |
| right: `omarchy.audio` | **present** — muted-speaker glyph |
| right: `omarchy.monitor` | **present** — display glyph |
| right: `omarchy.power` | absent — the VM has no battery (`/sys/class/power_supply` empty) |

The two absences with a stated hardware cause (bluetooth, power) are VM
properties, not regressions. `omarchy.tray` being empty with nothing in the tray
is expected. `shokupan.notifications` not showing is **not explained** here —
four notifications were on screen at the time. It is recorded as an open
question, not as a pass.

`shokupan.omenu` and `austinkarren.clock` rendering is the important part: those
are the shokupan plugins and the rice's own bar modules, fetched and linked by
`loaf plugins` and loaded by a real Quickshell. The previous run could only
report `✓ bar widgets no duplicate ids in bar.layout`, a static check. This is
the load.

**The launcher opens.** `virsh send-key --codeset linux KEY_LEFTMETA KEY_SPACE`
brings up the Omarchy Menu with `Apps / Learn / Trigger / Style / Setup /
Install / Remove / Update / About / System`.

**One more gap the session surfaced.** Omarchy's own migrations are not run by a
package-layered install: the session raised `Pending Omarchy Migrations — Click
to run 80 pending migrations.` `loaf doctor`'s Migrations section reports
`✓ pending none` — correctly, because it tracks the *rice's* migrations in
`dotfiles-arch/migrations`, not Omarchy's 80 in `/usr/share/omarchy/migrations`.
Nothing in the guide tells a stranger those exist or that they are pending.

## State left behind

- `shokupan-quattro-lab` and its three snapshots: **untouched**. Still running,
  as found.
- `shokupan-luks-lab`: new domain, running, at the `desktop-verified` snapshot.
- New volumes in the `images` pool: `shokupan-luks-lab.qcow2` and
  `ovmf-vars-4m.qcow2` (a qcow2 `OVMF_VARS` template, needed for internal
  snapshots on a pflash domain).
- Host: nothing installed, no config changed, no firewall rule added, no sudo
  used. Everything that needed root happened inside the guest.
- Credentials, recorded deliberately because this is a throwaway lab:
  LUKS passphrase `labluks`; user `austin` / `labpass`; root / `labpass`;
  `ssh -p 2222 austin@127.0.0.1` from the host.

## What was not done

- **`loaf heal` against a live Hyprland session** was not exercised, so the
  known hazard — a stow re-link leaving Hyprland briefly in config-error state —
  is still neither confirmed nor ruled out.
- **`loaf doctor` was never fully green.** The four residual lines are explained
  above; none was chased to zero.
- **`system-update`'s false positive was not fixed**, only diagnosed. The script
  lives in `dotfiles-arch`.
- **The `omarchy-base.packages` gap was closed by hand in the guest**, not by a
  change to `loaf` or to the guide. What the right fix is — a `loaf install`
  step, a manifest, or a documented manual step — is a decision, not a finding.
