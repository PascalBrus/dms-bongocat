# Can a DMS plugin show a widget that stays above normal windows?

Research for [PascalBrus/dms-bongocat#2](https://github.com/PascalBrus/dms-bongocat/issues/2). Researched 2026-11-06 from online primary sources only. No DMS install was available.

**Versions read**

- DMS: the repo **has moved/renamed** to `duxlabs-dev/dux-shell` (the old `AvengeMedia/DankMaterialShell` URL redirects).
  - Main branch: `master` @ [`fa1b1a2`](https://github.com/duxlabs-dev/dux-shell/tree/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e), committed 2026-11-06.
  - Latest release: `v2.1.1` @ [`a9a7e14`](https://github.com/duxlabs-dev/dux-shell/tree/a9a7e1439171ced668ff58d7d6eb010db546f932).
  - I also checked `v2.0.0` @ [`9b41329`](https://github.com/duxlabs-dev/dux-shell/tree/9b413296b16721f7679fb62e624cb1ed2433fa58). The desktop-widget window code quoted below is identical in v2.0.0, v2.1.1 and master.
- Quickshell docs: v0.3.0 (`quickshell.org/docs/v0.3.0/...`).
- Hyprland wiki: the versioned 0.57.0 pages.

## Answer

**Yes.** It works two ways, and the first needs no custom window code.

1. **Built-in option:** DMS desktop widgets (plugin `type: "desktop"`, or a `desktop` component in a composite plugin) have a per-instance **"Show on overlay"** toggle. Its description reads "Keeps the widget above windows instead of below them".
   - The toggle switches the widget's layer-shell surface from `WlrLayer.Bottom` (the default) to `WlrLayer.Overlay`.
   - The same instance settings also have a **"Click through"** toggle, which sets an empty input `Region` mask.
   - The widget window never takes keyboard focus unless the widget opts in, and it ignores exclusive zones.
   - The user sets these in Settings → Desktop widgets. They can also use IPC: `dms ipc call desktopWidget setOverlay <instanceId> true`.
2. **Custom window:** the plugin loader puts **no restriction** on plugin QML ("Plugins run with full QML runtime access"). A plugin (typically its `daemon` surface) can create its own `PanelWindow` with any `WlrLayershell.layer`. Several published community plugins do exactly this with `WlrLayer.Overlay`.

**Caveats**

- The plugin cannot declare in `plugin.json` that its desktop widget starts on Overlay. The user (or IPC) turns it on per instance.
- The plugin API is officially "experimental" and may break in minor releases.
- On Hyprland, fullscreen windows cover the **Top** layer but not **Overlay**. Use Overlay if the cat must also show over fullscreen apps.
- DMS's plugin API has **no keyboard-event API**. The existing community "Bongo Cat" plugin reads `/dev/input` through `evtest`/`libinput` (the user must be in the `input` group).

**Surprise:** a published **Bongo Cat plugin already exists** (`hthienloc/dms-plugins` → `bongoCat`). It is a composite bar widget plus desktop widget. See §3.

---

## 1. Plugin types and how they are declared

The manifest schema lists the allowed `type` values ([plugin-schema.json L42-45](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/PLUGINS/plugin-schema.json#L42-L45)):

```json
"enum": ["widget", "daemon", "launcher", "desktop", "dash", "dashCard", "composite"]
```

Required manifest fields: `id`, `name`, `description`, `version`, `author`, `type`, `capabilities`, and either `component` or `components`. Optional fields: `icon`, `settings`, `requires_dms`, `requires`, `permissions`, `startupCheck`, and `trigger` (required for launchers). Sources: [PLUGINS/README.md](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/PLUGINS/README.md#plugin-manifest-pluginjson) and the [agent skill manifest reference](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/.agents/skills/dms-plugin-dev/references/plugin-manifest-reference.md).

| Surface / `type` | `components` key | Base QML component (verified file) | Notes |
|---|---|---|---|
| Bar/dock widget (`widget`) | `widget` | `PluginComponent` — [Modules/Plugins/PluginComponent.qml](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/Modules/Plugins/PluginComponent.qml) | Provides `horizontalBarPill`/`verticalBarPill` and an optional `popoutContent`. The same component also renders in docks. |
| Control Center tile | (part of `widget`) | `PluginComponent.controlCenterWidget`, `ccWidget*`, `ccDetailContent` | Capability string `"control-center"`. There is no separate type ([README §Control Center Integration](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/PLUGINS/README.md#control-center-integration)). |
| Desktop widget (`desktop`) | `desktop` | `DesktopPluginComponent` — [Modules/Plugins/DesktopPluginComponent.qml](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/Modules/Plugins/DesktopPluginComponent.qml) (or a plain `Item` following the contract) | Hosted by `DesktopPluginWrapper.qml`. Users can drag and resize it, and position and size are persisted. Capability `"desktop-widget"`. |
| Daemon (`daemon`) | `daemon` | any `Item` exposing `pluginService`/`pluginId` | Instantiated once. Suited to IPC handlers and background work ([README §Composite Plugins/Surfaces](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/PLUGINS/README.md#surfaces)). |
| Launcher (`launcher`) | `launcher` | launcher contract (`getItems`/`executeItem`) | Needs `trigger`. |
| Dash tab (`dash`) | `dash` | `DashTabComponent` | |
| Dash overview card (`dashCard`) | `dashCard` | `DashCardComponent` | |
| `composite` | any subset of the keys above | — | "Each surface is loaded independently into its own registry, so the same plugin can show up in the bar **and** on the desktop **and** run a daemon simultaneously." Composite plugins obey the enable toggle. A pure `desktop` plugin still auto-loads. |

Settings UI: a `PluginSettings` component, which needs the `settings_write` permission ([README §Security](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/PLUGINS/README.md#security-considerations)).

Example composite manifest, from the README:

```json
{ "type": "composite",
  "capabilities": ["daemon", "dankbar-widget", "desktop-widget"],
  "components": { "daemon": "./MyDaemon.qml", "widget": "./MyBarWidget.qml", "desktop": "./MyDesktopWidget.qml" } }
```

## 2. Layer used by DMS desktop widgets, and whether it is configurable

The window is created in `DesktopPluginWrapper.qml`. These permalinks point to the v2.1.1 tag, and the lines are the same at v2.0.0 and master ([L17-21](https://github.com/duxlabs-dev/dux-shell/blob/a9a7e1439171ced668ff58d7d6eb010db546f932/quickshell/Modules/Plugins/DesktopPluginWrapper.qml#L17-L21), [L55-82](https://github.com/duxlabs-dev/dux-shell/blob/a9a7e1439171ced668ff58d7d6eb010db546f932/quickshell/Modules/Plugins/DesktopPluginWrapper.qml#L55-L82)):

```qml
readonly property bool showOnOverlay: instanceData?.config?.showOnOverlay ?? false
...
readonly property bool clickThrough: instanceData?.config?.clickThrough ?? false
...
PanelWindow {
    id: widgetWindow
    ...
    Region { id: emptyMask }
    mask: root.clickThrough ? emptyMask : null

    WlrLayershell.namespace: "dms:desktop-widget:" + root.pluginId + (root.instanceId ? ":" + root.instanceId : "")
    WlrLayershell.layer: {
        if (root.showOnOverlay)
            return WlrLayer.Overlay;
        if (root.overviewActive && (root.showOnOverview || root.showOnOverviewOnly))
            return WlrLayer.Overlay;
        return WlrLayer.Bottom;
    }
    WlrLayershell.exclusionMode: ExclusionMode.Ignore
    WlrLayershell.keyboardFocus: {
        if (PopoutManager.screenshotActive)
            return WlrKeyboardFocus.None;
        if (content.acceptsKeyboardFocus)
            return WlrKeyboardFocus.OnDemand;
        return WlrKeyboardFocus.None;
    }
    anchors { left: true; top: true }
    WlrLayershell.margins { left: geometry.widgetX; top: geometry.widgetY }
```

What the code shows:

- **Default layer: `Bottom`.** The widget sits above the wallpaper and below normal windows. The README's "background layer" wording is loose: wallpaper uses `WlrLayer.Background` ([WallpaperBackground.qml L31](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/Modules/WallpaperBackground.qml#L31)).
- **`Overlay` when the instance config has `showOnOverlay: true`.** The `showOnOverview` options only apply on Niri, where `overviewActive` requires `CompositorService.isNiri`.
- **No `Top` option exists** for desktop widgets.
- **Users configure it per widget instance**, not per plugin:
  - Settings UI: [DesktopWidgetTab.qml L96-124](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/Modules/Settings/DesktopWidgetTab.qml#L96-L124) has the "Show on overlay" toggle ("Keeps the widget above windows instead of below them") and the "Click through" toggle. Both are hidden for lock-screen instances.
  - IPC: [DMSShellIPC.qml L1697-1726](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/DMSShellIPC.qml#L1697-L1726) defines `toggleOverlay(instanceId)`, `setOverlay(instanceId, "true"|"false")`, `toggleClickThrough`, `setClickThrough`, `list()` and `status()` under IPC target `desktopWidget` ([L1859](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/DMSShellIPC.qml#L1859)). Usage should be `dms ipc call desktopWidget setOverlay <id> true`; the exact CLI form is unverified, see open questions.
- **No manifest field lets the plugin choose its own layer.** It is unofficial and unsupported, but a plugin could call `SettingsData.updateDesktopWidgetInstanceConfig(instanceId, { showOnOverlay: true })` itself, because plugins can import `qs.Common`. That is internal API, not plugin API.

## 3. Can a plugin create its own PanelWindow / layer-shell window?

**Nothing in the loader blocks it.**

- `PluginService` loads each surface with `Qt.createComponent(url, …)` and `createObject(root, …)`. It has no allow-list of types or imports ([PluginService.qml L547-567](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/Services/PluginService.qml#L547-L567), [L616-619](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/Services/PluginService.qml#L616-L619)).
- The README says so directly: *"Plugins run with full QML runtime access. Only install plugins from trusted sources."* Of the permissions, only `settings_write` is enforced, and "Sandboxed plugin execution" is listed as roadmap ([README §Security / §API Stability](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/PLUGINS/README.md#security-considerations)).

Existing plugins that do this:

- **agneswd/dms-voxtype-activity-overlay** (a `daemon`, root is `PluginComponent`) declares a `PanelWindow` with `mask: Region { item: cancelButton }`, `exclusiveZone: 0` and `WlrLayershell.layer: WlrLayer.Overlay` ([DmsVoxtypeActivityOverlay.qml L309-324 @ dd7c00e](https://github.com/agneswd/dms-voxtype-activity-overlay/blob/dd7c00ed978717a74adc0e0c4944159f6aa44dfe/DmsVoxtypeActivityOverlay.qml#L309-L324)).
- **rdannenbring/voxtype-dms-overlay** has one full-screen `PanelWindow` per monitor with `WlrLayershell.keyboardFocus: WlrKeyboardFocus.None`, `exclusiveZone: -1` and a mask covering only a close button ([OverlayWindow.qml L17-34 @ c1be466](https://github.com/rdannenbring/voxtype-dms-overlay/blob/c1be466e170adcbf4dc4ee1e9fa94d37b53e0955/OverlayWindow.qml#L17-L34)).
  - Caution: it writes `WlrLayershell.layer: WlrLayershell.Overlay`, which is most likely a typo for `WlrLayer.Overlay`. The value would be undefined, and the window would probably stay on the default `Top`. Don't copy that line.
- **DMS's own first-party plugins** (`duxlabs-dev/dux-plugins`, formerly `AvengeMedia/dms-plugins`) contain no `PanelWindow` or `WlrLayershell` usage. They all use the hosted surfaces.
- **Existing Bongo Cat:** [`hthienloc/dms-plugins` → `bongoCat`](https://github.com/hthienloc/dms-plugins/tree/b7ec1d6cd0fa1d42cab7c2f5996305c34897275c/bongoCat) (registry entry `hthienloc-bongo-cat.json`, `requires_dms >= 2.0.0`, v2.5.0).
  - It is `type: "composite"` with `widget` and `desktop` components, and it has no custom window.
  - Its README says it needs `evtest` (specific keyboard) or the `libinput` CLI ("All Keyboards" mode), and that the user must be in the `input` group.
  - It is worth reading for prior art, or as a reason to reconsider building our own.
- The community registry (`duxlabs-dev/dux-plugin-registry` @ `7dbadcb`, 406 entries) links to the web registry at <https://plugins.duxlabs.dev/>, according to the README.

**Keypress access:** DMS plugins get no global key-event API. Wayland gives no global key snooping to clients, and layer surfaces only receive keys while they have focus. Prior art reads evdev (`/dev/input/event*`) through an external process such as `evtest` or `libinput debug-events`. For our privacy goal (categories only), that process should map keycodes to categories before the data reaches QML. Run it through Quickshell `Process`/`StdioCollector` with the `process` permission declared, even though that permission is not enforced.

## 4. Input/focus, click-through, exclusive zone, Hyprland rules, fullscreen

**Keyboard focus**

The Quickshell `WlrLayershell.keyboardFocus` property "Defaults to `KeyboardFocus.None`" ([docs v0.3.0](https://quickshell.org/docs/v0.3.0/types/Quickshell.Wayland/WlrLayershell/)). The `WlrKeyboardFocus` values are ([docs](https://quickshell.org/docs/v0.3.0/types/Quickshell.Wayland/WlrKeyboardFocus/)):

- `None`: "No keyboard input will be accepted."
- `OnDemand`: "Access to the keyboard as determined by the operating system."
- `Exclusive`: "Exclusive access to the keyboard, locking out all other windows."

The wlr-layer-shell protocol adds that for the Top and Overlay layers, the top-most `exclusive` surface always gets keyboard focus ([protocol](https://wayland.app/protocols/wlr-layer-shell-unstable-v1)). **Use `None`.**

DMS desktop widgets already use `None` unless the widget component sets `acceptsKeyboardFocus: true`, which switches them to `OnDemand` (wrapper L76-82, [DesktopWidgetContent.qml L47](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/Modules/Plugins/DesktopWidgetContent.qml#L47)). `PanelWindow.focusable` "Defaults to false" ([docs](https://quickshell.org/docs/v0.3.0/types/Quickshell/PanelWindow/)).

**Click-through**

`QsWindow.mask` is "The clickthrough mask… If non null then the clickable areas of the window will be determined by the provided region" ([docs](https://quickshell.org/docs/v0.3.0/types/Quickshell/QsWindow/)).

- An empty `Region {}` gives no clickable area, so the window is fully click-through. DMS does exactly this for desktop widgets with "Click through" on.
- `Region { item: someChild }` makes only that child clickable.

Caveat: with an empty mask, the DMS desktop widget's right-click context menu and edit-mode drag can't be reached on that surface. Positioning is presumably done in edit mode, so check the order (turn click-through on after positioning).

**Exclusive zone**

`ExclusionMode.Ignore` is "Ignore exclusion zones of other shell layers. You cannot set an exclusion zone in this mode" ([docs](https://quickshell.org/docs/v0.3.0/types/Quickshell/ExclusionMode/)). DMS desktop widgets use `Ignore`, so the widget doesn't reserve space or push tiled windows.

For a custom window, use `exclusionMode: ExclusionMode.Ignore`, or `exclusiveZone: 0` (protocol: "would like to be moved to avoid occluding surfaces with a positive exclusive zone", meaning it avoids the bar). Avoid the default `Auto` with 3 anchors, which reserves space.

**Hyprland layer rules (0.57)**

The 0.57.0 wiki documents **Lua config**, not the old `layerrule = …` lines ([wiki 0.57.0 Layer rules](https://wiki.hypr.land/0.57.0/configuring/core/rules/layer-rules/)):

```lua
hl.layer_rule({ match = { namespace = "^dms:desktop-widget:bongoCat.*" }, no_anim = true })
```

- Effects are now snake_case: `above_lock`, `animation`, `blur`, `blur_popups`, `dim_around`, `ignore_alpha` (float 0–1; replaces the old `ignorezero`/`ignorealpha`), `no_anim`, `no_screen_share`, `order`, `xray`.
- `namespace` is a regex. Check it with `hyprctl layers`.
- DMS's own generated Hyprland config already adds `hl.layer_rule({ match = { namespace = "^dms:.*" }, no_anim = true })` ([embedded/hyprland.lua L81-82](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/core/internal/config/embedded/hyprland.lua#L81-L82)). A desktop widget namespace (`dms:desktop-widget:<pluginId>:<instanceId>`) is covered by that rule.
- A custom window should also use a `dms:`-prefixed namespace, such as `dms:bongocat`, to get the same rule.
- Optional: `no_screen_share = true` keeps the cat out of screen shares.
- ⚠️ I did not check whether Hyprland 0.57 still accepts the legacy hyprlang `layerrule =` syntax. Use the Lua form.

**Fullscreen vs Top/Overlay**

- The protocol says layers are z-ordered Background < Bottom < Top < Overlay, and "Fullscreen shell surfaces are typically rendered at the top layer" ([protocol](https://wayland.app/protocols/wlr-layer-shell-unstable-v1)).
- In Hyprland, a fullscreen window covers Top-layer surfaces (this is why bars hide). Overlay-layer surfaces stay above fullscreen windows. Sources: [Omarchy PR #6429](https://github.com/basecamp/omarchy/pull/6429) (says Hyprland renders a fullscreen window above top-layer surfaces; I only saw this through a search summary and did not read the PR myself), and [Hyprland discussion #15881](https://github.com/hyprwm/Hyprland/discussions/15881) / [#14721](https://github.com/hyprwm/Hyprland/discussions/14721), which describe edge cases where a newly created Top surface briefly shows over fullscreen.
- What this means here:
  - **Top:** above normal and floating windows, hidden under fullscreen (games, videos). Arguably the nicer behaviour for a cat.
  - **Overlay:** above everything, fullscreen included. It may also sit above DMS popouts and launchers, which DMS puts on Top by default.
- DMS desktop widgets only offer Bottom or Overlay. Getting **Top** needs a custom `PanelWindow`.

## 5. Fallback: the top-bar widget (`PluginComponent`)

- **Pill components:** `horizontalBarPill` (top/bottom bars) and `verticalBarPill` (left/right bars). Define both, because docks can sit on any edge ([widget-plugin-guide.md](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/.agents/skills/dms-plugin-dev/references/widget-plugin-guide.md)).
- **Injected properties:** `axis` / `isVertical`, `section` (`"left"`, `"center"` or `"right"`), `widgetThickness`, `barThickness`, `barConfig`, `parentScreen`, and `surfaceContext` (`kind`: `"bar"` or `"dock"`, plus `isVertical` and `thickness`).
- **Placement:** the user places the widget in a section in bar settings. Each bar config has `leftWidgets`, `centerWidgets` and `rightWidgets`. There can be multiple bars, on top, bottom, left or right ([SettingsSpec.js barConfigs L1630-1700](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/Common/settings/SettingsSpec.js#L1630-L1700)).
- **Size limits:** the thickness is set by the bar's `innerPadding`, which the settings slider allows from −8 to 24 (default 2) ([Theme.qml L2351-2357](https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/Common/Theme.qml#L2351-L2357), `barHeight: 48` at L1297):
  ```js
  barWidgetThickness(p) = snapEven(max(20, 26 + p*0.6))
  barThickness(p)       = snapEven(max(barWidgetThickness(p) + p + 4, 48 - 4 - (8 - p)))
  ```
  - At default `p = 2`, the pill is **≈28 px** tall in a ≈38 px bar.
  - The range is ≈22 px (p = −8) to ≈40 px (p = 24). These are my calculations, not measured.
  - The pill's length along the bar is up to the content.
  - A tiny cat fits, but it will be small (icon-sized). A `popoutContent` can show a bigger view on click.

## Recommended approach for Bongo Cat

1. **Make the plugin a `composite`** with `desktop` and `widget` components, plus a `daemon` that runs the key-category reader process and publishes state with `PluginService.setGlobalVar(pluginId, …)`. This shares one input reader across surfaces. A single reader in the daemon matters, because the bar widget is instantiated per bar per screen.
2. **Primary path, with no custom window:** a `DesktopPluginComponent`-based cat. Tell the user (README or settings hint) to turn on **Show on overlay** and **Click through** for the instance, or to run `dms ipc call desktopWidget setOverlay <id> true` and `setClickThrough <id> true`. Leave `acceptsKeyboardFocus` unset so focus stays `None`. This gets drag and resize, persistence, multi-monitor support and the lock-screen variant for free.
3. **Option when Top-layer behaviour is wanted** (above windows, hidden under fullscreen): a `PanelWindow` that the daemon creates itself, sketched below.

```qml
// Inside the daemon surface (root: Item / PluginComponent). Requires: import Quickshell, Quickshell.Wayland
Variants {
    model: Quickshell.screens
    PanelWindow {
        required property var modelData
        screen: modelData
        visible: root.enabled                    // destroy the surface when off
        color: "transparent"
        implicitWidth: 160; implicitHeight: 120
        anchors { right: true; bottom: true }
        margins { right: 24; bottom: 24 }        // or user-configurable

        WlrLayershell.namespace: "dms:bongocat"  // matched by DMS's ^dms:.* no_anim rule
        WlrLayershell.layer: WlrLayer.Top        // or WlrLayer.Overlay to stay above fullscreen
        WlrLayershell.keyboardFocus: WlrKeyboardFocus.None
        exclusionMode: ExclusionMode.Ignore      // don't reserve space / don't push windows
        mask: Region {}                          // fully click-through

        BongoCat { anchors.fill: parent; state: root.catState }
    }
}
```

The custom-window path loses DMS's edit-mode drag, resize and persistence. Positioning would have to come from settings (anchor corner plus margins).

4. Keep the bar widget (`horizontalBarPill` and `verticalBarPill`) as the always-available fallback.

## Open questions / to verify on a real DMS install

- What exact IPC CLI syntax and instance IDs does the install use? Check `dms ipc call desktopWidget list`, then `setOverlay <id> true`.
- Does a desktop-widget instance with **Click through** on still allow moving and resizing in DMS edit mode? (`DesktopWidgetEditor.qml` uses its own Overlay window, which suggests yes.)
- On Hyprland 0.57: confirm that Overlay shows over a fullscreen window and Top does not, using `hyprctl layers`. Check the behaviour when a widget is created while a fullscreen window is active (discussion #15881).
- Is the installed DMS ≥ 2.0.0? The `showOnOverlay` code is confirmed in v2.0.0, v2.1.1 and master. Package name and version on the target distro: `dms-shell`.
- Does Hyprland 0.57 still accept the legacy hyprlang `layerrule =` config, or only Lua?
- Should Overlay-layer widgets sit above DMS popouts and launchers? DMS puts those on Top by default, and on Overlay via `DMS_POPOUT_LAYER` or the bar `useOverlayLayer` setting. Check for visual clashes.
- Evdev access needs `input` group membership, which is a real privacy and security footgun. Decide whether this plugin reads evdev itself or reuses the approach of the existing `bongoCat` plugin. Or reuse that plugin outright.

## Sources

- DMS repo (moved): https://github.com/duxlabs-dev/dux-shell (master `fa1b1a2`, tag v2.1.1 `a9a7e14`, tag v2.0.0 `9b41329`)
  - DesktopPluginWrapper.qml: https://github.com/duxlabs-dev/dux-shell/blob/a9a7e1439171ced668ff58d7d6eb010db546f932/quickshell/Modules/Plugins/DesktopPluginWrapper.qml#L55-L82
  - DesktopWidgetTab.qml: https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/Modules/Settings/DesktopWidgetTab.qml#L96-L124
  - DMSShellIPC.qml (desktopWidget IPC): https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/DMSShellIPC.qml#L1697-L1726
  - PluginService.qml: https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/Services/PluginService.qml#L547-L619
  - PLUGINS/README.md: https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/PLUGINS/README.md
  - plugin-schema.json: https://github.com/duxlabs-dev/dux-shell/blob/fa1b1a2198ec9cd58d06eeae1f0951bfa899855e/quickshell/PLUGINS/plugin-schema.json
  - PluginComponent.qml, Theme.qml, SettingsSpec.js, embedded/hyprland.lua (permalinks inline above)
- First-party plugins: https://github.com/duxlabs-dev/dux-plugins (formerly AvengeMedia/dms-plugins)
- Registry: https://github.com/duxlabs-dev/dux-plugin-registry (`7dbadcb`), https://plugins.duxlabs.dev/
- Community plugins:
  - https://github.com/hthienloc/dms-plugins/tree/b7ec1d6cd0fa1d42cab7c2f5996305c34897275c/bongoCat
  - https://github.com/agneswd/dms-voxtype-activity-overlay/blob/dd7c00ed978717a74adc0e0c4944159f6aa44dfe/DmsVoxtypeActivityOverlay.qml#L309-L324
  - https://github.com/rdannenbring/voxtype-dms-overlay/blob/c1be466e170adcbf4dc4ee1e9fa94d37b53e0955/OverlayWindow.qml#L17-L34
- Quickshell docs v0.3.0:
  - https://quickshell.org/docs/v0.3.0/types/Quickshell.Wayland/WlrLayershell/
  - https://quickshell.org/docs/v0.3.0/types/Quickshell.Wayland/WlrKeyboardFocus/
  - https://quickshell.org/docs/v0.3.0/types/Quickshell/PanelWindow/
  - https://quickshell.org/docs/v0.3.0/types/Quickshell/QsWindow/ (mask)
  - https://quickshell.org/docs/v0.3.0/types/Quickshell/ExclusionMode/
- wlr-layer-shell protocol (v5): https://wayland.app/protocols/wlr-layer-shell-unstable-v1
- Hyprland wiki:
  - 0.57.0 layer rules: https://wiki.hypr.land/0.57.0/configuring/core/rules/layer-rules/
  - Latest: https://wiki.hypr.land/configuring/core/rules/layer-rules/
- Fullscreen vs Top layer:
  - https://github.com/basecamp/omarchy/pull/6429
  - https://github.com/hyprwm/Hyprland/discussions/15881
  - https://github.com/hyprwm/Hyprland/discussions/14721
