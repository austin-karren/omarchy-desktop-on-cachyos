---
status: proposed
---

# Quattro is a Hyprland rewrite, not a bar swap

Omarchy's `quattro` branch replaces Waybar with a Quickshell/QML shell. The decision is
to go, and to treat the existing ADRs as the specification: they record *why* each
binding and module exists, which is the part that has to survive. The syntax does not.

The instinct is that this is a bar migration. It is not. The bar is the cheap half.

## What the bar actually costs

Quattro's bar takes plugins in three tiers, documented in `shell/plugins/bar/README.md`:
a full QML plugin, a drop-in QML module, and a **command module** —
`{"type":"command","exec":"…","interval":5,"onClick":"…"}` — which runs a process and
reads plain text or **Waybar-style JSON** from it.

That third tier is the migration path. `tailscale-icon --status`, `ratio-toggle --status`
and `weather-icon` already emit exactly that shape. They port as-is.

Disposition of the bar and launcher ADRs:

| ADR | Under quattro |
|---|---|
| 0004, 0011 second-click dismissal | Native. **Delete the toggle wrappers.** |
| 0029 sorted by question, measured ink widths | Native layout. The measured-width CSS goes away entirely |
| 0012 / 0027 merged app + command list | Search is native (`AppLibrary.qml`); the palette is a mechanical rewrite into the plugin's config language |
| 0009 bar stays dark in every theme | Ports, modest work |
| 0013 / 0026 zen ratio toggle | Ports as a command module |
| 0031 weather Reading | **The one real regression.** Rebuild as a plugin clone |
| 0005 waybar watchdog | Retire. It guarded a Waybar GTK3 cursor-reload crash; that code no longer exists. Quattro's shell is also unsupervised, so a watchdog stays defensible — but it would guard an unmeasured failure, not the observed one |

Three more get hit that were not on the original list: **0006** (quattro's clock popup
ships a month grid, so GNOME Calendar and `calendar-toggle` become unnecessary), **0019**
(idle moves into `shell.json` as `idle.screensaver` / `idle.lock`), **0030**
(`omarchy.audio` replaces wiremix).

Four ADRs therefore end in *deleting* code. That is the good news.

## The expensive half is Hyprland

Quattro ships `config/hypr/{hyprland,bindings,input,looknfeel,monitors,autostart}.lua`.
Only `hyprsunset.conf` and `xdph.conf` stay `.conf`.

**This is not forced by Hyprland.** Confirmed: Hyprland 0.56.2 links `liblua5.4` and
`liblua.so.5.5` and ships an example `hyprland.lua`, and this machine runs `.conf` on
that same binary today. Both formats work.

It is forced by wanting Omarchy's defaults. `~/.config/hypr/hyprland.conf` opens with
thirteen `source =` lines pointing at `default/hypr/*.conf`. On quattro **none of those
files exist** — every one is `.lua`. Keeping `.conf` means forgoing the default layer and
owning all of it, which inverts what this rice is: narrow overrides on top of Omarchy.

And the defaults are no longer a flat list of settings. `bootstrap.lua`, `helpers.lua`,
`paths.lua`, `require_all.lua` make them a small framework with an API — `o.bind(...)`,
`hl.on("layer.opened", ...)`, `hl.config{...}`. Overriding it is learning that API, not
translating syntax. That reaches ADRs 0020–0025, the whole window-management layer.

## Known collision

Quattro binds `SUPER+SPACE` and `SUPER+ALT+SPACE` to its own menu. ADR-0027 put the
merged list on `SUPER+SPACE` and `ALT+SPACE`. Decide whether to keep our chords or adopt
theirs before rebuilding the palette, because ADR-0027's whole point was that either
reflex lands somewhere sensible.

## What is not known

- **No migration path was found.** Confirmed: there is no migration document and
  `shell/README.md` has no user-migration section. Not confirmed: 62 numbered
  `migrations/*.sh` exist and only the 5 newest were read, so one may exist among them.
- **The CachyOS bridge's position is unknown.** The bridge (ADR-0001) carries 9
  uncommitted edits to `install/` files in Omarchy's checkout. Those files have almost
  certainly moved on a branch 1,578 commits ahead. This is the bridge repo's call, not
  something testable locally.

## Mechanics, and why `omarchy update` will not do it

This checkout fetches exactly one tag:

    remote.origin.fetch = +refs/tags/v3.8.4:refs/tags/v3.8.4

There are no remote-tracking branches, which is why `git pull` fails on a detached HEAD
and `omarchy-branch-set` reports `invalid reference: master`. `omarchy-branch-set` offers
only master, rc and dev — quattro is not among them, even though it is now the default
branch upstream. Getting there is deliberate:

    git -C ~/.local/share/omarchy config remote.origin.fetch '+refs/heads/*:refs/remotes/origin/*'
    git -C ~/.local/share/omarchy fetch origin
    git -C ~/.local/share/omarchy switch quattro   # expect conflicts with the 9 bridge edits

## Rollback

`omarchy-v3.8.4` tags the last verified state, with a GitHub release carrying the
inventory and the bar pixel references. `~/snapshots/pre-omarchy-update-*` holds the
machine-side config and the bridge patch as a diff, and
`@home-pre-quattro-*` / `@-pre-quattro-*` are read-only btrfs snapshots at the
filesystem top level. Re-verify against a new version and tag again before trusting it.
