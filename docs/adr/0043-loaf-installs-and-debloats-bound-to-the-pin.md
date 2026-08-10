---
status: accepted
---

# Loaf installs and debloats, bound to the Omarchy pin

Two gaps, found the same day. Omarchy's defaults were trimmed by hand — HEY,
WhatsApp, Basecamp and ChatGPT removed with nothing recording the decision —
and ADR-0035 closed with the repo owing anyone who clones it an install path
that works. Both are now loaf commands.

## Debloat is a manifest, not a memory

Removing a default web app by hand only lasts until the next update:
`omarchy-refresh-applications` copies **every** launcher in
`$OMARCHY_PATH/applications/` back into `~/.local/share/applications/`. The
kept web apps survive this because they are tracked and stowed — heal turns
the overwritten real file back into our symlink. The removed ones had no
representation at all, so they quietly returned.

`packages/removed.webapps` records the decision the same way
`chosen.packages` records its opposite. `loaf debloat` re-asserts it
(idempotent, `--dry-run` supported), `loaf heal` runs it on every pass so the
post-update hook covers it, and `loaf doctor` reports a resurrected launcher
as a failure.

## Install binds to the pin

`loaf install` takes a machine with the two base layers already in place —
CachyOS installed normally, quattro layered per lab (ADR-0035) — and puts
everything this repo decides on top: `linux-cachyos` if a minimal base lacks
it, the chosen packages and flatpaks, the stowed rice, the debloat list, the
migrations. It is an orchestrator over existing idempotent commands, so a
failed run is resumed by running it again.

The one decision in it: **the installed Omarchy version must equal
`packages/omarchy.pin`, or install refuses.** The rice reaches into upstream's
files, so "does this work" is only answerable against the version it was
verified on — installing onto any other version is a claim nobody has tested.
`--force` accepts the mismatch out loud. This is also what a GitHub release
means here: a release of this repo *is* a pin, and `loaf install` is the
enforcement.

## Consequences

- The removed-defaults list is reviewable history: un-removing an app is a
  one-line revert, not archaeology.
- A fresh machine is: CachyOS install → quattro layering (lab) → `git clone`
  → `stow --adopt`? No — `loaf install`, which stows via heal and never
  adopts. `--adopt` remains a first-machine-only tool (README).
- Releases are cut against a verified pin; bumping the pin without
  re-verifying the fork manifest (ADR-0042) is the new way to lie to
  yourself, which is why doctor checks both.
