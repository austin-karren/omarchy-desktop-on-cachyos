---
status: accepted
---

# Omarchy layered onto CachyOS, not installed from the Omarchy ISO

This machine runs CachyOS as its base and had Omarchy layered on top with
[mroboff/omarchy-on-cachyos](https://github.com/mroboff/omarchy-on-cachyos),
rather than being installed from Omarchy's own ISO. CachyOS was already the
wanted base — its kernel and package mirrors — and Omarchy is the desktop layer,
so this keeps the base distro decision and the desktop decision separate.

## Consequences

We are not on a supported Omarchy install path. `omarchy update` works and
migrations apply, but anything assuming Omarchy's own installer laid down the
system may not hold, and the third-party bridge repo is another thing that can
drift.

The installer did not work as shipped. `install-omarchy-on-cachyos.sh` and
`fetch-omarchy.sh` disagreed about where the Omarchy checkout goes — one resolved
`/../../omarchy`, the other `/../omarchy` — leaving a stale clone at `~/omarchy`
and a broken `~/.local/share/omarchy`. What fixed it:

```bash
# 1. Clear both destinations (confirm ~/omarchy is the stale clone first)
rm -rf ~/.local/share/omarchy
rm -rf ~/omarchy

# 2. Patch both scripts to agree on the path
cd ~/omarchy-on-cachyos/bin
sed -i 's|/\.\./\.\./omarchy|/../omarchy|' \
  install-omarchy-on-cachyos.sh fetch-omarchy.sh

# 3. Verify
grep -n 'omarchy"$\|cd \.\./omarchy' install-omarchy-on-cachyos.sh fetch-omarchy.sh

# 4. Re-run
./install-omarchy-on-cachyos.sh
```

Recorded because it is unobvious from the symptom, and because re-running the
installer on a new machine — or after the bridge repo updates — will hit it again
unless upstream has fixed it. The clone still sits at `~/omarchy-on-cachyos`.
