# Draft issue: omarchy_hooks.conf can brick encrypted installs whose cmdline predates it

**Status: draft — not posted.** Per ADR-0044 rule 5: issue first, PR only
after upstream's temperature is known, nothing posted without an explicit go.

## Title

`omarchy_hooks.conf` switches LUKS unlocking to the `encrypt` hook without
checking the kernel cmdline's dialect — unbootable at the next initramfs
rebuild on installs that use `rd.luks.uuid=`

## Body (draft)

`install`/update writes `/etc/mkinitcpio.conf.d/omarchy_hooks.conf`, which
**replaces** the `HOOKS` array with the udev/busybox flavour, including
`encrypt`. On a machine installed by something other than the Omarchy ISO —
here, the CachyOS installer — the kernel cmdline speaks the *sd-encrypt*
dialect (`rd.luks.uuid=<uuid>`), which the `encrypt` hook ignores.

Nothing breaks immediately: the running initramfs keeps working. The machine
dies at the **next initramfs rebuild** (typically the next kernel upgrade),
stopping at `failed to load /dev/mapper/luks-<uuid>` — days after the update
that caused it, which makes it miserable to diagnose. Recovery requires a
live USB.

Observed on CachyOS (limine + UKIs + snapper), omarchy r1744, 2026-08-14.

### Two workable fixes

1. **Make the hooks follow the cmdline that already exists** (preferred):
   keep `systemd`+`sd-encrypt` when the current cmdline carries
   `rd.luks.uuid=`, switch to `encrypt` only when `cryptdevice=` is present.
   The hooks file already does conditional logic (the nvidia kms scan), so
   the shape exists.
2. **Emit the missing parameter**: when an encrypted root is detected and the
   cmdline lacks `cryptdevice=`, write a bootloader drop-in appending
   `cryptdevice=UUID=<uuid>:<mapper>` (mapper matching `root=`).

Related smaller point: because `omarchy_hooks.conf` *replaces* `HOOKS`
rather than appending, it also silently discards hooks added by other
drop-ins (e.g. `sd-btrfs-overlayfs` from `10-limine-snapper-sync.conf` on
snapper systems). Appending or filtering would compose better with the
`.d` mechanism it lives in.

Happy to test patches on an encrypted CachyOS install.
