# Battle Realms on Mac — attempts log

Chronological record of what was tried to get Battle Realms running on an Apple Silicon Mac
(macOS 15.7.3, M1 Max). The win is at the bottom.

## Prior attempts (2026-04-26)

Documented in an earlier ops note. Whisky / Wine 7.7, six approaches, all failed:

| # | Approach | Result |
|---|---|---|
| 1 | Direct `wine64 Battle_Realms_F.exe` | `Could not find supported display mode` popup |
| 2 | `wine64 explorer /desktop=BR,1280x720` | same popup |
| 3 | macOS display dropped to a low-res mode | same popup |
| 4 | dgVoodoo2 v2.45 + `WINEDLLOVERRIDES=ddraw=n,b` | `opengl32` chain crash, `c0000005` |
| 5 | dgVoodoo2 v2.87 | same crash |
| 6 | cnc-ddraw | DLLs load, BR exits silently within seconds |

Conclusion at the time: blocked. Two walls — (1) display-mode enumeration, (2) DDraw shims
crash the d3d/opengl chain.

## This session (2026-05-17)

### Verifying / new tooling

- **Whisky Wine 7.7, direct launch** — reproduced wall #1: `Could not find supported display
  mode`. Whisky 2.3.5 is the last release; its Wine is GPTK-locked at 7.7. "Update Whisky"
  is a dead end.
- **Porting Kit / Wineskin** (an undocumented prior install, Wine 4.12.1 / CrossOver-19
  engine) — BR exits silently after ~5s.
- **CrossOver 26.1 (Wine 11.0)** — installed the free trial, created a fresh `winxp` bottle,
  copied BR in. `Battle_Realms_F.exe` loads every DLL (gfx + audio) then **hangs** — process
  alive, no window, indefinitely. Same in windowed and fullscreen. CrossOver's CLI suppresses
  `WINEDEBUG`, so no trace was obtainable. The "newer Wine fixes macdrv" hypothesis is
  falsified: Wine 11 does not even reach the display code, it hangs earlier.

### The breakthrough — windowed mode

Root cause, stated precisely: `Battle_Realms.ini` requested `1024×768, 16-bit, fullscreen`.
Modern Macs have no 16-bit modes. Every attempt so far fought this by swapping the *Wine*;
nobody changed what the *game asks for*.

Setting `Fullscreen=0` in `Battle_Realms.ini` makes BR skip `SetDisplayMode` entirely and
render into a normal window. Under Whisky / Wine 7.7, with the dgVoodoo shims removed and
`WINEDEBUG=-all`, **Battle Realms rendered its main menu** — the first time any attempt got
past the menu wall.

### Resolution / display tuning

| Config | Result |
|---|---|
| Windowed 1280×800, desktop 2624×1696 | works — menu + gameplay |
| Windowed 1024×768 (+ `HardwareTL=0`) | BR exits |
| Windowed 1920×1200, desktop 2624×1696 | works (window felt small) |
| Display pre-set to 1920×1200, window 1920×1200 | `could not initialize display mode` |
| Display pre-set to 1280×800 HiDPI, window 1280×800 | `could not initialize display mode` |
| Windowed 2560×1600, desktop 2624×1696 | works |

Takeaway: BR's windowed DirectDraw surface needs the desktop to be **larger** than the game
window. Pre-changing the macOS display to match the window breaks it. So: leave the display
alone, run BR as a large window.

### Launcher

- Virtual-desktop launcher (`explorer /desktop`) — BR hung, "Application Not Responding".
- Plain windowed launcher — works. This is the shipped version.

## Final working configuration

- Whisky, Wine 7.7, `winxp64` bottle.
- `Battle_Realms.ini`: `Fullscreen=0`, `Depth=32`, large windowed `Width`/`Height`,
  `HardwareTL=1`.
- No DirectDraw shims in the game folder.
- macOS display left at its normal resolution.
- `WINEDEBUG=-all`.

Outstanding: the animated main-menu elements render with stray texture quads (cosmetic Wine
DirectDraw artifact); in-game rendering is clean.
