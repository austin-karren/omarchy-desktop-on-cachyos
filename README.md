# omarchy-desktop-on-cachyos

Running [Omarchy](https://omarchy.org) as a desktop layer on **CachyOS**, without
letting it manage the base.

Omarchy assumes it installed the machine. CachyOS users did not install it that
way, and the seams show — most sharply in the boot path, where Omarchy's
initramfs hooks and the CachyOS installer's kernel cmdline disagree about how to
unlock an encrypted root, and the machine stops booting one kernel upgrade later.

This repo is the layer contract: what to choose at install time, what Omarchy is
allowed to touch, and the guards that keep it that way.

**Start here: [`docs/install-from-scratch.md`](docs/install-from-scratch.md)** —
the blank-disk path, including the CachyOS installer choices that matter and why.

## Why this exists

It replaces [mroboff/omarchy-on-cachyos](https://github.com/mroboff/omarchy-on-cachyos),
which patched Omarchy's installer so it left the base alone. That approach built
the machine this repo comes from, but it targets a much older Omarchy and offers
no upgrade path. Omarchy is package-backed now, so the contract is enforced by
checks and a drop-in rather than by patching an installer.

## The three rules

1. **CachyOS owns the base** — kernel, znver4 repos, mirrorlist, bootloader,
   snapper. Never run `omarchy-channel-set`, and never let anything replace
   `/etc/pacman.d/mirrorlist` with an omarchy.org mirror: it is a frozen Arch
   snapshot, and pinning a rolling distro to it skews the two halves apart.
2. **Omarchy owns the desktop**, installed as packages. Nothing under
   `/usr/share/omarchy` is ever edited.
3. **The boot contract is guarded, not assumed** — see
   [ADR-0047](docs/adr/0047-the-boot-contract-is-guarded-not-assumed.md). This is
   the one that will strand you, and it detonates one initramfs rebuild after the
   change that caused it.

## Layout

- `docs/install-from-scratch.md` — the guide
- `docs/adr/` — the seven decisions this layer rests on (original numbering, with gaps)
- `docs/upstream/` — a drafted issue for Omarchy about the hooks/cmdline mismatch
- `lab/` — the VM rehearsal harness used to test upgrades before running them live

## Status

Extracted 2026-08-18 from a working machine that has been running this way since
2026-05, through the 3.8.x → quattro (4.0) migration and the r1046 → r1744
upgrade. The install path is documented and guarded, and ADR-0047's guard has
now been watched to hold *and* to fail on a from-blank-disk LUKS VM
([`docs/vm-validation-luks-2026-08-18.md`](docs/vm-validation-luks-2026-08-18.md)).
That run also found that layering Omarchy as packages leaves a fresh machine
well short of this one — see §3 of the install guide.
