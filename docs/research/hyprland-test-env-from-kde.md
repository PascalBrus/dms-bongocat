# Hyprland + DMS test environment from a KDE dev machine

Researched 2026-11-06. Dev machine: CachyOS, KDE Plasma 6.8.1 Wayland (kwin_wayland), RX 9070 + Radeon 610M.
Repo versions at that date: `hyprland 0.57.0`, `quickshell 0.3.2`, `dux-shell 2.0.0` (extra), `dms-shell 2.0.0` (cachyos).
Nothing below has been run on this machine yet. Anything marked **(unverified)** is my inference, not a quote from a source.

## Question

What is the best way to get a throwaway Hyprland + DMS session for testing the Bongo Cat DMS plugin from the KDE
dev machine? The candidates are:

1. Hyprland nested as a window inside KWin
2. A VM (qemu/virt-manager)
3. A container (distrobox/podman)
4. A second login session

The options are weighed on setup effort, fidelity (layer-shell layers, global key reading), keeping the host KDE
setup clean, and how fast a plugin edit in the repo shows up on screen.

## TL;DR

- **Use nested Hyprland inside KWin for the daily loop.** Hyprland's backend library, Aquamarine, can render "on a
  Wayland session (in a window)". Hyprland falls back to that Wayland backend when it can't get DRM. In a nest it
  skips all the systemd/D-Bus environment handling, so it doesn't touch the KDE session's environment. Layer-shell
  behaves like the real thing, because the nested Hyprland is the compositor DMS talks to.
  - Launch it under `dbus-run-session` so DMS's notification server, tray and polkit agent don't clash with Plasma.
  - Start DMS with `dms run` from the Lua `hyprland.start` hook.
  - Symlink the plugin from the repo into `~/.config/dms/plugins/`.
  - Reload with `dms ipc call plugins reload <id>`.
- **Biggest fidelity gap: global keys.** A nested compositor only gets keys while its window has focus in KWin, and
  KWin keeps its own shortcuts. That matters only if the plugin gets keys from the compositor (Hyprland binds/IPC).
  If it reads evdev (`/dev/input`), as other Wayland Bongo Cat overlays do, it sees every host key whichever
  compositor is in front. But then the dev user needs `input` group access, and it does not have it today.
- **Fallback for full-fidelity checks: a separate test user, logged in on another VT or via SDDM** and running
  Hyprland + DMS on real DRM/libinput. Only that test user needs the `input` group, and your `~/.config` stays out of
  it. Avoid a second Hyprland session as the *same* user while Plasma is running (it shares the systemd user manager,
  see below).
- **Skip containers.** distrobox shares `$HOME`, the Wayland socket and devices, and "isolation and sandboxing are
  not the main aims". That gives no real fidelity gain over nesting and adds setup work.
- **Keep VMs as a last resort.** Hyprland needs 3D acceleration in the VM ("otherwise Hyprland will not work"), and
  the wiki calls VM use "not officially supported".
- **dux-shell is DMS, renamed.** `AvengeMedia/DankMaterialShell` now 301-redirects to `duxlabs-dev/dux-shell`.
  Install `dux-shell`: it `Provides: dms-shell=2.0.0`, `Replaces: dms-shell`, and the CLI is still `dms`.

## Options comparison

| Option | Setup effort | Layer-shell fidelity (bar, overlay widget) | Global-key fidelity | Host cleanliness | Iteration speed |
|---|---|---|---|---|---|
| **Nested Hyprland in KWin** | Low: 3 packages + 1 Lua file | High. It is the real Hyprland layer-shell implementation, but in one window/output, with no real monitor or hotplug | Compositor keys: only while the nest is focused, and KWin shortcuts are eaten first. evdev: full, but needs `input` group on the dev user | Good. Packages + `~/.config/dms`, `~/.config/hypr*`. No session env changes in a nest. Use `dbus-run-session` to avoid D-Bus name clashes | Best: seconds, with the editor on the host next to it |
| **Second session, dedicated test user** (TTY or SDDM) | Low–medium: same packages + a user | Highest: real DRM, libinput, monitors, fullscreen | Full. Real seat. evdev works if only the test user is in `input` | Very good. Separate `$HOME`, separate systemd user manager | Slow: VT switch, editor not in the same session. Symlink needs a shared, readable path |
| Second session, *same* user | Low | Highest | Full | **Poor.** Hyprland imports env into the shared systemd user manager and starts `hyprland-session.target` | Slow (VT switch) |
| VM (libvirt/QEMU, virgl) | High: guest install, GL setup | Medium–high. Real Hyprland, but on virtio-gpu, "may be pretty slow" | Guest gets keys only when the viewer has focus. evdev inside the guest sees only virtual devices **(unverified)** | Excellent once libvirt is installed (needs `libvirt` group etc.) | Poor: code has to be shared or copied into the guest |
| Container (distrobox/podman) | Medium–high: no first-party docs for running a compositor in one | Same as nesting at best, because it would still nest in KWin | Same as nesting | Weak: shares `$HOME` and devices by design | Similar to nesting, more friction |

## Details

### 1. Nested Hyprland inside KWin (recommended)

- **Aquamarine supports it.** Aquamarine "provides basic abstractions for an application to render on a Wayland
  session (in a window) or a native DRM session"
  ([Aquamarine README](https://github.com/hyprwm/aquamarine/blob/main/README.md)).
- **Backend selection.** Hyprland registers backends as headless (mandatory), DRM (if available), then **Wayland
  (fallback)** ([Hyprland `src/Compositor.cpp`](https://github.com/hyprwm/Hyprland/blob/main/src/Compositor.cpp)).
  So starting `Hyprland` from a terminal inside KWin should land on the Wayland backend.
- **What the host compositor must offer.** The Wayland backend binds `wl_seat`, `xdg_wm_base`, `wl_compositor`,
  `wl_shm` and `zwp_linux_dmabuf_v1`, and aborts with "Missing protocols" if one is absent
  ([Aquamarine `src/backend/Wayland.cpp`](https://github.com/hyprwm/aquamarine/blob/main/src/backend/Wayland.cpp)).
  KWin 6 provides all of these **(unverified on this machine; first smoke-test item)**.
- **Official docs only describe nesting inside Hyprland.** The wiki covers nesting for developers: "Launch the
  output `Hyprland` binary … _when logged into a Hyprland session_. A new window should open with Hyprland running
  inside of it." It recommends removing autoexec directives and changing the bind modifier `SUPER → ALT`
  ([wiki: Contributing & Debugging](https://wiki.hypr.land/Contributing-and-Debugging/)). The plugin page says "you
  definitely should use a nest" unless you need real hardware like trackpad gestures
  ([wiki: Plugin dev, getting started](https://wiki.hypr.land/Plugins/Development/Getting-Started/)). I found no
  wiki statement either supporting or ruling out KWin as the parent compositor.
- **No session pollution.** The env-import block is gated on `m_aqBackend->hasSession()`, with the code comment
  "Session-less Hyprland usually means a nest, don't update the env in that case". The same gate covers
  `systemctl --user start hyprland-session.target`
  ([Compositor.cpp](https://github.com/hyprwm/Hyprland/blob/main/src/Compositor.cpp)). As a result:
  - The nest won't clobber Plasma's `WAYLAND_DISPLAY` / `XDG_CURRENT_DESKTOP` in the systemd/D-Bus activation
    environment.
  - The DMS systemd unit won't start, which is fine: start DMS with `dms run` from the config instead.
- **Config format.** As of 0.57 the config is Lua (`~/.config/hypr/hyprland.lua`). Autostart uses
  `hl.on("hyprland.start", function() hl.exec_cmd(...) end)`, and env vars use `hl.env()`
  ([wiki: Configuring/Core](https://wiki.hypr.land/Configuring/Core/),
  [Autostart](https://wiki.hypr.land/Configuring/Core/Autostart/),
  [Environment variables](https://wiki.hypr.land/Configuring/Core/Environment-variables/)). DMS states that
  "Hyprland 0.57 reads only `hyprland.lua`" and that DMS needs Hyprland ≥ 0.55
  ([DMS: Compositors](https://duxlabs.dev/docs/dms/compositors)). `--config` / `-c` picks a non-default file.
- **Aquamarine env vars.** `AQ_TRACE`, `AQ_DRM_DEVICES`, `AQ_NO_MODIFIERS`, `AQ_MGPU_NO_EXPLICIT`, `AQ_NO_ATOMIC`,
  `AQ_LIBINPUT_NO_PLUGINS` ([Aquamarine docs/env.md](https://github.com/hyprwm/aquamarine/blob/main/docs/env.md)).
  All except `AQ_TRACE` concern DRM/libinput, so in a nest only `AQ_TRACE=1` is useful, for debugging. The nested
  backend picks its GPU from the parent's dmabuf feedback "main device" (Wayland.cpp `initDmabuf`).
- **Layer-shell fidelity.** DMS and the plugin talk `zwlr_layer_shell_v1` to the *nested* Hyprland, not to KWin, so
  Hyprland's layer ordering, layer rules (`dms:*` namespaces) and `ext-background-effect-v1` blur all apply.
  - Quickshell's `WlrLayershell.layer` defaults to `WlrLayer.Top`
    ([Quickshell WlrLayershell](https://quickshell.org/docs/v0.3.0/types/Quickshell.Wayland/WlrLayershell/)).
  - DMS namespaces: bar `dms:bar`, desktop widgets `dms:desktop-widget:<plugin id>`, plugin popouts
    `dms:plugins:<ns>` ([DMS: Layers](https://duxlabs.dev/docs/dms/layers)).
  - Limits: one output only (the KWin window), unless you add one with `hyprctl output create wayland`, which
    "will only work if you're already running Hyprland with the Wayland backend, i.e. as a nested session"
    ([wiki: Using hyprctl](https://wiki.hypr.land/Configuring/Core/Advanced-configuration/Using-hyprctl/)). Fullscreen
    behaviour and scaling follow the window, not a real panel.
- **Keys.** KWin delivers keyboard events to the nested window only while it has focus. KWin's own global shortcuts
  (Meta, Alt+Tab, etc.) are handled before the client sees them **(unverified; standard compositor behaviour)**. So
  use `ALT` as the nest's main modifier, as the wiki suggests for dev nests.

### 2. Second login session

- The wiki says to launch Hyprland with `start-hyprland` from a TTY, never as root. On login managers: "SDDM: Works
  flawlessly", "plasma-login-manager: Works flawlessly"
  ([wiki: Master tutorial](https://wiki.hypr.land/Getting-Started/Master-Tutorial/)).
- **Same user while Plasma is running is risky.** On a real session, Hyprland runs
  `systemctl --user import-environment … WAYLAND_DISPLAY … XDG_CURRENT_DESKTOP …` and
  `systemctl --user start hyprland-session.target`. On exit it runs `systemctl --user stop graphical-session.target`
  ([Compositor.cpp](https://github.com/hyprwm/Hyprland/blob/main/src/Compositor.cpp)). One user has one systemd
  user manager, so this would hit the running Plasma session. Plasma 6 is systemd-managed, so stopping
  `graphical-session.target` could take Plasma services down **(unverified, but follows from the code)**.
  `HYPRLAND_NO_SD_VARS=1` disables the env management and `HYPRLAND_NO_SD_TARGET` skips the target (same file).
  Logging out of Plasma first avoids the problem entirely.
- **A dedicated test user avoids all of this.** It gets its own user manager, its own `$HOME`, and its own `input`
  group membership. That is the only way here to test evdev key reading without granting `input` to your main user.
  The cost is getting the plugin into the test user's home: copy it, or use a group-readable checkout.

### 3. VM (QEMU/libvirt)

- "In a VM, make sure you have 3D acceleration enabled in your `virtio` config (or `virt-manager`), otherwise
  Hyprland _**will not work**_ … 3D acceleration in VMs may be pretty slow"
  ([wiki: Master tutorial](https://wiki.hypr.land/Getting-Started/Master-Tutorial/)).
- The install page marks VM use "_YMMV, this is not officially supported._" Its example is `virt-install` with
  `--graphics spice,listen=none,gl.enable=yes,rendernode=/dev/dri/renderD128`, plus "Make sure you install `mesa`
  … The virgl drivers are included in `mesa`"
  ([wiki: Installation → Running in a VM](https://wiki.hypr.land/Getting-Started/Installation/)).
- QEMU: virglrenderer is the `gl` device variant (`virtio-gpu-gl`, `virtio-vga-gl`). Vulkan through Venus needs
  `blob=true,hostmem=…,venus=true` ([QEMU virtio-gpu docs](https://www.qemu.org/docs/master/system/devices/virtio/virtio-gpu.html)).
- The FAQ notes that the host compositor grabs keybinds before the VM ("Hyprland takes precedence")
  ([wiki: FAQ](https://wiki.hypr.land/FAQ/)). The same applies to KWin as host.
- Verdict: the best host isolation, but the most setup (libvirt, group membership, guest OS) and the slowest edit
  loop. This machine has 16 cores and a strong dGPU, so speed is fine. Effort is the cost.

### 4. Container (distrobox/podman)

- distrobox gives "access to the user's home directory, the Wayland and X11 sockets, networking, removable
  devices", and "Isolation and sandboxing are **not** the main aims of the project"
  ([distrobox.it](https://distrobox.it/)).
- A compositor inside a container would still need either the host Wayland socket (making it a nest, like
  option 1) or DRM/seat access (which conflicts with KWin owning the seat). The useful-tips page documents nothing
  about running a compositor or desktop session ([distrobox useful tips](https://distrobox.it/useful_tips/)).
- Verdict: no fidelity gain over option 1. It also needs podman + distrobox installed on the host, and the image
  would have to be Arch-based to get `hyprland 0.57` / `dux-shell`. Not recommended.

### Global key input: what a Bongo Cat plugin will actually see

- Existing Wayland Bongo Cat overlays read evdev. `saatvik333/wayland-bongocat` reads keyboards from `/dev/input`,
  requires the `input` group ("Reading keyboards needs access to `/dev/input`"), and offers
  `layer = background|bottom|top|overlay`. "The `overlay` layer … always stay[s] visible" over fullscreen windows
  ([wayland-bongocat README](https://github.com/saatvik333/wayland-bongocat)).
- **evdev reader in the DMS plugin** (e.g. a daemon `Process` running a helper). It sees every key from every
  compositor, nested or not, so a nest is fine for this. It needs `input` on the user that runs DMS, which the dev
  user lacks.
- **Hyprland-side keys.** Hyprland IPC has no per-key event, only `activelayout` and `submap`
  ([wiki: IPC](https://wiki.hypr.land/IPC/)). Bindings can use the `non_consuming` flag ("Key/mouse events will be
  passed to the active window in addition to triggering the dispatcher")
  ([wiki: Bind flags](https://wiki.hypr.land/Configuring/Core/Binds/Flags/)), which could call
  `dms ipc call …` per key. In a nest, that only fires for keys typed into the nest window.
- **Quickshell/Wayland keyboard focus.** A layer surface only gets keys while it has keyboard focus. DMS desktop
  widgets have `acceptsKeyboardFocus` off by default
  ([DMS: Plugin development](https://duxlabs.dev/docs/dms/plugin-development)). So this route can't produce
  "global" key reading anyway.

## dux-shell vs dms-shell

- `https://github.com/AvengeMedia/DankMaterialShell` returns **HTTP 301 → `https://github.com/duxlabs-dev/dux-shell`**.
  The GitHub API reports `fork: false` and `created_at: 2025-07-10` (DMS's original creation date). So this is a
  rename/transfer, not a fork. `danklinux.com/docs` 301-redirects to `duxlabs.dev/docs`.
- The README now expands DMS as "Dux Modular Shell (DMS)" and keeps the `dms` CLI (`dms run`); the installer is
  `duxinstall` ([dux-shell README](https://github.com/duxlabs-dev/dux-shell)). DMS's `shell.qml` now has
  `AppId dev.duxlabs.shell`.
- `pacman -Si` output on this machine:
  - `extra/dux-shell 2.0.0-1`: URL `github.com/duxlabs-dev/dux-shell`, `Provides: dms-shell=2.0.0`,
    `Conflicts: dms-shell`, `Replaces: dms-shell dux-shell-bin dms-shell-hyprland dms-shell-niri`.
  - `cachyos/dms-shell 2.0.0-1`: URL still `AvengeMedia/DankMaterialShell`. Same version, older naming.
- The official docs say `sudo pacman -S dux-shell` (or `dux-shell-git` from the AUR)
  ([DMS: Installation](https://duxlabs.dev/docs/dms/installation)). **Use `dux-shell`.**
- Config/plugin path in 2.0: `Paths.config` = `<XDG config>/dms`, so the user plugin dir is `~/.config/dms/plugins`.
  The system plugin dir is `/etc/xdg/quickshell/dms-plugins`
  ([Paths.qml](https://github.com/duxlabs-dev/dux-shell/blob/master/quickshell/Common/Paths.qml),
  [PluginService.qml](https://github.com/duxlabs-dev/dux-shell/blob/master/quickshell/Services/PluginService.qml)).
  Older DMS releases used `~/.config/DankMaterialShell/…` **(from the task brief, not re-verified)**, so check which
  path the laptop's version uses.

## Draft launch steps: nested Hyprland in KWin (recommended)

```sh
# 1. Packages (host). Hyprland also adds an entry to SDDM's session list; harmless.
sudo pacman -S hyprland quickshell dux-shell
#    Optional: inotify-tools for an auto-reload loop.

# 2. A separate nest config so it never collides with a future real ~/.config/hypr
mkdir -p ~/.config/hypr-nest
```

`~/.config/hypr-nest/hyprland.lua` (uses only API shapes from the wiki and example config; **untested**):

```lua
hl.monitor({ output = "", mode = "preferred", position = "auto", scale = "1" })
hl.config({ input = { kb_layout = "de" } })          -- adjust layout
local mod = "ALT"                                     -- KWin keeps SUPER
hl.bind(mod .. " + Return", hl.dsp.exec_cmd("alacritty"))
hl.bind(mod .. " + Q", hl.dsp.window.close())
hl.bind(mod .. " + SHIFT + E", hl.dsp.exec_cmd("hyprctl dispatch 'hl.dsp.exit()'"))
hl.on("hyprland.start", function()
  hl.exec_cmd("dms run")
end)
```

```sh
# 3. Plugin from the repo (directory name = plugin; manifest id is what reload uses)
mkdir -p ~/.config/dms/plugins
ln -s ~/Projects/dms-bongocat/<plugin-dir> ~/.config/dms/plugins/BongoCat

# 4. Launch the nest from a Konsole on KDE. dbus-run-session gives DMS its own
#    session bus (notifications/tray/polkit won't fight plasmashell).
XDG_CURRENT_DESKTOP=Hyprland QT_QPA_PLATFORMTHEME= \
  dbus-run-session Hyprland -c ~/.config/hypr-nest/hyprland.lua
#    (start-hyprland -- -c … is the wrapper the wiki recommends for TTY use)

# 5. Inside the nest (ALT+Return → alacritty), first time only:
#    DMS Settings → Plugins → "Scan for Plugins", enable Bongo Cat.
dms ipc call plugins list
dms ipc call plugins reload <pluginId>   # after each edit
dms restart                              # if reload isn't enough
#    Optional loop, run inside the nest:
#    while inotifywait -r -e close_write ~/Projects/dms-bongocat/<plugin-dir>; do dms ipc call plugins reload <pluginId>; done
```

- The reload and plugin commands come from [DMS: Keybinds & IPC](https://duxlabs.dev/docs/dms/keybinds-ipc) and
  [DMS: Plugin development](https://duxlabs.dev/docs/dms/plugin-development). That page recommends symlinking for
  "live testing" and has no built-in plugin file watcher; it suggests a "Run on Save" editor hook instead.
- DMS turns on Quickshell's `watchFiles` (unless `DMS_DISABLE_HOT_RELOAD=1`)
  ([shell.qml](https://github.com/duxlabs-dev/dux-shell/blob/master/quickshell/shell.qml)). Whether that also covers
  plugin QML outside the shell dir is **unverified**.
- To get rid of it all: `sudo pacman -Rns hyprland quickshell dux-shell`, then
  `rm -r ~/.config/hypr-nest ~/.config/dms ~/.config/hypr` (DMS may create `~/.config/hypr/dms/*.lua`) and
  `~/.local/state/dms` **(paths partly unverified)**.

### Fallback: a dedicated test user on another VT

1. `sudo useradd -m -G video,input hyprtest` (`input` only for this throwaway user), then `sudo passwd hyprtest`.
2. Make the plugin readable for that user: copy it, or `rsync` the plugin dir into `~hyprtest/.config/dms/plugins/`
   from a script.
3. Press Ctrl+Alt+F3, log in as `hyprtest`, run `start-hyprland`. Use a real `~/.config/hypr/hyprland.lua` with
   `hl.on("hyprland.start", … hl.exec_cmd("dms run"))` or `dms setup`, per
   [DMS: Compositors](https://duxlabs.dev/docs/dms/compositors). Alternatively, log out of Plasma and pick
   Hyprland in SDDM as that user.
4. Use this for checks that need the real thing: overlay over a true fullscreen app, multiple monitors, and an evdev
   key reader with all keyboards.

## Caveats / open questions (ticket candidates)

1. **Smoke test: Hyprland 0.57 nested in KWin 6.8.** Does the Wayland backend start (dmabuf feedback on the dual
   GPU), and does a Lua `hyprland.start` + `dms run` come up? If dmabuf init fails, try `AQ_TRACE=1` for the log.
2. **Decide the plugin's key source.** Options: evdev helper, Hyprland `non_consuming` binds → `dms ipc`, or
   something DMS exposes. This decides whether the nest is enough, or whether the test-user/VT path and the `input`
   group are needed. Also decide whether to grant `input` to the dev user at all; it gives every process of that
   user a keylogger-grade capability.
3. **D-Bus clashes without `dbus-run-session`.** DMS ships `BusName=org.freedesktop.Notifications`
   ([dms.service](https://github.com/duxlabs-dev/dux-shell/blob/master/assets/systemd/dms.service)) and has a
   `NotificationServer` and a polkit agent. On the shared session bus these would collide with plasmashell
   **(unverified behaviour: fail vs steal)**. Check that `dbus-run-session` doesn't break DMS features you need
   (MPRIS from host apps won't be visible).
4. **Does `dms ipc` from a host Konsole reach the nested instance?** Quickshell IPC finds instances through
   `$XDG_RUNTIME_DIR`, not the Wayland socket **(unverified)**. Safest is to run it from a terminal inside the nest.
5. **Version parity with the laptop.** Which package (`dms-shell` vs `dux-shell`), which version, which plugin dir
   (`~/.config/dms/plugins` vs old `DankMaterialShell`), and Hyprland ≥ 0.55 with Lua config? On CachyOS, `-Syu`
   will offer to replace `dms-shell` with `dux-shell` (`Replaces:`).
6. **Does Quickshell/DMS hot reload pick up plugin QML edits?** Or is `plugins reload` always needed? Is a
   `Loader`-based plugin's state reset on reload?
7. **Layer choice for the cat.** `Top` (Quickshell default) vs `Overlay` (visible over fullscreen, per
   wayland-bongocat). Can this be tested in the nest at all, given fullscreen is the nest window's size?
8. **Do KWin shortcuts swallow keys meant for the nest?** E.g. Meta. Check whether KWin honours a
   keyboard-shortcuts-inhibit request from the nested window **(unverified)**.

## Sources

- Hyprland wiki: [Master tutorial](https://wiki.hypr.land/Getting-Started/Master-Tutorial/) · [Installation (VM)](https://wiki.hypr.land/Getting-Started/Installation/) · [Contributing & Debugging](https://wiki.hypr.land/Contributing-and-Debugging/) · [Plugin dev getting started](https://wiki.hypr.land/Plugins/Development/Getting-Started/) · [Configuring/Core](https://wiki.hypr.land/Configuring/Core/) · [Autostart](https://wiki.hypr.land/Configuring/Core/Autostart/) · [Environment variables](https://wiki.hypr.land/Configuring/Core/Environment-variables/) · [Using hyprctl](https://wiki.hypr.land/Configuring/Core/Advanced-configuration/Using-hyprctl/) · [IPC](https://wiki.hypr.land/IPC/) · [Bind flags](https://wiki.hypr.land/Configuring/Core/Binds/Flags/) · [FAQ](https://wiki.hypr.land/FAQ/) · wiki source: https://github.com/hyprwm/hyprland-wiki (content/, main branch)
- Hyprland source: https://github.com/hyprwm/Hyprland/blob/main/src/Compositor.cpp · https://github.com/hyprwm/Hyprland/blob/main/example/hyprland.lua
- Aquamarine: https://github.com/hyprwm/aquamarine/blob/main/README.md · https://github.com/hyprwm/aquamarine/blob/main/docs/env.md · https://github.com/hyprwm/aquamarine/blob/main/src/backend/Wayland.cpp
- DMS docs: [Installation](https://duxlabs.dev/docs/dms/installation) · [Compositors](https://duxlabs.dev/docs/dms/compositors) · [Layers](https://duxlabs.dev/docs/dms/layers) · [Keybinds & IPC](https://duxlabs.dev/docs/dms/keybinds-ipc) · [Plugin development](https://duxlabs.dev/docs/dms/plugin-development) · [Process management](https://duxlabs.dev/docs/dms/cli-process-management)
- DMS source: https://github.com/duxlabs-dev/dux-shell (README, `quickshell/shell.qml`, `quickshell/Common/Paths.qml`, `quickshell/Services/PluginService.qml`, `assets/systemd/dms.service`); redirect from https://github.com/AvengeMedia/DankMaterialShell
- Quickshell: https://quickshell.org/docs/v0.3.0/types/Quickshell.Wayland/WlrLayershell/ · https://quickshell.outfoxxed.me/docs/types/quickshell/quickshell (`watchFiles`, `reload()`)
- QEMU: https://www.qemu.org/docs/master/system/devices/virtio/virtio-gpu.html
- distrobox: https://distrobox.it/ · https://distrobox.it/useful_tips/
- wayland-bongocat: https://github.com/saatvik333/wayland-bongocat
- Local: `pacman -Si dux-shell dms-shell hyprland quickshell` (2026-11-06)
