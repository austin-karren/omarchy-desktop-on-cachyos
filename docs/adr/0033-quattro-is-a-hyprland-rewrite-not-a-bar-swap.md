---
status: accepted
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

## Addendum: measured requirements, 2026-08-08

Established against a `git worktree` of `origin/quattro` at `~/quattro-lab/omarchy`,
without moving this machine's checkout.

**Hyprland does not need upgrading.** 0.56.2 already carries the whole `hl.*` Lua API —
`hl.bind`, `hl.config`, `hl.dispatch`, `hl.dsp.exec_cmd`, `hl.animation`, `hl.device` are
all in the binary. It registers them only when a Lua config is loaded:

    [cfg] Config is lua, loading lua mgr
    [cfg] Config is NOT lua, loading regular mgr

`hyprctl dispatch 'hl.dsp.exec_cmd("true")'` returns *Invalid dispatcher* on this machine
purely because the session was started from `hyprland.conf`. The rewrite to `.lua` is what
turns the API on; it is not gated on a newer compositor. An earlier reading of that error
as a version blocker was wrong.

**Quickshell must come from the AUR, not CachyOS.** `omarchy-restart-shell` says so
directly: *"Requires our quickshell-git build; 0.3.0's kill returns immediately."* Its
restart loop relies on `quickshell kill` blocking until the process is gone.
`cachyos-extra-znver4` ships `quickshell 0.3.0-2.1`; quattro's own package list asks for
`quickshell-git`. This is the one place the CachyOS base cannot supply what quattro needs.

**Nineteen packages are missing**, of which most are ordinary (`ddcutil`, `dua-cli`,
`expac`, `foot`, `inotify-tools`, `udiskie`, `xournalpp`, `mpv-mpris`, `bluez-tools`) and
four are Omarchy's own from the `[omarchy]` repo (`omacalc`, `omacut`, `omawrite`,
`tensaku`). The `[omarchy]` repo ships neither Hyprland nor Quickshell.

**Five of the nine bridge patches target files that no longer exist** on quattro:
`install.sh`, `install/config/hardware/{network,nvidia}.sh`,
`install/config/omarchy-ai-skill.sh`, `install/config/walker-elephant.sh`,
`install/preflight/all.sh`. The walker one is likely moot anyway — quattro has its own
launcher and does not use Walker. This machine is **5,913 commits** behind quattro.

**There is no upgrade path, confirmed by absence.** Not one of quattro's 62 migrations
mentions waybar. Six create `shell.json` or quickshell state, but they migrate *between*
quattro versions and assume the shell already exists. The shell is set up by the
installer. Converting a 3.x machine in place is therefore a desktop-layer reinstall over
a live system, not an update — which is why the chosen approach is to build a fresh
quattro elsewhere, port the rice onto it, and cut over.

> **Wrong, corrected 2026-08-09.** `bin/omarchy-upgrade-to-quattro` exists — 2,444
> self-contained lines, explicitly written to be shipped in 3.8.x or run with
> `curl | bash` on older systems. Searching the migrations was the wrong place to look,
> and "confirmed by absence" was too strong a claim for a search that never covered
> `bin/`. It has since been run end to end against the lab's CachyOS base and the
> result is a working quattro desktop; see ADR-0035. The fresh-install-elsewhere
> approach was still the right call — it is what made testing this safe — but it is
> no longer the only path.

**Quattro is package-backed, not a checkout.** The upgrade installs `omarchy` and
`omarchy-settings` from `pkgs.omarchy.org` into `/usr/share/omarchy`, replacing the
`~/.local/share/omarchy` git checkout that ADR-0001 and `loaf doctor` are built around.
That is the structural change ADR-0035 has to absorb, and it is bigger than the bar.

## Addendum: the Delete bucket is done, 2026-08-09

Twelve tracked config files were removed — every one belonging to a package the
upgrade uninstalled:

    .config/waybar/{config.jsonc,style.css,theme-override.css}
    .config/walker/{config.toml,themes/omarchy-custom/{layout.xml,style.css}}
    .config/elephant/{calc,desktopapplications,symbols,websearch}.toml
    .config/elephant/menus/palette.lua
    .config/wiremix/wiremix.toml

Measured before removing, not assumed: `pacman -Qq` matches none of `waybar`,
`omarchy-walker`, `elephant`, `wiremix`; none of the four `~/.config` directories
exist, so no live symlink pointed at any of them and Stow had nothing to unlink;
and `theme-set.d/`, which generated `theme-override.css` for ADR-0009, is absent
from both `~/.config/omarchy` and `/usr/share/omarchy`. The upgrade script does
`rm -rf "$HOME/.config/elephant" … "$HOME/.config/wiremix"` itself, so this is
upstream's intent rather than local drift.

**Two of these are also the specification for the Port bucket** and must be read
before it is attempted, not merely remembered as deleted: `config.jsonc` carries
the bar's module list and order (ADR-0029, and the `custom/{ratio,weather,
tailscale,calendar}` modules of ADR-0013/0026, 0031, 0006), and `palette.lua`
carries the merged list's entry inventory (ADR-0027). Both are at tag
`omarchy-v3.8.4-prequattro`:

    git show omarchy-v3.8.4-prequattro:.config/waybar/config.jsonc
    git show omarchy-v3.8.4-prequattro:.config/elephant/menus/palette.lua

This removes 12 of the 31 not-installed symlinks `loaf doctor` reports. The rest
are the Port and Re-decide buckets and are still genuinely missing.

## Addendum: the bar is ported, 2026-08-09

The bar was called the cheap half, and it was. The layout, both custom modules
and the fixed-dark treatment all landed in one sitting against a live session,
with no restart of the shell and none of Hyprland.

Three predictions in the table above need correcting:

- **0031 weather is not "the one real regression".** Quattro's weather plugin
  already keeps a stale reading rather than blanking, which was the whole point
  of ADR-0031. The residual gap is only that the reading is in memory rather than
  on disk, so it is missing for a few seconds after a shell restart. No plugin
  clone was needed or written.
- **0009 is not a "modest" port of the same mechanism.** The hook is gone
  entirely. Quattro renders theme surfaces from templates, and user templates in
  `~/.config/omarchy/themed/` outrank the built-ins, so pinning the bar is two
  changed lines in a tracked `shell.toml.tpl`. Note also that the claim in an
  earlier draft that `theme-set.d` no longer existed was wrong — it lives at
  `~/.config/omarchy/hooks/theme-set.d/` and still fires, which is why the stale
  `20-waybar-theme-override` had to be deleted rather than left.
- **0013 ratio does not port "as a command module".** Command modules cannot be
  hidden, and the better home turned out to be the centre's hover-reveal group.
  It is a custom QML module binding to `bar.centerSectionRevealHeld`. The
  command-module tier is still the right answer for `apexshot`, which is a static
  glyph with three click actions.

**`tailscale-icon` is retired, not ported.** `omarchy.tailscale` is first-party
and does more — connection switcher, machine browser, copy actions. ADR-0029's
argument about *where* it sits was about the question it answers, and that is
unaffected by who implements it.

**The one thing that bit.** Quickshell runs module commands through `bash -lc`,
and a minimal login shell here has no `~/.local/bin` on `PATH`. A module whose
`exec` cannot be found renders as an empty box with no error in any log. Absolute
paths for anything in `.local/bin`, until `.config/uwsm/env` is re-decided —
that file is what used to put `~/.local/bin` on the session path, and it also
still points `OMARCHY_PATH` at the checkout that no longer exists.

**Still outstanding on the bar:** the model-usage widget cannot be made to
hover-reveal by configuration; it has no visibility setting and only the
package's own `indicators/` directory feeds the hover group. Matching the ratio
treatment would mean a second custom QML module that re-implements its chip and
loses its popup, which is a real trade rather than an oversight.

## Addendum: field notes from the executed port, 2026-08-09

The port this ADR specified is done — every disposition in the table above was
executed or corrected the same day (corrections annotated inline). These are the
traps the work surfaced, moved here from the working document so they survive it:

Traps found the hard way (first ones recorded in ADR-0013):

- Quickshell runs module commands through `bash -lc`, which has **no
  `~/.local/bin` on `PATH`**. A module whose `exec` is not found renders as an
  empty box with no error anywhere. Use absolute paths until `.config/uwsm/env`
  is re-decided. **The same trap holds for Omarchy Menu actions**
  (`Quickshell.execDetached`, and the floating-terminal presentation): every
  rice-owned command in `extensions/omarchy-menu.jsonc` carries an absolute path,
  measured by `update.system` failing with "command not found" inside an
  otherwise-working terminal.
- The shell's `FileView` watch on `extensions/omarchy-menu.jsonc` goes **stale
  when the file's inode is replaced** (rm + ln, stow re-stow): the old content
  stays live with no error. The symlink itself is fine — verified by summoning a
  rice-only submenu through it. After swapping the file, run `omarchy-menu
  refresh` (or any shell restart) and re-check.
- `loaf doctor` tests each tracked path with `[[ -L ]]`, so a **directory**
  symlink makes every file under it read as "replaced by a real file". Link
  files individually.
- The shell registers **new** files in `bar/modules/` only at startup. Layout
  edits hot-reload, and edits to an already-registered module hot-reload, but a
  freshly added `.qml` file renders nothing until `omarchy-restart-shell` — with
  no error, because quickshell's stdout and stderr both point at `/dev/null`.
  Cost three "failed" implementations before it was identified.
- **`hyprctl dispatch` changed languages.** With `configProvider: lua`, the
  classic `hyprctl dispatch movetoworkspacesilent …` form is parsed as Lua and
  fails; dispatchers are `hl.dsp.*` calls now, e.g.
  `hyprctl dispatch 'hl.dsp.workspace.toggle_special("calendar")'` and
  `hl.dsp.window.move({ workspace = …, window = "address:…", silent = true })`.
  `calendar-toggle` is converted and verified end to end. **`float-snap`,
  `close-surface`, `window-resize` and `window-toggle` still carry the old
  syntax** and are broken until converted — their keybindings are dead anyway,
  so they belong to the `bindings.conf` → `.lua` port, not the bar.
- QML string literals do not understand `\U000FXXXX`; supplementary-plane Nerd
  Font glyphs must be embedded as literal characters (or `\u{F00ED}`).
