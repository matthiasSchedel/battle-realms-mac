# Battle Realms on Apple Silicon Mac

Getting **Battle Realms** (2001 RTS — GOG "Complete" edition, including *Winter of the Wolf*)
playable on an Apple Silicon Mac via [Whisky](https://getwhisky.app).

Tested on macOS 15.7.3, M1 Max. Battle Realms is unusually hostile to modern Wine — this is
the recipe that works after a long string of dead ends (see [`ATTEMPTS.md`](ATTEMPTS.md)).

## TL;DR — the recipe

1. Install the GOG version of Battle Realms into a Whisky bottle (Wine 7.7, a `winxp64` bottle).
2. Edit `Battle_Realms.ini` → `[VideoState]` → `Fullscreen=0` (**windowed mode — this is the key**).
3. Run it as a **large window on the unchanged desktop**. Do *not* switch the macOS display resolution.
4. Delete any dgVoodoo2 / cnc-ddraw `ddraw.dll` (and `d3d8.dll`, `d3dim*.dll`) shims from the game folder.
5. Launch Wine with `WINEDEBUG=-all`.

The [`launcher/`](launcher/) folder contains a one-click `.app` that does all of this.

## Why it is hard

Battle Realms uses **DirectDraw** (the pre-Direct3D 2D API) and by default asks for a
**1024×768, 16-bit, fullscreen** mode. Modern Macs have no 16-bit display modes, and Wine's
`macdrv` DirectDraw mode enumeration on a Retina display does not expose anything Battle
Realms' fullscreen check will accept. The result is a `Could not find supported display mode`
popup, then exit.

## What does NOT work

| Approach | Result |
|---|---|
| `Fullscreen=1` (true fullscreen) | `Could not find supported display mode` |
| Changing the macOS display resolution to match the game window | `could not initialize display mode` — BR's windowed DirectDraw surface needs the desktop to be *larger* than the window |
| CrossOver 26 (Wine 11.0) | BR loads every DLL, then hangs at startup |
| Porting Kit / Wineskin (Wine 4.12.1) | BR exits silently after ~5s |
| dgVoodoo2 / cnc-ddraw DirectDraw shims | crash Wine's `opengl32` chain (`c0000005`) or silent exit |
| Wine virtual desktop (`explorer /desktop`) | BR hangs ("Application Not Responding") |

## What works

**Windowed mode.** With `Fullscreen=0`, Battle Realms never calls `SetDisplayMode` — it
creates an ordinary window and blits into it, so the entire "supported display mode" wall
disappears.

Two extra constraints, learned the hard way:

- **Do not change the macOS display resolution.** If you drop the display to the game's
  size, BR's windowed surface can no longer be created (`could not initialize display mode`).
  BR must run as a window *smaller than* the current desktop.
- **`WINEDEBUG=-all`.** With verbose logging on, BR's startup display-mode scan walks Wine's
  huge synthesized Retina mode list and crawls. Silencing the debug channels keeps it fast.

## Setup

1. **Install Whisky** — <https://getwhisky.app>. Create a bottle (Windows version: `winxp64`).
2. **Install Battle Realms** — run the GOG installer (`setup_battle_realms_complete_*.exe`)
   inside the bottle.
3. **Clean the game folder** — in `…/GOG.com/Battle Realms Complete/`, remove any of
   `ddraw.dll`, `ddraw.ini`, `d3d8.dll`, `d3dim.dll`, `d3dim700.dll`, `dgVoodoo.conf` if a
   previous attempt left DirectDraw shims behind. BR must use Wine's built-in `ddraw`.
4. **Edit `Battle_Realms.ini`** → `[VideoState]`:
   ```ini
   Width      =2560
   Height     =1600
   Depth      =32
   Fullscreen =0
   HardwareTL =1
   ```
   Width/Height = your preferred window size (must be smaller than your desktop resolution).
5. **Launch** from the game directory:
   ```bash
   cd "…/Bottles/<UUID>/drive_c/Program Files (x86)/GOG.com/Battle Realms Complete"
   WINEPREFIX="…/Bottles/<UUID>" WINEDEBUG=-all "…/Whisky/Libraries/Wine/bin/wine64" Battle_Realms_F.exe
   ```

## The launcher

[`launcher/battle-realms`](launcher/battle-realms) is a shell script wrapped in a macOS
`.app` bundle, so the game starts with a double-click. It:

- forces `Fullscreen=0` and the window size on every launch,
- launches BR with `WINEDEBUG=-all`,
- runs a background watchdog that restores the display resolution and refreshes the menu
  bar when BR exits (BR's intro can briefly flip the display).

The script has two machine-specific values you must set for your own Mac:

- `BOTTLE` — your Whisky bottle's UUID path.
- `SCREEN_ID` — your display's persistent ID, from `displayplacer list`
  (install via `brew install displayplacer`).

To assemble the `.app`: create `Battle Realms.app/Contents/{MacOS,Resources}`, drop
`battle-realms` into `MacOS/` (and `chmod +x` it), `Info.plist` into `Contents/`, and an
`.icns` icon into `Resources/`.

## Known issue — glitchy main menu

Battle Realms' animated main-menu elements (the corner preview animations) render with stray
texture quads under Wine's DirectDraw. This is **cosmetic** — the in-game isometric
rendering is clean and the game is fully playable. The menu buttons stay in their correct
positions, so it remains usable. No fix found; it appears at every tested resolution.

## Credits

Worked out on 2026-05-17. Full blow-by-blow in [`ATTEMPTS.md`](ATTEMPTS.md).
