# Battle Realms on Mac — independent verification & findings

Independent dogfood pass, 2026-05-17. Fresh agent, fresh eyes, fully isolated:
own git worktree/branch + own Whisky bottle (clone of the verified install, never
touched the shared bottle `8786C8B7`).

## Verification result — ✅ recipe works

| Step | Result |
|---|---|
| Bottle isolation | Cloned `8786C8B7` → `0644666F` via APFS `cp -Rc` (instant, zero extra disk) |
| Launch (`Fullscreen=0`, 1280×800, `WINEDEBUG=-all`, no DDraw shims) | ✅ wine process up in 4s, window in ~35s |
| Reaches UI | ✅ rendered the Skirmish setup screen — clean: map list, clan/color/team columns, map-preview thumbnail, all readable |
| Window | 1280×800 windowed on the unchanged 2624×1696 desktop, as documented |

The README/ATTEMPTS recipe reproduced exactly. No deviation needed.

## Findings — launcher robustness, ranked

### 1. Watchdog forces a hardcoded display mode on every exit — risk of leaving the wrong resolution

`launcher/battle-realms:45` restores the display with a literal
`res:2624x1696 hz:120 color_depth:8`. Two problems:

- It is **machine-specific** and undocumented as such (README only flags `BOTTLE`
  and `SCREEN_ID`). On any other Mac this sets a resolution the user never had.
- In the *windowed* recipe BR never calls `SetDisplayMode`, so the display should
  not flip in the first place. The watchdog restore is likely **dead insurance** —
  and dead insurance that can itself change the user's resolution is worse than none.

Suggest: capture the *actual* pre-launch mode (`displayplacer list`) into a temp
file and restore *that*, or drop the resolution-restore entirely and keep only the
menu-bar refresh. Verify first whether windowed BR ever flips the display at all.

### 2. `killall Dock SystemUIServer` on every quit is heavy-handed

`launcher/battle-realms:46` restarts the entire Dock + menu-bar process on every
BR exit, even a clean quit where nothing was hidden. Brief UI flash for the user.
Suggest gating it on "did the menu bar actually get hidden" or dropping it if the
windowed recipe never hides it.

### 3. Machine-specific values should be auto-detected, not hand-edited

`BOTTLE` and `SCREEN_ID` are hardcoded constants the README tells the user to edit
by hand. `SCREEN_ID` is trivially auto-detectable (`displayplacer list` → first
persistent id). `BOTTLE` could glob for the one bottle containing
`Battle Realms Complete/Battle_Realms_F.exe`. Removes the only manual-edit step and
the "expected Whisky bottle 8786C8B7" failure path.

### 4. ini-pinning `sed` silently no-ops if a key is missing

`launcher/battle-realms:33-38` rewrites `Width/Height/Depth/Fullscreen` via
`sed -E 's/^Width.*/.../'`. If `Battle_Realms.ini` ever lacks one of those lines
(fresh install, different edition) the substitution silently does nothing and BR
launches with whatever was there. Low probability, but a corrupt/fullscreen ini
would resurrect the "supported display mode" wall with no diagnostic. Suggest
verifying `Fullscreen=0` is actually set after the `sed` and alerting if not.

### 5. wine crash is invisible

If wine exits non-zero (crash, missing DLL) the launcher just ends — the `.app`
disappears with no message. The pre-flight `osascript` alert only covers
missing files. A trailing exit-code check with an alert would make a hard failure
legible instead of a silent no-op.

## Menu glitch

Not re-investigated — `battle-realms-fullfix` (Codex) owns it and was mid-fix. The
Skirmish setup screen I reached rendered cleanly, consistent with the README note
that the glitch is confined to the animated main-menu corner elements.

## Recommendation

Recipe is solid and reproducible — ship it. The launcher findings above are
robustness polish, not blockers; #1 (hardcoded display restore) is the one worth
fixing first because it can leave another Mac on the wrong resolution.
