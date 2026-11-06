# The existing DMS `bongoCat` plugin

Research for [#9](https://github.com/PascalBrus/dms-bongocat/issues/9). Map: [#1](https://github.com/PascalBrus/dms-bongocat/issues/1).

Source: `hthienloc/dms-plugins`, folder `bongoCat/`, read at commit
[`b7ec1d6`](https://github.com/hthienloc/dms-plugins/tree/b7ec1d6cd0fa1d42cab7c2f5996305c34897275c/bongoCat)
(2026-11-06). Line numbers below refer to that commit. I read the code without
running it.

Short links used below:

- `IEL` = [`shared/InputEventListener.qml`](https://github.com/hthienloc/dms-plugins/blob/b7ec1d6cd0fa1d42cab7c2f5996305c34897275c/bongoCat/shared/InputEventListener.qml)
- `BW` = [`BongoWidget.qml`](https://github.com/hthienloc/dms-plugins/blob/b7ec1d6cd0fa1d42cab7c2f5996305c34897275c/bongoCat/BongoWidget.qml) (bar widget, 1456 lines)
- `BDW` = [`BongoDesktopWidget.qml`](https://github.com/hthienloc/dms-plugins/blob/b7ec1d6cd0fa1d42cab7c2f5996305c34897275c/bongoCat/BongoDesktopWidget.qml) (desktop widget, 441 lines)
- `BS` = [`BongoSettings.qml`](https://github.com/hthienloc/dms-plugins/blob/b7ec1d6cd0fa1d42cab7c2f5996305c34897275c/bongoCat/BongoSettings.qml)

## Summary

- **It works, but it breaks the privacy rule.** It runs `libinput debug-events --show-keycodes` (or `evtest <device>`). The flag turns off libinput's own key masking. The full key name (`KEY_A`) and keycode then reach QML/JS in every widget instance.
- **Space does slam both arms.** Space, Enter and keypad Enter all do, behind the `doubleSlam` toggle, which is on by default.
- **Placement:** it has a bar widget and a desktop widget. For overlay and click-through it relies on DMS's per-instance toggles and adds no code of its own.
- **Art licence is unclear.** The code is MIT. The art is credited to StrayRogue and StrawberryMatchaArt, but neither gives a licence.
- **Very active:** a single maintainer, about 30 commits to this folder since 2026-09-19.

## 1. Manifest

From [`plugin.json`](https://github.com/hthienloc/dms-plugins/blob/b7ec1d6cd0fa1d42cab7c2f5996305c34897275c/bongoCat/plugin.json):

- `id: bongoCat`, version `2.5.0`, `requires_dms: ">=2.0.0"`, `license: MIT`, author Loc Huynh.
- `type: "composite"`, capabilities `dankbar-widget` and `desktop-widget`.
- Components: `widget: BongoWidget.qml`, `desktop: BongoDesktopWidget.qml`, settings page `BongoSettings.qml`.
- `permissions: ["settings_read", "settings_write", "process"]`. It needs `process` because all input comes from child processes.

## 2. How key data flows (privacy)

### 2.1 Input source

`IEL:85-93` starts one long-running child process:

| Device setting (`selectedDevicePath`) | Command |
|---|---|
| `"all"` (default, "All Keyboards (Auto)") | `libinput debug-events --show-keycodes` |
| a specific `/dev/input/eventN` | `evtest /dev/input/eventN` |

Optional mouse support adds a second `libinput debug-events` process (`IEL:162-189`). That one runs without `--show-keycodes` and only handles pointer events.

Prerequisites, from the [README](https://github.com/hthienloc/dms-plugins/blob/b7ec1d6cd0fa1d42cab7c2f5996305c34897275c/bongoCat/README.md):

- `evtest` and/or the `libinput` CLI must be installed.
- The user must be in the `input` group (`usermod -aG input`).

The plugin checks for these with `sh -c "command -v …"` and `id -nG | … grep -qx input` (`IEL:57-82`) and shows a warning badge if one is missing.

Device listing runs an inline Python script through `python3 -c`. The script reads `/proc/bus/input/devices` and filters by name patterns (see [`shared/InputDeviceScanner.qml`](https://github.com/hthienloc/dms-plugins/blob/b7ec1d6cd0fa1d42cab7c2f5996305c34897275c/bongoCat/shared/InputDeviceScanner.qml)). So `python3` is another, undocumented runtime dependency.

### 2.2 What `--show-keycodes` turns off

libinput masks key identity by default. In upstream
[`src/util-libinput.c`](https://gitlab.freedesktop.org/libinput/libinput/-/blob/main/src/util-libinput.c) (lines ~324-330 on `main`), every key with `KEY_ESC <= code < KEY_ZENKAKUHANKAKU` (1–84) is printed as `*** (-1)` unless `show_keycodes` is set.

The [man page](https://gitlab.freedesktop.org/libinput/libinput/-/blob/main/tools/libinput-debug-events.man) explains why: keys are "partially obfuscated to avoid passwords and other sensitive information showing up in the output."

The plugin passes `--show-keycodes`, so it explicitly turns that protection off.

The masked range covers letters, digits, punctuation, Space (57), Enter (28), Ctrl, Shift, Alt, CapsLock, F1–F10 and the keypad. Our own design can't just leave the masking on, because then Space would look exactly like a letter.

### 2.3 What reaches QML

The full raw line goes from the child's stdout into a QML `SplitParser` (`IEL:95-98`). There:

- **Every raw line is re-emitted** as the signal `rawLineReceived(string line)` (`IEL:25, 98`).
- For key lines, a regex extracts `KEY_[A-Z0-9_]+` and the numeric code. The plugin then emits `keyPressed`, `keyReleased` or `keyRepeated` with the arguments `(string keyName, int keyCode)` (`IEL:21-23, 100-137`).

So the exact key identity (`"KEY_P"`, `25`) reaches the JS handlers of every widget instance (`BW:573-596`, `BDW:328-360`). Any QML in the shell process could listen to the same signals.

The handlers use the name for only two things:

1. Deciding whether the key is a "big hit": `KEY_SPACE`, `KEY_ENTER` or `KEY_KPENTER` (`BW:580-590`).
2. `recordKeystroke(keyName)`, which sorts the key into `char`, `correction` or `ignore` for the WPM metrics (`BW:262-284`).

In effect, the plugin turns keys into categories in JS, after the full key has already arrived.

### 2.4 Logging and storage

- I found **no logging of key data**. The plugin's own `console.log` / `console.warn` calls cover only device-path changes (`BW:57`), right-click sleep toggles (`BW:614`), settings and font failures, and scanner errors.
- The child processes' `stderr` goes into a `StdioCollector` that nothing reads (`IEL:158`).
- **Nothing key-related is persisted.** `savePluginData` is called only for settings (`BW:318-325`).
- WPM metrics keep plain in-memory arrays of timestamps (`_charStamps` and `_correctionStamps`, with no key names), trimmed to a sliding window (`BW:234-305`; desktop copy at `BDW:61-125`). The code comment says so: "no key contents are ever stored or logged".

### 2.5 Gaps against our privacy rule

1. The full key reaches QML. Our rule requires that only a category ever leaves the lowest input layer, and here that layer outputs everything.
2. `--show-keycodes` disables the one built-in safeguard libinput offers.
3. `rawLineReceived` sends unfiltered raw lines to every listener.
4. Each widget instance starts its **own** reader process: one per bar instance (one per monitor) and one per desktop widget. More copies of a full keylogger stream end up in the shell process.
5. The `input` group membership it needs lets *any* process run by the user read every keyboard. That cost belongs to the evtest/libinput approach as a whole, not to this plugin alone.

## 3. Animation and key mapping

The cat has six poses: idle, left paw down, right paw down, both paws down, blink, sleep. They map to state numbers 0–3 plus blink and sleep flags (`BW:402-421` sprite/glyph selection, `BW:423-475` press/release/repeat handlers).

| Input | Result |
|---|---|
| Any key except the "big" ones, press | Paws alternate: left, right, left, … (`leftWasLast` toggles) |
| Second key pressed while a paw is still down | Both paws down (state 3) |
| Release while both are down | Back to the last single paw |
| `KEY_SPACE`, `KEY_ENTER`, `KEY_KPENTER` press, if `doubleSlam` is on (default **true**) | **Both paws slam** (state 3); release returns to idle |
| Auto-repeat (key held) | Keeps the current pose |
| `pawHoldTime` > 0 | Paws stay down that many ms after release (0–100 ms, default 0) |
| No input for `waitingTimeout` (default 5 s) | Sleep pose |
| Idle and awake | Random blink every 6–14 s, lasting 300 ms |
| Right-click on the cat | Force sleep on/off |
| Mouse (off by default) | Left button → left paw, right button → right paw, both or middle → both paws, scroll → alternating paws |

Modifiers, arrows, F-keys and so on animate exactly like letters. The categories exist only for WPM counting, not for animation. A per-key option is also missing: Enter is tied to Space in the same toggle.

Keyboard, mouse buttons and scroll each keep their own state, merged in `resolveCatState()` (`BW:91-100`).

## 4. Placement

- **Bar widget:** a horizontal pill, with dock support since DMS 2.0 (`BW:20-22, 599-692`). Left-click opens a large popout that has its own settings and metrics tiles.
- **Desktop widget:** `DesktopPluginComponent`, minimum 100×80 (`BDW:11-18`). Optional background opacity (default 0) and a WPM line underneath.
- The code has no overlay, layer or click-through logic. It doesn't use `PanelWindow` or `WlrLayer` (I grepped for them). Always-on-top and click-through therefore come only from DMS's per-instance "Show on overlay" and "Click through" toggles, which the user sets (see [#2](https://github.com/PascalBrus/dms-bongocat/issues/2)).
- Multi-monitor: each bar or desktop instance is independent. Sound goes through a singleton (`services/BongoSoundService.qml`) so that it plays once rather than once per monitor. The input readers are **not** shared that way (see 2.5.4).

## 5. Art and licence

- **Code:** MIT, "Copyright (c) 2024 Loc Huynh" ([`LICENSE`](https://github.com/hthienloc/dms-plugins/blob/b7ec1d6cd0fa1d42cab7c2f5996305c34897275c/bongoCat/LICENSE)).
- **Classic skin:** a custom font, `assets/bongocat-Regular.otf`, whose glyph pairs (`"bc"`, `"dc"`, `"ba"`, `"da"`, blink `"gh"`, sleep `"ef"`) draw the poses (`BW:416-421`). That makes it vector art and lets it be recoloured: classic, theme primary or a custom hex.
  - The font's metadata has no copyright or licence string, and the repo has no licence file for it.
  - The README credits "[@StrayRogue](https://x.com/StrayRogue) — creator of the original Bongo Cat artwork". The original meme art has no stated open licence.
- **Pixel Doodle skin:** 32×25 PNGs (`idle`, `left`, `right`, `both`, `blink`, `sleep`) plus nearest-neighbour `@4x` copies (`assets/skins/pixel/`). It is credited to [StrawberryMatchaArt](https://www.etsy.com/shop/StrawberryMatchaArt), a commercial Etsy shop, again with no licence.
  - Seven recolours (black, orange, calico, siamese, pink, cyberpunk, zombie) are generated by `scripts/generate_pixel_skins.py` through palette swaps.
  - [`assets/skins/README.md`](https://github.com/hthienloc/dms-plugins/blob/b7ec1d6cd0fa1d42cab7c2f5996305c34897275c/bongoCat/assets/skins/README.md) documents the pixel layout in detail and would help anyone drawing their own skin.
- **Sounds:** WAVs generated by `scripts/generate_sounds.py`, so presumably original.

Conclusion: if we reuse the art privately, licensing doesn't matter (map note). Anything we publish needs our own art or explicit permission.

## 6. Settings

The settings page keys are defined in `BS`, and the desktop widget reads the same keys through `getData` (`BDW:20-43`):

| Key | Meaning / default |
|---|---|
| `catSkin` | `classic` (vector font) or `pixel` |
| `pixelSkinColor` | one of the 8 pixel colourways, default `white` |
| `catColorMode`, `catCustomColor` | `classic`, `primary` or `custom` hex (classic skin only) |
| `classicIdle` | revert to classic black/white colour when idle or asleep (default off) |
| `catSizePercent` | 50–200 %, default 100 |
| `catYOffset` | vertical nudge in the bar |
| `enableBlinking` | default on |
| `waitingTimeout` | seconds until sleep, default 5 |
| `pawHoldTime` | 0–100 ms, default 0 |
| `doubleSlam` | Space/Enter slam both paws, default on |
| `mouseEnabled` | default off |
| `showMetrics`, `metricsInBar`, `metricsWindowSec` | WPM/accuracy display (window: bar default 60 s, desktop default 15 s) |
| `soundEnabled`, `soundProfile`, `soundVolume`, `soundOnMouse` | key sounds, off by default |
| `selectedDevicePath` | `all` or a specific `/dev/input/eventN` |
| `desktopBackgroundOpacity`, `desktopShowMetrics` | desktop widget only |
| `showHints`, `usageGuideExpanded`, `creditsExpanded` | UI only |

There are 21 translation files under `translations/`.

## 7. Code quality

**Good:**

- Comments are clear.
- Keyboard, mouse and scroll states are kept separate.
- A sound singleton prevents double playback on multi-monitor setups.
- Missing tools show a badge instead of failing silently.
- Settings migrate from older versions (`migrateColorSetting`).
- `surfaceLive` gates the timers when the widget is hidden.

**Weak:**

- **Heavy duplication.** `BDW` copies most of `BW`'s logic (state machine, metrics, key lists), and the copies have already drifted:
  - `_nonCharKeys` differs: `BW` counts Enter as "ignore" and `BDW` doesn't.
  - Defaults differ: the WPM window is 60 s in `BW` and 15 s in `BDW`, and sound volume is 60 in `BW` and 70 in `BDW`.
- `BW` is a 1456-line file that mixes the popout UI with the logic.
- Parsing is substring and regex matching on human-readable debug-tool output, a format libinput doesn't promise to keep stable.
- `BW:60-64` has a leftover `inputRestartTimer` that refers to `inputProc`, which isn't in scope (it lives in `IEL`).
- There are no tests.

## 8. Maintenance and issues

- **Commits:** 30 commits touch `bongoCat/` in `dms-plugins`, from 2026-09-19 (when the plugin moved into the monorepo) to 2026-11-05 ("add Zombie and Cyberpunk skins"). Desktop widget support arrived on 2026-11-03. Everything is by one maintainer (`gh api repos/hthienloc/dms-plugins/commits?path=bongoCat`).
- **Repo:** `dms-plugins` is MIT-licensed, has 35 stars, and was last pushed 2026-11-05.
- **Earlier history:** the standalone repo `hthienloc/dms-bongo-cat` (created 2026-05-08, 17 stars) is **archived**; its last commit was 2026-07-29. The README's manual-install line still points to that archived repo.
- **Open issues:**
  - `dms-plugins` #39, "Use bongo cat pixel doodle as a mouse cursor?" (2026-11-05). This is the only open issue about the bongo cat.
  - In the archived repo, #14 "Turkish Translation" is still open.
  - Closed issues in the archived repo cover the desktop-widget request (#3), "widget does nothing when I type" (#4), skins (#8) and sounds (#10, #12).

## 9. Gaps against the map's destination

| Destination item | Existing plugin | Gap |
|---|---|---|
| Always-on-top, click-through desktop widget | Has a desktop widget; overlay and click-through come from DMS's per-instance toggles | None. We can rely on the same DMS toggles, so no extra layer code is needed |
| Keyboard only | Mouse is optional and off by default | None |
| Space = both arms slam | Yes (also Enter and keypad Enter), on by default | Small: Enter is coupled to Space; we may want Space alone |
| Key categories decide the animation | No. All non-big keys alternate paws the same way | We have to define our own category → animation table |
| Only categories leave the lowest input layer | **No.** `--show-keycodes`; full name and code reach QML; raw lines re-emitted | **Main gap:** we need our own reader that reduces keys to categories before anything reaches the shell |
| Nothing logged or stored | Met in practice: no key logging, metrics are in-memory timestamps | Our reader must keep that and have no metrics at all |
| One reader process | One per widget instance | Use one shared reader (a singleton or an external helper) |
| Art | Vector font plus pixel PNGs, licence unclear | Fine for private use; we'd need our own art to publish |
| Settings page | Rich (above) | Good template; we need fewer settings |
| Hyprland 0.57 test environment | n/a | Not covered here |

**Can we reuse it?** Possibly. The animation state machine, the pose set and the desktop-widget approach are worth copying or forking for private use; the code is MIT. The input layer should not be reused.
