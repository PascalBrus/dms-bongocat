# How can the Bongo Cat learn about keypresses system-wide on Hyprland?

Research for [PascalBrus/dms-bongocat#3](https://github.com/PascalBrus/dms-bongocat/issues/3).
Researched 2026-11-06.

**Question:** How can a Bongo Cat plugin for DankMaterialShell (dms-shell 2.x, built on Quickshell 0.3.x / QML), running on Hyprland 0.57, find out that a key was pressed, whichever window has focus?

**Constraint (hard privacy rule):** the cat may only learn a key's *category* (space, letter, digit, modifier, enter/backspace, other, …), never the key itself. Nothing is logged or stored. The collapse to a category must happen inside the component that reads the raw input, before anything leaves it.

## TL;DR

* A normal Wayland client **cannot** see keys meant for other windows. That is by design. No standard protocol offers passive, non-consuming, all-keys observation (see [§1](#1-wayland-security-model)).
* **New finding:** Hyprland's Lua config (0.57 is Lua-configured) has a **`input.keyboard.key` event**. It fires for *every* key press, release and repeat, before keybinds and before the focused client receive the key, and passes the XKB keycode to a Lua callback ([§2.1](#21-lua-event-inputkeyboardkey--the-best-fit)). From that callback, `hl.dispatch(hl.dsp.event("…"))` can emit a `custom>>…` line on Hyprland's event socket (socket2). Quickshell already exposes socket2 as `Hyprland.rawEvent`. This path needs **no extra privileges, no helper binary, no input group**. The keycode is collapsed to a category inside the compositor's own config code, so nothing below that layer ever sees the raw key.
* The classic approach is an **evdev helper** that reads `/dev/input/event*`, collapses each key to a category and prints category tokens. Quickshell reads them via `Process` + `SplitParser` or a `Socket`. It works on any compositor and in any nested setup, but it needs a permission model. The least bad options are a **setgid-`input` helper binary** or a **root system service with a user-restricted socket**. Putting the user in the `input` group gives every user process keylogger power. A uaccess ACL is equivalent to that for the active session. `wayland-bongocat` and DMS itself both use the plain input group.
* **DMS itself reads evdev.** Its Go backend opens every keyboard to drive the Caps Lock / Num Lock OSD, and `dms setup` adds the user to the `input` group ([§5](#5-dankmaterialshell-dms)). The real laptop may therefore already be in the `input` group. Check this.
* **Recommendation:** use the **Hyprland Lua event bridge** as the primary design. Keep a **setgid category-only evdev helper** as the documented fallback, for non-Hyprland setups or if the Lua bridge turns out unusable. Details in [§9](#9-recommendation).

## Source versions examined

| Project | Version / commit |
|---|---|
| Hyprland | tag `v0.57.0` (released 2026-10-28, [release](https://github.com/hyprwm/Hyprland/releases/tag/v0.57.0)); source browsed at `main` [`ab61418`](https://github.com/hyprwm/Hyprland/tree/ab6141897d57e609e24c571a762c30c963e0885a); key files re-checked at the tag where noted |
| hyprland-wiki | [`096316e`](https://github.com/hyprwm/hyprland-wiki/tree/096316e5e401f310b9a3357d445daa4dd813ff6a) (git wiki = "latest git" docs, see version note) |
| Quickshell | `master` [`9b0f339`](https://github.com/quickshell-mirror/quickshell/tree/9b0f339b19057f4731bf3b48293307d7ef241671) (latest tag v0.3.2) |
| DankMaterialShell | `master` [`fa1b1a2`](https://github.com/AvengeMedia/DankMaterialShell/tree/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e) (`quickshell/VERSION` = `v2.2-beta`) |
| dms-plugins | [`e06aa40`](https://github.com/AvengeMedia/dms-plugins/tree/e06aa40b1cb2b9ce50d3ee75f2874af05ae1763f) |
| wayland-bongocat | [`91a719b`](https://github.com/saatvik333/wayland-bongocat/tree/91a719bf6ae24afa8fb5e3d6b394731fbf8a671b) |
| showmethekey | [`cddb161`](https://github.com/AlynxZhou/showmethekey/tree/cddb16130732ddd439f4c468790fb5735160d8e2) |
| Dev machine | systemd 262, libinput 1.32.0, libevdev 1.14.0, python-evdev 2.0.0, wayland 1.26.0 (local `pacman -Q`) |

Abbreviations used in links below: `HL` = Hyprland at `ab61418`, `HLW` = hyprland-wiki at `096316e`, `QS` = Quickshell at `9b0f339`, `DMS` = DankMaterialShell at `fa1b1a2`.

---

## 1. Wayland security model

* `wl_keyboard` key events go to the keyboard's single **active surface**, the one that received `wl_keyboard.enter`. The interface defines its logical state as "an active surface (possibly null), the keys currently logically down, …" ([wayland.xml, `wl_keyboard`](https://gitlab.freedesktop.org/wayland/wayland/-/blob/main/protocol/wayland.xml), checked locally at `/usr/share/wayland/wayland.xml` lines 2573–2700). So a client sees keys only while it has keyboard focus. A Quickshell/DMS layer surface is no exception: `WlrLayershell.keyboardFocus` defaults to `None`, and `Exclusive` locks out all other windows ([QS `wlr_layershell.hpp` L45–L108](https://github.com/quickshell-mirror/quickshell/blob/9b0f339b19057f4731bf3b48293307d7ef241671/src/wayland/wlr_layershell/wlr_layershell.hpp#L45-L108)). Neither option gives passive observation.
* **input-method-unstable-v2** (`grab_keyboard`): "The compositor should send all keyboard events on the seat to the grab holder", it is "an exclusive grab", and "There must be no more than one input method object per seat" ([wayland.app: input-method-unstable-v2](https://wayland.app/protocols/input-method-unstable-v2)). In Hyprland, keys go to the IME *instead of* the focused client when an IME holds a grab ([HL `InputManager.cpp` L1704–L1752](https://github.com/hyprwm/Hyprland/blob/ab6141897d57e609e24c571a762c30c963e0885a/src/managers/input/InputManager.cpp#L1704-L1752)). Using it would mean consuming every key, re-injecting it via `virtual-keyboard`, and conflicting with fcitx5 and similar tools. It is a full keylogger by construction. **Not viable.**
* **virtual-keyboard** only *injects* keys and does not observe them. The same goes for **uinput** at the kernel level. Neither is relevant here except as the re-injection half of an interception design.
* **xdg-desktop-portal GlobalShortcuts:** apps `BindShortcuts()` with per-shortcut IDs and an optional `preferred_trigger`, and get `Activated`/`Deactivated` signals per `shortcut_id`. The page describes no wildcard or catch-all mechanism ([portal docs](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.GlobalShortcuts.html)). **Not viable** for "all keys".
* **hyprland-global-shortcuts-v1** (Quickshell `GlobalShortcut`): same idea, named `appid:name` shortcuts that the compositor triggers from a bind ([QS `global_shortcuts/qml.hpp` L13–L34](https://github.com/quickshell-mirror/quickshell/blob/9b0f339b19057f4731bf3b48293307d7ef241671/src/wayland/hyprland/global_shortcuts/qml.hpp#L13-L34)). It has no catch-all of its own. It could serve as a *transport* (see §2.1 alt), but it doesn't help with detection.
* **Input capture** (hyprland-input-capture-v1 / InputCapture portal / libei): Hyprland forwards keys to a capture session only while it is *active*, and while captured the key is **not** passed to clients (`passEvent = … && !PROTO::inputCapture->isCaptured()`) ([HL `InputCapture.cpp` L426–L429](https://github.com/hyprwm/Hyprland/blob/ab6141897d57e609e24c571a762c30c963e0885a/src/protocols/InputCapture.cpp#L426-L429), [HL `InputManager.cpp` L1716–L1722](https://github.com/hyprwm/Hyprland/blob/ab6141897d57e609e24c571a762c30c963e0885a/src/managers/input/InputManager.cpp#L1716-L1722)). It is also behind the `input-capture` permission ("capture all input events", default ASK) ([HLW permissions](https://github.com/hyprwm/hyprland-wiki/blob/096316e5e401f310b9a3357d445daa4dd813ff6a/content/configuring/core/advanced-configuration/permissions.md)). It consumes input, so it is **not viable**.
* **HyprlandFocusGrab** only keeps focus on the listed windows ([QS `focus_grab/qml.hpp` L20–L30](https://github.com/quickshell-mirror/quickshell/blob/9b0f339b19057f4731bf3b48293307d7ef241671/src/wayland/hyprland/focus_grab/qml.hpp#L20-L30)). Not relevant.

**Conclusion:** at the protocol level, only the compositor (or a process with raw `/dev/input` access) can see all keys. The next two sections cover those two places.

## 2. Hyprland compositor-side options

> Version note: the wiki is versioned and `wiki.hypr.land` shows *latest git* by default ([HLW configuring/_index.md](https://github.com/hyprwm/hyprland-wiki/blob/096316e5e401f310b9a3357d445daa4dd813ff6a/content/configuring/_index.md)). Hyprland 0.57 uses a **Lua** config (`hyprland.lua`). The FAQ says "Hyprland will continue using Lua" ([HLW faq L34–L37](https://github.com/hyprwm/hyprland-wiki/blob/096316e5e401f310b9a3357d445daa4dd813ff6a/content/faq/_index.md)), `src/config/` contains only the Lua config manager, and the default config embedded in the binary is `example/hyprland.lua` ([HL `DefaultConfig.hpp` L15](https://github.com/hyprwm/Hyprland/blob/ab6141897d57e609e24c571a762c30c963e0885a/src/config/lua/DefaultConfig.hpp#L15)). The hyprlang `bindn`/`bindt`/`bindr` letter flags are now Lua bind flags (`non_consuming`, `transparent`, `release`, …). *Unverified:* whether 0.57 still accepts a legacy `hyprland.conf`. Quickshell has a `Hyprland.usingLua` property ([QS `ipc/qml.hpp` L77](https://github.com/quickshell-mirror/quickshell/blob/9b0f339b19057f4731bf3b48293307d7ef241671/src/wayland/hyprland/ipc/qml.hpp#L77)), which suggests both config kinds exist across versions.

### 2.1 Lua event `input.keyboard.key`: the best fit

* **Documented:** "`input.keyboard.key` — Emitted when a key is pressed or released — integer: XKB keycode; integer: Unix timestamp …; integer: released (0), pressed (1), or repeated (2)" ([HLW events.md L38](https://github.com/hyprwm/hyprland-wiki/blob/096316e5e401f310b9a3357d445daa4dd813ff6a/content/configuring/core/advanced-configuration/events.md#L38)). Callbacks are registered with `hl.on(name, fn)` (same page, L8–L15).
* **Present in v0.57.0:** the listener exists at the tag ([HL v0.57.0 `LuaEventHandler.cpp` L174–L180](https://github.com/hyprwm/Hyprland/blob/v0.57.0/src/config/lua/LuaEventHandler.cpp#L174-L180)). It pushes `keyEvent.keycode + 8` ("Because to xkbcommon it's +8 from libinput"), `timeMs` and `state`, and the event name is registered in the known-events list (L288).
* **Where it fires:** `CInputManager::onKeyboardKey` emits the bus event *first*, before IME, input-capture, keybind handling and delivery to the focused surface ([HL `InputManager.cpp` L1701–L1714](https://github.com/hyprwm/Hyprland/blob/ab6141897d57e609e24c571a762c30c963e0885a/src/managers/input/InputManager.cpp#L1701-L1714)). The Lua listener ignores the cancel flag (`const SCallbackInfo& _`), so it cannot swallow keys. This makes it **inherently non-consuming** and it **sees keys consumed by binds**. It skips disabled or not-allowed keyboards (`!pKeyboard->m_enabled || !pKeyboard->m_allowed`, L1702).
* **Getting the category out of the compositor:**
  * `hl.dsp.event(string)` is the `event` dispatcher, documented as "Send an event to socket2" ([HLW dispatchers.md L59](https://github.com/hyprwm/hyprland-wiki/blob/096316e5e401f310b9a3357d445daa4dd813ff6a/content/configuring/core/dispatchers.md); Lua binding [HL `LuaBindingsDispatchers.cpp` L352–L360, L1402](https://github.com/hyprwm/Hyprland/blob/ab6141897d57e609e24c571a762c30c963e0885a/src/config/lua/bindings/LuaBindingsDispatchers.cpp#L352-L360)). It posts `custom>>DATA` on socket2 ([HL `ConfigActions.cpp` L1326–L1329](https://github.com/hyprwm/Hyprland/blob/ab6141897d57e609e24c571a762c30c963e0885a/src/config/shared/actions/ConfigActions.cpp#L1326-L1329)).
  * Calling `hl.dispatch(...)` from inside an `hl.on` callback is the documented pattern ([HLW code-snippets.md L81–L87](https://github.com/hyprwm/hyprland-wiki/blob/096316e5e401f310b9a3357d445daa4dd813ff6a/content/configuring/code-snippets.md)).
  * Quickshell's `Hyprland.rawEvent` is "Emitted for every event that comes in through the hyprland event socket (socket2)", and the event object exposes `name` and raw `data` ([QS `ipc/qml.hpp` L85–L89](https://github.com/quickshell-mirror/quickshell/blob/9b0f339b19057f4731bf3b48293307d7ef241671/src/wayland/hyprland/ipc/qml.hpp#L85-L89), [QS `ipc/connection.hpp` L42–L47](https://github.com/quickshell-mirror/quickshell/blob/9b0f339b19057f4731bf3b48293307d7ef241671/src/wayland/hyprland/ipc/connection.hpp#L42-L47)). DMS already consumes `Hyprland.rawEvent` this way, for example in `KeyboardLayoutService` ([DMS `KeyboardLayoutService.qml` L95–L101](https://github.com/AvengeMedia/DankMaterialShell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/Services/KeyboardLayoutService.qml#L95-L101)), and DMS plugins may `import Quickshell.Hyprland` ([dms-plugins `DankHyprlandWindows.qml` L3](https://github.com/AvengeMedia/dms-plugins/blob/e06aa40b1cb2b9ce50d3ee75f2874af05ae1763f/DankHyprlandWindows/DankHyprlandWindows.qml#L3)).
  * **Alternative transport:** `hl.dsp.global("bongocat:space")` → Quickshell `GlobalShortcut { appid: "bongocat"; name: "space" }`. `Actions::global` only fires if a client has registered that `appid:name` ([HL `ConfigActions.cpp` L1683–L1697](https://github.com/hyprwm/Hyprland/blob/ab6141897d57e609e24c571a762c30c963e0885a/src/config/shared/actions/ConfigActions.cpp#L1683-L1697)). The advantage is that only the registered client receives it, whereas every socket2 reader sees socket2 events. The cost is one `GlobalShortcut` per category, and the pressed/released semantics come from the bind state (`m_passPressed`), which is *unverified* for calls made outside a bind.
* **Distribution:** DMS's generated `hyprland.lua` already composes itself from modules via `require("dms.colors")`, `require("dms.binds")`, … ([DMS embedded `hyprland.lua` L84–L91](https://github.com/AvengeMedia/DankMaterialShell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/core/internal/config/embedded/hyprland.lua#L84-L91)). Hyprland's `package.path` includes `<configDir>/?.lua` ([HL `lua/ConfigManager.cpp` L509](https://github.com/hyprwm/Hyprland/blob/ab6141897d57e609e24c571a762c30c963e0885a/src/config/lua/ConfigManager.cpp#L509)). The plugin can therefore ship a `bongocat.lua` that the user loads with one `require("bongocat")` line.
* **Sketch (illustrative, untested):**
  ```lua
  -- ~/.config/hypr/bongocat.lua — the ONLY place that sees keycodes.
  local function category(xkb)
    local c = xkb - 8                            -- back to linux/input-event-codes.h
    if c == 57 then return "space" end
    if c == 28 or c == 96 or c == 14 then return "enter" end
    -- … letters/digits/modifiers by evdev code table …
    return "other"
  end
  hl.on("input.keyboard.key", function(xkb, _t, state)
    if state ~= 1 then return end                -- press only; drop release/repeat
    hl.dispatch(hl.dsp.event("bongocat," .. category(xkb)))
  end)
  ```
  QML side: `Connections { target: Hyprland; function onRawEvent(e) { if (e.name === "custom" && e.data.startsWith("bongocat,")) … } }`.
* **Privacy enforcement point:** the compositor already holds the libinput fds and sees every key, so it is the lowest layer that reads input. The Lua callback is the first code we control, and it collapses the keycode to a category before `hl.dsp.event`. Nothing that leaves the compositor (socket2) contains a key identity. Caveats:
  1. socket2 is readable by every process of the user. Category tokens plus their **timing** are visible there. That is far less than a key identity, but keystroke timing is not nothing (see the lock-screen open question).
  2. The Lua state has full stdlib including `io`/`os` (`luaL_openlibs`, [HL `lua/ConfigManager.cpp` L494](https://github.com/hyprwm/Hyprland/blob/ab6141897d57e609e24c571a762c30c963e0885a/src/config/lua/ConfigManager.cpp#L494)). Other config code could also subscribe to the event, but that is the user's own config, which is as trusted as the compositor.
* **Cost / risk:** the callback runs synchronously on the compositor's input path, so a slow callback would add input lag to the whole desktop. Keep it to a table lookup and one dispatch. The Lua config API is young: the 0.57 release notes are full of `config/lua:` changes and renames ([v0.57.0 release](https://github.com/hyprwm/Hyprland/releases/tag/v0.57.0)), so the event name or signature could change between releases. If it breaks, it fails visibly (the cat stops moving), not unsafely.
* **Latency:** in-process callback → socket2 write → Quickshell socket read. Expected well under the 16 ms frame budget. *Unverified, measure it.*
* **Nested Hyprland (inside KDE):** the nested Hyprland only receives keys from the KDE host while its window has focus, so the cat reacts only to typing inside the nested session. That is fine for testing. *Unverified:* that the nested (Wayland-backend) keyboard goes through the same `onKeyboardKey` path. It very likely does, since the Lua snippet for sharing keybinds with nested sessions relies on the host's keyboard routing ([HLW code-snippets.md L71–L93](https://github.com/hyprwm/hyprland-wiki/blob/096316e5e401f310b9a3357d445daa4dd813ff6a/content/configuring/code-snippets.md)). On a **real session** it sees all keyboards Hyprland manages.

### 2.2 Binds (`non_consuming`, `transparent`, `release`, `catchall`)

* Bind flags in 0.57 Lua syntax ([HLW flags.md](https://github.com/hyprwm/hyprland-wiki/blob/096316e5e401f310b9a3357d445daa4dd813ff6a/content/configuring/core/binds/flags.md)): `non_consuming` ("Key/mouse events will be passed to the active window in addition to triggering the dispatcher", the old `bindn`), `transparent` ("Cannot be shadowed by other binds", the old `bindt`), `release` (the old `bindr`), plus `ignore_mods`, `submap_universal`, `repeating`, and others. There is no `bindp`-style "passive" flag beyond these.
* `catchall` "activates no matter which key is pressed" ([HLW submaps.md L117–L124](https://github.com/hyprwm/hyprland-wiki/blob/096316e5e401f310b9a3357d445daa4dd813ff6a/content/configuring/core/binds/submaps.md#L117-L124)). However:
  * it is **only allowed inside submaps** (`"hl.bind: catchall keybinds are only allowed in submaps."`, [HL `LuaBindingsToplevel.cpp` L82–L83](https://github.com/hyprwm/Hyprland/blob/ab6141897d57e609e24c571a762c30c963e0885a/src/config/lua/bindings/LuaBindingsToplevel.cpp#L82-L83));
  * it fires **only when no other bind matched** (`if (candidates.empty() && hits.empty())`, [HL `keybinds/Manager.cpp` L504–L508, L565–L568](https://github.com/hyprwm/Hyprland/blob/ab6141897d57e609e24c571a762c30c963e0885a/src/keybinds/Manager.cpp#L504-L568));
  * the dispatcher doesn't receive *which* key matched.
  * `non_consuming` catchall does pass keys through. The wiki's nested-session snippet uses `hl.bind("catchall", hl.dsp.no_op(), { non_consuming = true })` ([HLW code-snippets.md L71–L93](https://github.com/hyprwm/hyprland-wiki/blob/096316e5e401f310b9a3357d445daa4dd813ff6a/content/configuring/code-snippets.md)).
* Using it would mean living permanently in a custom submap with every normal bind marked `submap_universal`, and the cat would still only learn "some key that isn't a bind", not its category. A per-key bind list (`non_consuming` + `ignore_mods` for every key of the keyboard, each dispatching its category) is possible in principle but enormous and fragile. Binds on modifiers combined with `ignore_mods` behave oddly. **Superseded by §2.1.** `exec`-per-key (`hl.dsp.exec_cmd`) would also fork a process per keypress, which is far too heavy.

### 2.3 Hyprland IPC socket2 (without config changes)

The socket2 event list contains **no key events**. The keyboard-related ones are `activelayout` (`KEYBOARDNAME,LAYOUTNAME`) and `submap` ([HLW ipc/_index.md L58–L66](https://github.com/hyprwm/hyprland-wiki/blob/096316e5e401f310b9a3357d445daa4dd813ff6a/content/ipc/_index.md)). Grepping the source for `postEvent` event names confirms this. The full set is window, workspace, monitor, layer, group, `submap`, `activelayout`, `screencast`, `bell`, `custom` and similar. socket2 is useful only as the *transport* for §2.1's `custom` events.

### 2.4 Hyprland C++ plugin

A plugin could subscribe to the same internal `Event::bus()->m_events.input.keyboard.key` signal that the Lua handler uses. The old `HyprlandAPI::registerCallbackDynamic(…"keyPress"…)` is now marked `[[deprecated]]` ([HL `PluginAPI.hpp` L177–L184](https://github.com/hyprwm/Hyprland/blob/ab6141897d57e609e24c571a762c30c963e0885a/src/plugins/PluginAPI.hpp#L177-L184)). Costs:

* The plugin must be built against the exact Hyprland headers. The official template throws on `COMPOSITOR_HASH != CLIENT_HASH` ("prevent random crashes coming from mismatched header versions"), and API stability is only "guaranteed … as long as the version matches" ([HLW development/getting-started.md L74–L90](https://github.com/hyprwm/hyprland-wiki/blob/096316e5e401f310b9a3357d445daa4dd813ff6a/content/hyprland-plugins/development/getting-started.md), [plugin-guidelines.md L75](https://github.com/hyprwm/hyprland-wiki/blob/096316e5e401f310b9a3357d445daa4dd813ff6a/content/hyprland-plugins/development/plugin-guidelines.md)). In practice that means a rebuild via `hyprpm` on every Hyprland update ([HLW using-plugins.md](https://github.com/hyprwm/hyprland-wiki/blob/096316e5e401f310b9a3357d445daa4dd813ff6a/content/hyprland-plugins/using-plugins.md)).
* Plugins are subject to the `plugin` permission (default ASK) ([HLW permissions.md](https://github.com/hyprwm/hyprland-wiki/blob/096316e5e401f310b9a3357d445daa4dd813ff6a/content/configuring/core/advanced-configuration/permissions.md)).
* It runs native code inside the compositor, where a crash takes the session down.

It gives nothing that the Lua event doesn't, at much higher cost. **Not recommended.**

## 3. evdev via `/dev/input`

### 3.1 Facts on the dev machine (local checks, 2026-11-06)

* `id`: user `widu` is in `sys, network, wireshark, …, video, storage, lp, audio, wheel` and **not** in `input`.
* `ls -l /dev/input`: `crw-rw---- root input` for `event*`. This comes from systemd's `SUBSYSTEM=="input", GROUP="input"` (`/usr/lib/udev/rules.d/50-udev-default.rules` L51).
* `/usr/lib/udev/rules.d/70-uaccess.rules` tags only **joysticks** among input devices (`SUBSYSTEM=="input", ENV{ID_INPUT_JOYSTICK}=="?*", TAG+="uaccess"`, L61). Keyboards get **no** logind ACL by default. Upstream file: [systemd `src/login/70-uaccess.rules.in`](https://github.com/systemd/systemd/blob/main/src/login/70-uaccess.rules.in).

### 3.2 Permission models for a reader

| Model | How | Who gains raw-key power | Notes |
|---|---|---|---|
| User in `input` group | `usermod -aG input $USER` | **Every process of the user** (browser, npm postinstall, …) can read every keystroke of every keyboard. | What wayland-bongocat and DMS ask for. Weakest. A user-level systemd service has the same rights as the user, so it is not a solution on its own. |
| uaccess / logind ACL | custom udev rule `SUBSYSTEM=="input", ENV{ID_INPUT_KEYBOARD}=="1", TAG+="uaccess"` | Every process of the **active-seat** user (ACL dropped on VT switch / logout). | Practically the same exposure as the input group for a single-user laptop. Not default, for good reason (§3.1). |
| **setgid `input` helper binary** | `root:input`, mode `2755`, small C/Rust program | Only code running *inside the helper*. Callers only get its stdout (categories). | Kernel marks setgid processes non-dumpable, so the user can't ptrace it; the dynamic loader ignores `LD_PRELOAD`/`LD_LIBRARY_PATH` in secure-exec mode. The helper must take no options that change output granularity, must not exec anything, must open devices, then (optionally) drop the gid with `setgid(getgid())` while keeping fds, like wshowkeys' "permissions are dropped after startup". *Kernel/ld.so specifics unverified here, see `man 8 ld.so` "secure-execution mode" and `man 2 prctl` PR_SET_DUMPABLE.* |
| Root system service + socket | systemd **system** unit as root (or `SupplementaryGroups=input` + `DynamicUser=`) writing category tokens to a socket owned by the user (e.g. `SocketUser=`/`SocketMode=0600`) | Only the daemon. | Precedent: keyd runs as a system service (`sudo systemctl enable --now keyd`) and gates client access with a `keyd` group ([keyd README](https://github.com/rvaiya/keyd/blob/master/README.md)). More moving parts (system unit, socket unit). Hard to tie to "the current graphical user" on multi-user machines. Quickshell can connect with `Socket` ([QS `io/socket.hpp`](https://github.com/quickshell-mirror/quickshell/blob/9b0f339b19057f4731bf3b48293307d7ef241671/src/io/socket.hpp)). |
| pkexec per launch | polkit action, `auth_admin_keep` | Only the helper (runs as root). | Precedent: showmethekey runs `showmethekey-cli` via pkexec with a polkit policy, and an optional rule lets `wheel` members skip the password ([policy](https://github.com/AlynxZhou/showmethekey/blob/cddb16130732ddd439f4c468790fb5735160d8e2/dists/one.alynx.showmethekey.policy.in), [rules](https://github.com/AlynxZhou/showmethekey/blob/cddb16130732ddd439f4c468790fb5735160d8e2/dists/one.alynx.showmethekey.rules)). Fine for an on-demand tool, but bad for something started at every login. The rule makes it password-less for every wheel process, which is a weaker form of the input-group problem limited to that binary. |
| setuid root | `chmod a+s` | Only the binary, until it drops privileges. | Precedent: wshowkeys "must be configured as setuid … requires root permissions to read input events. These permissions are dropped after startup." ([wshowkeys README](https://git.sr.ht/~sircmpwn/wshowkeys/blob/master/README.md)). Root is more than needed, so setgid `input` is strictly smaller. |
| File capabilities | `setcap cap_dac_read_search+ep` | Binary can read **any file** on the system. | Broader than gid `input`. Not recommended. |

### 3.3 The helper → Quickshell path

* Quickshell `Process` has a `stdout` parser property, and `SplitParser` is "DataStreamParser for delimited data streams" ([QS `io/process.hpp` L23–L128](https://github.com/quickshell-mirror/quickshell/blob/9b0f339b19057f4731bf3b48293307d7ef241671/src/io/process.hpp#L23-L128), [QS `io/datastream.hpp` L65–L67](https://github.com/quickshell-mirror/quickshell/blob/9b0f339b19057f4731bf3b48293307d7ef241671/src/io/datastream.hpp#L65-L67)). For a daemon there are `Socket` and `SocketServer` ([QS `io/socket.hpp` L19–L95](https://github.com/quickshell-mirror/quickshell/blob/9b0f339b19057f4731bf3b48293307d7ef241671/src/io/socket.hpp)).
* DMS plugin manifests declare a `process` permission, which is "not currently enforced" ([DMS plugin-manifest-reference.md L43–L50](https://github.com/AvengeMedia/DankMaterialShell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/.agents/skills/dms-plugin-dev/references/plugin-manifest-reference.md)). Daemon plugins are invisible `PluginComponent`s ([DMS daemon-plugin-guide.md](https://github.com/AvengeMedia/DankMaterialShell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/.agents/skills/dms-plugin-dev/references/daemon-plugin-guide.md)), and `DankHooks` uses `Quickshell.Io` ([dms-plugins `DankHooks.qml` L2](https://github.com/AvengeMedia/dms-plugins/blob/e06aa40b1cb2b9ce50d3ee75f2874af05ae1763f/DankHooks/DankHooks.qml#L2)). Spawning a helper via `Process` is therefore within what plugins do.
* **Privacy enforcement point:** inside the helper, `read()` → `input_event` → category → write token. The raw code never crosses the process boundary. wayland-bongocat does exactly this split (see §6).
* **Device detection:** enumerate `/dev/input/event*`, keep devices whose `EVIOCGBIT(EV_KEY)` has `KEY_A`, `KEY_Z`, `KEY_ENTER`, `KEY_SPACE`, and watch `/dev/input` with inotify for hotplug (wayland-bongocat, §6). DMS does the same with a name heuristic plus a key-count check (§5).
* **Latency:** kernel event → poll → write. Sub-millisecond in principle. *Unverified, measure it.*
* **Nested vs real:** evdev is below every compositor, so in the nested test the helper sees **all typing on the dev machine** (including KDE windows), not just the nested session. On the real laptop it sees all keyboards. It needs the same permission setup on both machines. On the dev machine that is a setup step, because the user is not in `input`.
* It does not observe virtual keyboards from other compositors' IMEs or `wtype`-style injection that goes only through Wayland. *Unverified edge case, irrelevant for a cat.*

## 4. libinput

`libinput debug-events` needs the same `/dev/input` access as evdev. As a privacy precedent, its man page says "Key events shown by this tool are partially obfuscated to avoid passwords and other sensitive information showing up in the output. Use the `--show-keycodes` argument to make all keycodes visible." ([libinput-debug-events.man L41–L46](https://gitlab.freedesktop.org/libinput/libinput/-/blob/main/tools/libinput-debug-events.man)). The exact obfuscation rule (which keys stay visible) lives in libinput's shared print code. *Not inspected; unverified which keys are shown.* Using libinput in the helper only buys device-quirk handling, which a cat doesn't need, so plain evdev/libevdev is simpler. The dev machine doesn't have the `libinput` CLI tools installed (only the library, 1.32.0).

## 5. DankMaterialShell (DMS)

* **DMS reads evdev in its Go backend.** `core/internal/server/evdev/manager.go` globs `/dev/input/event*`, opens each with `O_RDONLY`, and keeps keyboards by name heuristic ("keyboard", "kbd", "input"+"key") or by EV_KEY state length > 100 ([DMS `manager.go` L130–L193](https://github.com/AvengeMedia/DankMaterialShell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/core/internal/server/evdev/manager.go#L130-L193)). It watches `/dev/input` for hotplug (L60). It reads **every** event and reacts only to `KEY_CAPSLOCK`/`KEY_NUMLOCK` and the LEDs (L274–L313). Its published state is only `{available, capsLock, numLock}` ([DMS `models.go`](https://github.com/AvengeMedia/DankMaterialShell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/core/internal/server/evdev/models.go)). That is the same "collapse at the reader" pattern this project wants, for lock state only.
* Without access it refuses with the message "Add your user to the 'input' group … or run `dms setup`" ([DMS `manager.go` L440–L446](https://github.com/AvengeMedia/DankMaterialShell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/core/internal/server/evdev/manager.go#L440-L446)), and **`dms setup` runs `usermod -aG input` via privesc** "for Caps Lock OSD support" ([DMS `commands_setup.go` L359–L362, L468–L493](https://github.com/AvengeMedia/DankMaterialShell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/core/cmd/dms/commands_setup.go#L468-L493)).
* The QML side gets the state over the DMS socket subscription `"evdev"` and exposes `DMSService.capsLockState` ([DMS `DMSService.qml` L81–L87, L321–L329](https://github.com/AvengeMedia/DankMaterialShell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/Services/DMSService.qml#L321-L329)), which `CapsLockOSD.qml` consumes ([DMS `CapsLockOSD.qml`](https://github.com/AvengeMedia/DankMaterialShell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/Modules/OSD/CapsLockOSD.qml)).
* **There is no generic "key pressed" event from DMS** that a plugin could subscribe to. Plugins use the same Quickshell modules as the shell (`Quickshell.Io`, `Quickshell.Hyprland`).
* **Consequence:** if the laptop user ran `dms setup`, they are **already in the `input` group**. Every user process can already read raw keys, and an evdev helper would need no extra setup there. That makes the evdev option *convenient* on the laptop but doesn't make it better for privacy. It also means the bongocat project cannot make the laptop more private than DMS already made it, short of the user leaving the group and losing the Caps Lock OSD.

## 6. Existing Linux Bongo Cat: saatvik333/wayland-bongocat

* Reads evdev and requires `sudo usermod -a -G input $USER` ([README L31–L35](https://github.com/saatvik333/wayland-bongocat/blob/91a719bf6ae24afa8fb5e3d6b394731fbf8a671b/README.md#L31-L35)).
* **Architecture precedent:** a separate helper process (spawned with a `socketpair(AF_UNIX, SOCK_SEQPACKET…)`) reads devices and sends only a **paw bitmask** (left/right) to the renderer ([`input.c` L338–L368, L487–L492](https://github.com/saatvik333/wayland-bongocat/blob/91a719bf6ae24afa8fb5e3d6b394731fbf8a671b/src/platform/input.c#L338-L368)). The mapping is `paw_for_keycode()` ([`paw_frame.h` L15–L26](https://github.com/saatvik333/wayland-bongocat/blob/91a719bf6ae24afa8fb5e3d6b394731fbf8a671b/include/graphics/paw_frame.h#L15-L26)). It counts presses only (`EV_KEY && value == 1`).
* **Device detection:** a device is a keyboard if `EVIOCGBIT(EV_KEY)` contains `KEY_A, KEY_Z, KEY_ENTER, KEY_SPACE` ([`input.c` L93–L105](https://github.com/saatvik333/wayland-bongocat/blob/91a719bf6ae24afa8fb5e3d6b394731fbf8a671b/src/platform/input.c#L93-L105)). Optional selection is by `/dev/input/by-id` path or name. inotify watches `/dev/input`, `by-id` and `by-path` for hotplug (L316–L322).
* Other bongo cat projects were not surveyed beyond this one. *Unverified whether any uses a compositor API.*

## 7. Other precedent daemons

* **keyd:** system-wide daemon enabled with `sudo systemctl enable --now keyd`, uses "kernel level input primitives (evdev, uinput)", and has a `keyd` group for client features ([README](https://github.com/rvaiya/keyd/blob/master/README.md)).
* **kanata:** user joins `input` and `uinput` groups, a udev rule sets `/dev/uinput` to `GROUP="uinput"`, and it optionally runs as a systemd *user* service, which only works because of the group membership ([docs/setup-linux.md](https://github.com/jtroo/kanata/blob/main/docs/setup-linux.md)).
* **showmethekey:** pkexec'd `showmethekey-cli` (libinput → JSON with `key_name` on stdout) + GTK UI ([`showmethekey-cli/main.c` L72–L100](https://github.com/AlynxZhou/showmethekey/blob/cddb16130732ddd439f4c468790fb5735160d8e2/showmethekey-cli/main.c#L72-L100)). The helper/UI split is the same as ours, but its output is the full key name.
* **wshowkeys:** setuid root, drops privileges after opening devices ([README](https://git.sr.ht/~sircmpwn/wshowkeys/blob/master/README.md)).
* **interception-tools:** runs `udevmon` as root and grabs devices. *Not checked in this research, mentioned for completeness only.*

## 8. Comparison

| # | Approach | Setup cost | Extra permissions | Latency | Category-only? Enforced where | Nested Hyprland in KDE | Real session | Verdict |
|---|---|---|---|---|---|---|---|---|
| A | **Hyprland Lua `input.keyboard.key` → `hl.dsp.event` → socket2 → `Hyprland.rawEvent`** | Low: ~30-line `bongocat.lua` + one `require` line + QML `Connections` | **None** | In-compositor callback + socket write; expected ≪ 1 frame *(unverified)* | **Yes.** In the compositor's Lua callback, before socket2. Timing still visible to socket2 readers. | Only keys typed while the nested window is focused *(path unverified)* | All keys, all keyboards Hyprland manages, including keys consumed by binds | **Recommended** |
| A′ | Same, transport via `hl.dsp.global` → `GlobalShortcut` | Low/medium: one shortcut per category | None | Same | Yes, same place; only the registering client receives it | Same | Same | Fallback transport if socket2 broadcast is unwanted *(press semantics unverified)* |
| B | Hyprland `catchall` / `non_consuming` binds | High, invasive: permanent submap, all binds `submap_universal` | None | Low | Only "unbound key". No identity available, so no categories without per-key binds. | Focus-dependent | Works, fragile | Rejected |
| C | Hyprland C++ plugin (event bus) | High: C++, hyprpm, rebuild every update, `plugin` permission | Plugin load permission | Lowest | Yes, in plugin | Same as A | Same as A | Rejected (A gives the same) |
| D | socket2 only (no config) | n/a | None | n/a | No key events exist | n/a | n/a | Not possible |
| E | **evdev helper, setgid `input`** | Medium: small C/Rust binary, install with `chown root:input; chmod 2755` (needs root once) | Helper only | Sub-ms read + pipe *(unverified)* | **Yes.** Inside the helper process | Sees **all** dev-machine typing (KDE too) | All physical keyboards, compositor-independent | **Fallback** |
| F | evdev helper as root system service + socket | Medium/high: system unit + socket unit + permissions | Daemon only (root or `DynamicUser`+`input`) | Same as E | Yes, in daemon | Same as E | Same as E | OK alternative to E, more ops work |
| G | evdev helper, user in `input` group | Low | **All user processes** become keylogger-capable | Same as E | Helper collapses, but any other process can read raw keys anyway | Same as E | Same as E (laptop may already be set up so by `dms setup`) | Avoid (privacy rule is spirit-violated) |
| H | evdev helper + uaccess udev rule for keyboards | Low/medium: one udev rule (root once) | All active-session processes | Same as E | Same as G | Same as E | Same as E | Avoid (≈ G) |
| I | evdev helper via pkexec | Low/medium: polkit policy | Helper only (root); password each login unless polkit rule | Same as E | Yes, in helper | Same as E | Same as E | Poor UX for autostart |
| J | libinput-based helper | As E | As E | As E | Yes, in helper | As E | As E | Same as E, more deps; no benefit |
| K | input-method-v2 grab | High | None, but exclusive per seat | Low | Would see every key (full keylogger) and must re-inject | — | Breaks fcitx/IME | Rejected |
| L | GlobalShortcuts portal / `GlobalShortcut` alone | Low | None | Low | n/a, named shortcuts only, no catch-all | — | — | Not possible |
| M | Input capture / libei | Medium | `input-capture` permission | Low | Consumes input while active | — | — | Rejected |

## 9. Recommendation

**Primary: approach A, the Hyprland Lua bridge.**

1. Ship `bongocat.lua` as the single component that ever sees keycodes. It holds an evdev-code → category table, emits **press events only** (`state == 1`), drops repeats, and calls `hl.dispatch(hl.dsp.event("bongocat,<category>"))`.
2. The DMS plugin listens on `Hyprland.rawEvent` for `name === "custom"` with the `bongocat,` prefix, and treats an unknown category as `other`.
3. The plugin can't tell "user isn't typing" apart from "bridge not installed", so it shows a dismissible hint ("add `require("bongocat")` to hyprland.lua") until it has seen its first `bongocat,` event.

Why it's best: zero privilege change, nothing to install as root, the category collapse happens inside the compositor (which already sees everything), and nothing else in the stack (DMS, Quickshell, the plugin) ever sees a key identity. The trade-off is that it is Hyprland-only and depends on a young Lua API, both acceptable for this project's stated target.

**Fallback: approach E, a setgid-`input` evdev helper**, if A fails hands-on verification or if the cat ever needs to run outside Hyprland. Requirements:

* minimal C (or Rust) with no config that changes output granularity;
* opens devices, then drops the gid;
* emits fixed tokens (`space\n`, `letter\n`, …) on stdout;
* is consumed by `Process` + `SplitParser`;
* uses wayland-bongocat-style device detection and inotify hotplug.

Comparing the permission models: setgid confines raw-key power to the helper with a one-time root install and no daemon. A root system service (F) achieves the same isolation at higher operational cost, and is only worth it if more than one consumer needs the stream. The input group (G) and uaccess (H) hand raw-key power to every user process, which contradicts the spirit of the privacy rule even if our own helper collapses correctly. pkexec (I) is wrong for something autostarted.

## 10. Open questions / needs hands-on verification

1. **Laptop config format:** is the laptop's Hyprland 0.57 running a Lua config (`~/.config/hypr/hyprland.lua`, `hyprctl version`)? Is it the DMS-generated one with `require("dms.*")`?
2. **Does `hl.on("input.keyboard.key", …)` fire in the nested Hyprland** (Wayland backend inside KDE)? Do `custom>>bongocat,…` lines reach DMS's `Hyprland.rawEvent` in that session? A quick check is `socat -U - UNIX-CONNECT:$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock | grep custom` inside the nested session. That shows only our category tokens, not keystrokes.
3. **Lock screen:** the Lua event fires before session-lock routing (§2.1), so category tokens and timing would flow while typing a password into the DMS lock screen. Decide whether to suppress. Candidate mechanisms: the plugin ignores events while DMS is locked (but they still hit socket2), or the Lua side checks lock state. No Lua lock-state query was found (only `hl.clear_crashed_lockscreen`), so this is *unverified*.
4. **Latency and CPU** of one Lua dispatch plus one socket2 event per keypress. Check for input lag during fast typing or key repeat.
5. **Keycode semantics:** the event carries a *physical* XKB keycode, not a keysym. On a German layout "letter" by physical position includes Z/Y (fine) but puts ä/ö/ü/ß on punctuation codes. Decide the taxonomy (see tickets).
6. **`hl.dsp.global` alternative:** does a dispatch outside a bind send press or release (`m_passPressed`)?
7. **Laptop `input` group:** run `id` on the laptop. If `dms setup` added the user, raw-key access is already open to all user processes. Decide whether to document or recommend anything about that (out of scope for the cat, but worth a note).
8. Whether Hyprland 0.57 still accepts a legacy hyprlang config at all. That matters only if the laptop isn't on Lua yet. *Unverified.*

## 11. Decisions this creates for other tickets

* **Category taxonomy ticket:** the exact category set and the evdev-code table (letters by physical position vs layout, umlauts, numpad, F-keys, media keys, modifiers incl. AltGr/Super). Whether release events or only presses are needed, since the animation may want arm-up timing.
* **Lock-screen policy ticket:** suppress or keep animation while the session is locked, and where to enforce it.
* **Install / distribution ticket:** how `bongocat.lua` gets into `~/.config/hypr/` and how the user is told to `require` it. DMS regenerates its own `dms/*.lua`, so this must not be placed inside DMS-managed files.
* **Transport contract ticket:** the wire format `bongocat,<category>` on socket2 (or the `GlobalShortcut` naming). Treat it as an interface shared by the Lua side and the plugin.
* **Fallback helper ticket (deferred):** only if verification of A fails. Setgid evdev helper with category-only output and its install story.
