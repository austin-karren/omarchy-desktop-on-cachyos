---
status: accepted
---

# The boot contract is guarded, not assumed

## Context

The r1744 upgrade (2026-08-14) left the machine unbootable: `failed to load
/dev/mapper/luks-cf6de841-…`. Recovery took a live USB; the full runbook lives
as a private artifact: <https://claude.ai/code/artifact/43296fae-214a-4e91-8b22-d881a7262793>.

Root cause: Omarchy writes `/etc/mkinitcpio.conf.d/omarchy_hooks.conf`, which
**replaces** the `HOOKS` array with the udev/busybox flavour — including the
`encrypt` hook. But this machine's base was installed by the CachyOS
installer, which wrote an *sd-encrypt*-style kernel cmdline
(`rd.luks.uuid=<uuid>`). The `encrypt` hook does not read that parameter; it
reads `cryptdevice=UUID=<uuid>:<mapper>`. With the hooks swapped and the
cmdline unchanged, nothing unlocks the root volume.

The trap has a **delay**: editing the hooks file changes nothing on its own —
the running initramfs still works. The break lands at the next initramfs
rebuild, usually a kernel upgrade hours or days later. So the update that
appears to break the machine is rarely the update that actually broke it, and
a snapper snapshot cannot save you: the ESP is not in btrfs snapshots.

This is a **layer-contract violation** in ADR-0034's terms — Omarchy reaching
into the base's boot machinery and assuming an Omarchy-ISO-shaped install.
It is exactly the class of problem the retired bridge repo (ADR-0001,
ADR-0035) existed to patch.

## Decision

Keep Omarchy's hooks file untouched and make the **cmdline carry both
dialects**, guarded at every point the rice controls:

1. **The fix** is a drop-in, `/etc/limine-entry-tool.d/luks-cryptdevice.conf`,
   appending `cryptdevice=UUID=<uuid>:<mapper>` to the default kernel
   cmdline. `rd.luks.uuid=` stays too, so the machine survives a swap back to
   `sd-encrypt`. Additive and in a `.d` directory, so updates cannot clobber
   it. The mapper name after the colon must match `root=`/`resume=`.
2. **`loaf doctor`** checks the *pre-detonation window* without sudo: if
   `omarchy_hooks.conf` selects `encrypt` and the running `/proc/cmdline`
   lacks `cryptdevice=` (or the drop-in is missing/mismatched), the board goes
   red while the machine still boots. It also warns when booted on the LTS
   fallback kernel — after the recovery this machine ran LTS for a day while
   the check said ✓ to "a CachyOS kernel".
3. **`system-update`** verifies the *rendered menu* (`/boot/limine.conf`,
   root-only) right after the pacman step, where sudo is already warm — every
   cmdline must carry `cryptdevice=`, and a failure says "do NOT reboot".
4. **`loaf install`** emits the drop-in on any fresh encrypted install (UUID
   and mapper read from the running cmdline) before a kernel rebuild can
   happen, then regenerates the menu.

The fix itself is machine-side (`/etc` is outside the stow tree); what the
repo tracks is the *guard* — the same manifest-vs-record posture as packages.

## Alternatives rejected

- **Reverting the hooks to `systemd`+`sd-encrypt` in a later-sorting drop-in**:
  fights upstream on every update, and Omarchy's own drop-in logic (nvidia kms
  scan, vconsole bundling) assumes its flavour. The cmdline is the smaller,
  additive surface.
- **Only fixing it once by hand**: the delayed detonation means the next
  regression would again present days after its cause. Unguarded sentinels
  that outlive their subject are ADR-0028's lesson.

## Consequences

- Every encrypted CachyOS install running this port needs the drop-in;
  `loaf install` now owes it (ADR-0035's install path grows a base-guard
  step).
- Upstream should fix this properly — draft in
  `docs/upstream/omarchy-hooks-vs-existing-cmdline.md` (ADR-0044 rule 5:
  issues first, nothing posted without an explicit go).
