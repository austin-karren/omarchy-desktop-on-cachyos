# VM validation, 2026-08-18

Attempt to validate `docs/install-from-scratch.md` by installing it in a VM,
per the `lane/vm-validation` brief. **The headline result is that Test A's
central claim could not be tested at all**, for a reason that is a property of
the lab base rather than of anything that went wrong during the run.

Everything below was observed. Where something was not observed, it says so.

## Summary

| | Result |
|---|---|
| Premise re-verification | Done — **three divergences found**, see below |
| Test A — install path | **Blocked.** Three independent blockers, one unfixable on this base |
| Test A — boot contract (ADR-0047) | **Not provable on this base — no LUKS anywhere on the disk** |
| Test A — `loaf doctor` no-sudo | **Passed** (observed) |
| Test A — `bash test/loaf-test.sh` | **122/122 passed** (observed, on a host with `stow` + `shellcheck`) |
| Test A — `loaf plugins` fetch/link | **Passed** offline (observed) |
| Test A — bar widgets load | **Not observed** — needs a graphical Omarchy session |
| Test A — reboot after initramfs rebuild | **Not attempted** — needs sudo |
| Test B — doctor catches drift | **Passed** (observed, r1046 against an r1744 pin) |
| Test B — `loaf heal` re-asserts | **Not attempted** — needs sudo and `stow` |

## Premise re-verification

The brief's premises were checked first. Three diverge.

**1. The three repos exist and are public — CONFIRMED.** All three return 200
and are public. Note the GitHub API reports `size: 0` for the two new repos;
that is stale repository statistics, not empty repos. Their trees are populated:

- `omarchy-desktop-on-cachyos` — created 2026-08-18T20:51:27Z, `README.md`, `docs/`, `lab/`
- `dotfiles-arch` — created 2026-08-18T20:51:22Z, the rice, `.local/bin/loaf*`, `packages/`, `test/`
- `shokupan` — created 2026-08-04, repurposed by the split; now `bar/ bin/ docs/ hooks/ plugins/ themes/`

**2. `loaf-install` and `loaf-plugins` exist and behave as the brief describes —
CONFIRMED, with one correction.** The boot contract is emitted by `loaf-install`
step 3 as stated. But the brief describes the install path as "`loaf install`
plus a `loaf plugins` fetch", implying two commands. `loaf-install` has its own
**step 4b, "Plugins"**, which calls `loaf-plugins` itself. A stranger runs one
command, not two.

**3. DIVERGENCE — `docs/install-from-scratch.md` was written before the split and
still describes the pre-split world.** This is the most consequential finding of
the run, because it means the document under test cannot be followed as written.
It is a documentation defect, not a code defect: the scripts are already correct.

- §5 says `gh repo clone austin-karren/shokupan` then `cd shokupan` then
  `.local/bin/loaf-install`. Post-split, **shokupan contains no `.local/bin` and
  no `loaf-install`**; those live in `dotfiles-arch`. A stranger following §5
  reaches a "no such file" and stops.
- §5 says "`loaf` defaults to `~/shokupan` (`LOAF_ROOT`)". The script reads
  `LOAF_ROOT="${LOAF_ROOT:-$LOAF_HOME/dotfiles-arch}"` — the default is
  **`~/dotfiles-arch`**.
- §1 says "Shokupan (this repo) is the rice". In this repo that is now false
  twice over: this repo is `omarchy-desktop-on-cachyos`, and shokupan is now the
  plugins repo, not the rice.
- §5's numbered walkthrough lists **seven** steps and omits the Plugins step
  (4b) entirely, so the step that fetches shokupan is undocumented.

**4. DIVERGENCE — `libvirtd` is usable.** The brief says libvirtd is inactive and
starting it needs root, and to stop if so. `libvirtd.service` is indeed
`inactive`, but **`libvirtd.socket` is `active`**, and the first `virsh` call
socket-activates the daemon. Combined with membership in the `libvirt` group,
all `qemu:///system` operations used here — including `snapshot-create-as`,
`snapshot-revert`, `start`, `destroy` — worked with no sudo and no prompt. This
blocker did not apply.

**5. DIVERGENCE — the test count is 121 or 122 depending on the machine.** See
"What passed" below.

## What was run

The existing `shokupan-quattro-lab` disk was reused rather than rebuilt, as the
brief prefers. It was treated as precious:

1. A safety snapshot `pre-vm-validation-2026-08-18` was taken **before anything
   else**, so the pre-run state is recoverable.
2. Both pre-existing snapshots (`base-cachyos`, `quattro-layered`) are intact.
3. The disk was reverted to `pre-vm-validation-2026-08-18` at the end and the
   domain left `shut off`, which is how it was found.

No host desktop config was touched. Nothing was installed on the host.

## The three blockers on Test A

**Blocker 1 — the lab base has no encryption, so ADR-0047 cannot be exercised.**
This is the important one, and it is not a matter of missing permissions. On both
snapshots:

```
$ lsblk -o NAME,FSTYPE
vda
├─vda1 vfat      /boot
└─vda2 btrfs               <- root, directly on btrfs
$ lsblk -o FSTYPE | grep -c crypto_LUKS
0
$ cat /proc/cmdline
quiet nowatchdog splash rw rootflags=subvol=/@ root=UUID=fb034a28-...
$ cat /etc/crypttab
(no entries)
```

Root is unencrypted. There is no `rd.luks.uuid=`, no `crypto_LUKS` device, and no
`crypttab` entry. The base otherwise matches the real machine closely — the same
btrfs subvolume set (`@ @home @root @srv @cache @tmp @log`) and a separate vfat
`/boot` rather than `/boot/efi`, which is the layout §2 says only Calamares can
produce. **Encryption is the one install-time choice that was not made.**

Because ADR-0047's failure requires *both* the `encrypt` hook *and* a LUKS root,
and this disk supplies only the first, the boot contract takes its
not-applicable branch and cannot detonate. Verified directly on the
`quattro-layered` snapshot:

```
cond A (encrypt hook present)     : TRUE
cond B (LUKS-style cmdline)       : FALSE
=> step 3 takes the branch: not applicable
```

The brief calls proving the boot contract "the single most valuable thing to
prove". **On this base it is not provable, and no amount of sudo would change
that.** Proving it requires a base installed *with* LUKS — which means a fresh
Calamares run, which means the wizard path.

**Blocker 2 — `sudo` in the guest requires a password that is not recorded
anywhere.** `sudo -n true` reports a password is required; `/etc/sudoers.d` and
`/boot` are unreadable; root SSH is refused (`publickey,password`); and no
credential appears in any of the three repos or the brief. This blocks every
mutating step of `loaf install`: pacman, writing the drop-in, `limine-update`,
and the reboot that the brief correctly insists is part of the test.

**Blocker 3 — the guest has no outbound internet.** DNS resolves and the gateway
answers, but outbound 443 is refused:

```
dns          : github.com -> 140.82.112.3   OK
ping gateway : OK
tcp 443 out  : BLOCKED
```

This is precisely the failure `lab/quattro-vm`'s `check_firewall` warns about:
`ufw` is active on the host, and the DHCP and DNS allowances for `virbr0` exist
(the guest did get a lease) but the **NAT egress route rule does not**. Fixing it
is a host firewall change needing root, so it was not attempted. It was worked
around for the read-only checks by copying both repos into the guest over SSH,
which is why the results below exist at all.

## What actually passed

Everything here was observed, on a guest reverted from a known snapshot.

**`bash test/loaf-test.sh` — 122/122.** Run on a machine with `stow` and
`shellcheck` present, the suite passes completely:

```
1..122
All 122 passed.
```

Run **inside the guest**, it reports `1..121` with **12 failures**. All twelve
trace to `stow` being absent from a stock CachyOS base — failures 86 and 87
(`heal: reports a failed stow as a failure`) are the tell, and the 122nd test is
the shellcheck check, which self-skips when shellcheck is missing. `stow` **is**
listed in `packages/chosen.packages`, and `loaf-install` installs chosen packages
in step 4, before `loaf heal` in step 5 — so the ordering is correct and this is
an artifact of my having to skip step 4, not a defect. Worth knowing that the
suite is not meaningful on a base that has not completed step 4.

The suite is hermetic — `BUILD=$(mktemp -d)`, a cleanup `trap`, and every test
pointed at a throwaway `LOAF_HOME` — which is why running it on the host was safe
and touched no host config.

**`loaf doctor` — read-only and sudo-free, as §6 claims.** It ran to completion
as an unprivileged user on both snapshots and diagnosed each correctly. On the
bare base it reported 8 problems, correctly identifying an absent desktop layer
and an unstowed rice. Notably it also confirmed the plugins step end-to-end:

```
✓ plugins        6 linked from .local/share/shokupan
✓ bar widgets    no duplicate ids in bar.layout
```

**`loaf plugins` — works, and is correct in detail.** Run `--offline` against a
checkout at `~/.local/share/shokupan`, it created 13 symlinks and one theme copy,
exited 0, and was **silent on a second run** (idempotent). Every link resolves —
`find -xtype l` found no broken links. The Tokyo Night theme override was
installed as a **real file, not a symlink**, which is the behaviour
`loaf-plugins`' header comment says is required because `omarchy-theme-set`
stages user themes with `cp -r` and would otherwise leave a dangling relative
symlink.

What was **not** observed: that the bar widgets actually *render*. That needs a
running Quickshell under an Omarchy session, which needs Blockers 2 and 3 lifted.
`no duplicate ids in bar.layout` is a static check, not a load.

**Test B — the drift guard works.** The `quattro-layered` snapshot carries
`omarchy-dev 4.0.0.r1046.gd570d99-1`, exactly the older Omarchy the brief
describes, against a pin of `4.0.0.r1744.gf002044-1`. `loaf doctor` caught it:

```
! version pin    verified against 4.0.0.r1744.gf002044-1, Omarchy is
                 4.0.0.r1046.gd570d99-1 — re-verify, then re-pin
✗ debloat        4 removed launcher(s) are back
                 Basecamp / ChatGPT / HEY / WhatsApp
                 an update restored them — run 'loaf heal'
```

Reported as a warning, not a failure, which is what ADR-0028 and the test suite
both specify. The debloat line is ADR-0028's "the rice re-asserts itself"
premise showing up in the wild. **The detective half of Test B is confirmed.**
The corrective half — that `loaf heal` re-asserts and displaces nothing — was
**not** attempted: it needs `stow` and sudo.

**ADR-0047's premise is half-confirmed empirically.** On `quattro-layered`,
Omarchy has in fact written a hooks file that replaces the array with the
busybox `encrypt` flavour:

```
$ cat /etc/mkinitcpio.conf.d/omarchy_hooks.conf
HOOKS=(base udev plymouth keyboard autodetect microcode modconf kms keymap
       consolefont block encrypt filesystems fsck btrfs-overlayfs)
```

That is the half of the boot contract this base *can* demonstrate, and it does.
`/etc/limine-entry-tool.d/` exists with `omarchy-defaults.conf` and
`omarchy-uki.conf` and correctly has **no** `luks-cryptdevice.conf`, since root
is not encrypted. The other half — that a CachyOS installer writes
`rd.luks.uuid=`, and that the two together strand the machine at the next
rebuild — remains **untested here**.

## Defects found in `docs/install-from-scratch.md`

Patches for all of these are in the following commit, so they can be reviewed
separately from these findings.

1. **§5 sends the reader to the wrong repo.** `gh repo clone austin-karren/shokupan`
   / `cd shokupan` / `.local/bin/loaf-install` — none of which exists in shokupan
   post-split. **Blocks a stranger completely.**
2. **§5's `LOAF_ROOT` default is wrong** — `~/shokupan`, actually `~/dotfiles-arch`.
3. **§1 calls shokupan "the rice" and "this repo".** Both false post-split.
4. **§5's step list omits the Plugins step (4b).** Seven steps documented, eight
   in the script; the omitted one is the one that fetches shokupan.
5. **§5 assumes `gh` is installed.** It is not on a stock CachyOS base — verified
   absent in the guest — and it is **not** in `packages/chosen.packages`, so
   nothing installs it. The very first command of §5 fails on a fresh machine.
   `git` *is* present on the base, so a plain `git clone` works and needs no
   new dependency.
6. **§6's boot-contract spot-check is misleading on unencrypted machines.** The
   snippet is

   ```bash
   grep -qE '^HOOKS=.*[( ]encrypt[ )]' /etc/mkinitcpio.conf.d/omarchy_hooks.conf && \
     grep -q 'cryptdevice=' /proc/cmdline && \
     echo "cryptdevice= is live" || echo "check loaf doctor's boot-contract line"
   ```

   On the `quattro-layered` guest — encrypt hook present, root not on LUKS,
   nothing wrong — it prints **"check loaf doctor's boot-contract line"**,
   sending the reader to investigate a non-problem. It never tests whether root
   is actually LUKS, so it cannot distinguish "cryptdevice is missing and you
   are about to be stranded" from "cryptdevice is not needed". `loaf doctor`
   itself gets this right and says so precisely — `✓ boot contract  encrypt
   hook present, root is not on LUKS` — so the shell snippet is strictly worse
   than the tool it is meant to supplement.
7. **§2 does not warn that skipping encryption makes the machine untestable
   against ADR-0047.** This is exactly how the lab base came to be unable to
   prove the repo's most important claim. Anyone building a test base needs to
   be told that encryption is the choice that makes the guard meaningful.
8. **Nothing tells the reader `loaf doctor` exits non-zero** when it reports
   problems (observed: `exit=1`). §6 implies it is purely informational.

## Not reproduced

The brief's known hazard — that a stow-based re-link leaves Hyprland in
config-error state for the seconds the files are missing — was **not
investigated**. Reaching it requires `loaf heal` to run against a live Hyprland
session, which is behind Blockers 2 and 3. It is neither confirmed nor ruled out
here. Reading `loaf-plugins` alone, its `link()` does `rm -rf "$dst"` immediately
before `ln -s`, which is a window of exactly the shape described — but
`loaf-plugins` is not stow, and the hazard was reported against the stow pass, so
this is a lead rather than a finding.

## What is needed to finish this

In dependency order. The first item is the one that matters.

1. **A LUKS-encrypted base.** Nothing about ADR-0047 can be proven without one,
   and the existing base cannot be retrofitted — encryption is chosen at install
   time. This means a fresh Calamares run, which is interactive by nature. The
   brief's wizard path is the right answer, and `lab/quattro-vm calamares`
   already records the hard-won specifics (UTF-8 locale or the pacstrap module
   dies; Alt+N mnemonics; ydotool's relative-pointer trap). A wizard was **not**
   built during this run, because two other blockers would have stopped it
   immediately after and it would not have been honest to ship an untested one.
2. **The guest sudo password**, or a NOPASSWD drop-in in the lab image.
3. **The host ufw route rule**, which needs root:

   ```
   sudo ufw route allow in on virbr0 out on <wan-iface> from 192.168.122.0/24 \
     comment 'libvirt guest NAT egress'
   ```

   `lab/quattro-vm check_firewall` prints this exact command.

Items 2 and 3 are things only the user can do, so per the brief this run stops
here rather than working around them.

## State left behind

- `shokupan-quattro-lab` — **shut off**, as found.
- Snapshots: `base-cachyos` and `quattro-layered` **untouched**, plus a new
  `pre-vm-validation-2026-08-18`, which the disk is currently reverted to.
- The qcow2 grew from 13,578,534,912 to 13,628,342,272 bytes — internal snapshot
  metadata. No data was destroyed.
- Host: nothing installed, no config changed, no firewall rule added.
