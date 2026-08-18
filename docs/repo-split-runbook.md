# Repo split runbook

Written 2026-08-18. **Not yet executed** — this is the plan to run with fresh
budget, after which this file moves to `dotfiles-arch/docs/` as history.

Decisions already settled (Austin, 2026-08-18):

| Question | Answer |
|---|---|
| Repo identity | `shokupan` keeps its name and becomes **plugins + default theme**; the personal tree moves to a **new** `dotfiles-arch` (public) |
| History | `git filter-repo`, per-path — real commits follow the files they explain |
| ADRs | Split by concern; each repo owns the decisions it implements |
| Timing | Plan now, execute next session; VM tests validate the new layout in that same session |
| `dotfiles-arch` visibility | Public (identity already sits behind untracked includes, ADR-0003) |

## 1. The three repos

| Repo | Public | Contains | Consumed by |
|---|---|---|---|
| **omarchy-desktop-on-cachyos** | yes | The layer contract: from-scratch install, the upgrade path, the boot-contract guards, the base/desktop halves of `loaf doctor`, `lab/` (VM rehearsal) | A stranger with a blank disk. Replaces the stale [mroboff/omarchy-on-cachyos](https://github.com/mroboff/omarchy-on-cachyos) |
| **shokupan** | yes | The five `shokupan.*`/`austinkarren.*` plugins, the Tokyo Night overrides (`shell.bar.toml`), the wallpaper-pool manifest | Any Omarchy user, via `omarchy plugin add <url>` and a theme overlay — **no stow, no loaf needed** |
| **dotfiles-arch** | yes | Everything personal: hypr bindings/windows/monitors, terminals, web apps, `.bashrc`, `.XCompose`, and the whole `loaf` rice machinery | Only Austin's machines, via `loaf install` |

The consumption rule that keeps this maintainable: **`dotfiles-arch` is the only
repo `loaf` manages.** The other two are dependencies it consumes — one via the
install scripts, one via `omarchy plugin add`. No cross-repo lockstep commits;
the version pin and forks board stay in `dotfiles-arch`, where the machine is.

## 2. File routing

Counts are today's tracked inventory (167 files).

### → omarchy-desktop-on-cachyos
- `docs/install-from-scratch.md` (already written, 2026-08-18)
- `lab/` (2) — the VM rehearsal harness
- `docs/upstream/omarchy-hooks-vs-existing-cmdline.md` — the boot-contract issue draft
- From `.local/bin/loaf-doctor`: the **Base** and **Desktop** check blocks
  (repos, mirrorlist, wifi backend, kernel, boot contract, omarchy package/pin)
- From `.local/bin/loaf-install`: the base + boot-contract steps (1–3)
- New: a thin `install.sh` a stranger can run, wrapping those steps
- `packages/omarchy.pin` **stays in dotfiles-arch** — it records what *this*
  rice was verified against, not a fact about the layer contract

### → shokupan (after the tree is emptied and refilled)
- `.config/omarchy/plugins/` — `shokupan-{omenu,apexshot,capture,dpms-guard,calendar}`,
  `austinkarren.{clock,network}` (the last two are clones; carry
  `packages/forks` lines with them)
- `.config/omarchy/themes/tokyo-night/shell.bar.toml`
- `.config/omarchy/bar/{modules,indicators}/` — the indicators fork + `Ratio.qml`
- `packages/wallpapers` + `.local/bin/loaf-wallpapers` (rename: the pool is a
  publishable idea, but drop the `loaf-` prefix once it leaves the rice)
- `.config/omarchy/hooks/theme-set.d/40-theme-apexshot`
- A `README` explaining installation via `omarchy plugin add`

### → dotfiles-arch (the remainder, ~120 files)
- `.config/hypr/` (9), terminals, `.bashrc`, `.XCompose`, `.config/git`, `uwsm`,
  `wireplumber`, `mimeapps.list`, browser flags, `btop`, `starship`, `zed`, `mise`
- `.local/share/applications/` (35) — web apps, icons, the Activity entry
- `.local/bin/` minus what leaves (21 → ~18), including all of `loaf`
- `migrations/` (7), `packages/` (minus `wallpapers`), `test/`, `packages.txt`
- `CONTEXT.md` — the glossary is the rice's; the two public repos get their own
  short ones
- `.stow-local-ignore`, `.gitignore`

## 3. ADR routing (48)

Split by which repo implements the decision. A few straddle; each of those gets
a **one-paragraph stub** in the second repo pointing at the owner, never a copy.

**→ omarchy-desktop-on-cachyos (7):** 0001 (on CachyOS, not the ISO), 0028
(re-assert after upstream updates), 0033 (quattro is a rewrite), 0034 (Omarchy
is clay, CachyOS is the base), 0035 (owns the install path), 0043 (installs
bound to the pin), 0047 (the boot contract). *Stubs back to dotfiles-arch:* 0028
and 0043 name `loaf` machinery that lives there.

**→ shokupan (12):** 0006 (calendar popup), 0007 + 0040 + 0048 (wallpaper pin,
picker, pool), 0008 (aether themes), 0009 (dark bar, Tokyo-only), 0013 + 0026
(ratio toggle, zen ratio), 0029 (bar sorted by question — the surviving half),
0039 (claude usage off the bar), 0041 (rice ids out of the omarchy namespace),
0044 (plugins are the default shape). *Stub back:* 0026's Hyprland half is a
`looknfeel.lua` setting in dotfiles-arch.

**→ dotfiles-arch (29):** everything personal and every retired decision —
0002, 0003, 0004, 0005, 0010, 0011, 0012, 0014, 0015, 0016, 0017, 0018, 0019,
0020, 0021, 0022, 0023, 0024, 0025, 0027, 0030, 0031, 0032, 0036, 0037, 0038,
0042, 0045, 0046. Superseded/withdrawn ones (0004, 0005, 0011, 0012, 0027, 0031)
stay here as the historical record rather than polluting a public repo.

Renumbering: **do not renumber.** Cross-references between ADRs are dense and
numbering is cited in code comments throughout. Each repo keeps the original
numbers with gaps; each `README` index explains the gaps in one line.

## 4. Execution

Prerequisite: `sudo pacman -S --needed git-filter-repo` (verify it is absent
first: `command -v git-filter-repo`).

```bash
# 0. Safety: a full mirror to fall back on, and a tag on the pre-split state
cd ~/shokupan && git tag pre-split && git push origin pre-split
git clone --mirror https://github.com/austin-karren/shokupan.git ~/backups/shokupan-pre-split.git

# 1. dotfiles-arch — keep the personal paths, drop what leaves
git clone https://github.com/austin-karren/shokupan.git /tmp/split/dotfiles-arch
cd /tmp/split/dotfiles-arch
git filter-repo \
  --path .config/omarchy/plugins --path .config/omarchy/themes \
  --path .config/omarchy/bar --path packages/wallpapers \
  --path .local/bin/loaf-wallpapers --path lab \
  --path docs/install-from-scratch.md \
  --invert-paths
# then drop the ADRs that leave, by path, in one more filter-repo --invert-paths run

# 2. shokupan (plugins + theme) — the mirror image
git clone https://github.com/austin-karren/shokupan.git /tmp/split/shokupan-plugins
cd /tmp/split/shokupan-plugins
git filter-repo \
  --path .config/omarchy/plugins --path .config/omarchy/themes \
  --path .config/omarchy/bar --path packages/wallpapers \
  --path .local/bin/loaf-wallpapers \
  --path docs/adr/0006-... (the 12)   # keep-list, no --invert-paths
# --path-rename to flatten: plugins/ at the repo root reads better for
# `omarchy plugin add`, and the .config/omarchy/ prefix means nothing to a consumer

# 3. omarchy-desktop-on-cachyos — new repo, seeded from the 7 ADRs + lab + install doc
#    The doctor/install *code* is extracted by hand, not filter-repo: the Base
#    and Desktop blocks are sections inside files that also serve the rice.

# 4. Push: shokupan force-updates in place (it keeps its URL);
#    the other two are `gh repo create --public --source=. --push`
```

**Order matters.** Push `dotfiles-arch` and verify the machine from it *before*
force-updating `shokupan`, so there is always one working checkout of the rice.

## 5. Code that must change with the move

This is the part that breaks the machine if missed:

1. **`LOAF_ROOT` default** — `loaf` defaults to `~/shokupan`; the rice becomes
   `~/dotfiles-arch`. Every `loaf-*` script derives from it, so change the
   default in one place and grep for the literal `shokupan` elsewhere.
2. **`loaf-doctor`** — its `REPO_ONLY` regex, the symlink census, and the
   copy-class themes rule all assume one repo. Base/Desktop checks either move
   out or get sourced from the install repo.
3. **`.stow-local-ignore`** — recheck against the new tree; a stale entry
   silently stows repo furniture into `$HOME` (this bit us once already).
4. **`packages/forks`** — rice-relative paths for the two clones move to the
   plugins repo. Decide who owns the board: recommendation is dotfiles-arch
   keeps it (it is the machine's re-verification gate) and points at the plugins
   repo's files.
5. **Absolute paths** — `grep -rn '/shokupan/'` across all three trees:
   `.desktop` `Exec=` lines, `omarchy-menu.jsonc` actions, QML `bar.run` calls,
   and hooks all carry absolute paths today.
6. **`hooks/post-update.d/10-loaf-heal`** — points at the rice; update path.
7. **README/CONTEXT cross-links** in all three.

## 6. Verification (the acceptance gate)

Run in this order; do not push the plugins repo until the first two pass.

1. `loaf doctor` green from `~/dotfiles-arch`, with the same symlink count as
   today (163 at the time of writing, ±what moved).
2. `bash test/loaf-test.sh` — 122/122.
3. `omarchy plugin add <shokupan url>` in the **VM**, not on the real machine —
   proves the plugins repo is consumable by a stranger.
4. The two VM sanity tests (task #9), which are the real gate: a fresh
   blank-disk install driven only by `omarchy-desktop-on-cachyos` +
   `dotfiles-arch`, and an old-omarchy → r1744 upgrade. Opus lanes, per
   Austin's standing instruction.

## 7. Rollback

`pre-split` tag plus the mirror clone from step 0. `shokupan` is the only repo
whose history is rewritten, so it is the only one needing the mirror; the other
two can simply be deleted and recreated. Nothing on the live machine changes
until step 5 lands, and that is a symlink re-stow away from reversible.

## 8. Known open questions

- Does `shokupan` (plugins) want its own `loaf`-like helper for consumers, or is
  `omarchy plugin add` genuinely enough? Assumption: enough.
- `.local/share/applications/` web apps: personal (dotfiles-arch) — but the
  Flatpak ref handler (ADR-0032) is generally useful and may want to be public
  later. Left personal for now; revisit rather than pre-split it.
- Whether `omarchy-desktop-on-cachyos` should vendor a copy of the boot-contract
  drop-in emitter, or keep it as documentation plus a `loaf install` step. Vendor
  it: a stranger without the rice still needs the guard.
