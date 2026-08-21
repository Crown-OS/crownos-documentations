# Configuration schema

Every section, field, type and default in `crownos-config`.

Files live at `<config-dir>/<section>.ron`, where the config directory is
`$CROWN_CONFIG_DIR` if set, otherwise `~/.config/crownos`. Format is
[RON](https://github.com/ron-rs/ron) with implicit `Some`, so optional fields are
written `network: "home"` rather than `network: Some("home")`.

Omitted fields fall back to their defaults. A file that fails to parse is left
alone and the defaults are used — see
[Config as IPC](../20-architecture/config-as-ipc.md).

---

## Sections at a glance

| File | Struct | Consumer | Status |
|---|---|---|---|
| `appearance.ron` | `Appearance` | crownpositor | Partly consumed |
| `bluetooth.ron` | `Bluetooth` | — | **No reader** |
| `compositor.ron` | `Compositor` | crownpositor | Consumed, live |
| `display.ron` | `Display` | crownpositor | Consumed |
| `input.ron` | `Input` | crowndictator | Consumed, live |
| `keybinds.ron` | `Keybinds` | (compositor, intended) | **No reader** |
| `notifications.ron` | `Notifications` | (crownotify, intended) | **No reader** |
| `power.ron` | `Power` | — | **No reader** |
| `sound.ron` | `Sound` | — | **No reader** |
| `wifi.ron` | `Wifi` | — | **No reader** |

Note the file is `notifications.ron` (plural) while the module is
`schema/notification.rs` (singular). The **file name is what matters**.

---

## appearance.ron

How the desktop looks. Consumed by `crownpositor`.

| Field | Type | Default | Notes |
|---|---|---|---|
| `dark_mode` | `bool` | `true` | |
| `accent` | `AccentColor` | `Purple` | `Purple` · `Blue` · `Green` · `Orange` · `Pink` |
| `transparency` | `f64` | `0.0` | |
| `wallpaper` | `String` | `""` | |
| `bar_height` | `u32` | `32` | **`crownbar` ignores this and hardcodes 40** |
| `gaps_inner` | `u16` | `8` | Pixels between tiled windows |
| `gaps_outer` | `u16` | `8` | Pixels between the tiled area and the output edge |
| `border_width` | `u16` | `2` | |
| `border_radius` | `u16` | `8` | |
| `animations` | `AnimationProfile` | `Standard` | `None` · `Snappy` · `Standard` · `Smooth` |

`AnimationProfile::None` snaps straight to the target — no springs stepped, no
redraws scheduled.

Window gaps, borders and animation live here rather than in `compositor` because
changing them is changing the theme, not the tiling.

```ron
(
    dark_mode: true,
    accent: Blue,
    wallpaper: "/home/me/pictures/wall.png",
    gaps_inner: 12,
    animations: Smooth,
)
```

---

## compositor.ron

Window management. Consumed by `crownpositor`, **live** — a rebind takes effect
without a restart.

| Field | Type | Default |
|---|---|---|
| `layout` | `LayoutMode` | `MasterStack` |
| `focus_follows_mouse` | `bool` | `false` |
| `keybinds` | `Vec<Binding>` | `[]` |
| `window_rules` | `Vec<WindowRule>` | `[]` |
| `outputs` | `Vec<OutputSetting>` | `[]` |

`LayoutMode`:

| Value | Behaviour |
|---|---|
| `MasterStack` | dwm/xmonad — one master area plus a stack |
| `ScrollingColumns` | niri/PaperWM — an infinite horizontal ribbon of columns |
| `Floating` | Nothing is tiled; every window keeps its own rect |

### Binding

```ron
(keys: "Super+Q", action: "close-window")
```

Both fields are strings so the file stays hand-editable; a bad row is logged and
skipped rather than failing the load.

**An empty `keybinds` list means "use the built-in defaults", not "nothing
bound"** — otherwise a fresh install would have no way to quit. To genuinely bind
nothing, write one row with `keys: "None"`.

Accepted actions: [Keybindings](keybindings.md#the-action-vocabulary).

### WindowRule

Matched at a window's first buffer commit — the earliest point at which `app_id`,
`title` and size hints exist. All fields are optional.

| Field | Type | Notes |
|---|---|---|
| `app_id` | `String` | Regex, **unanchored** — `"blender"` matches `"org.blender.Blender"` |
| `title` | `String` | Regex |
| `floating` | `bool` | |
| `fullscreen` | `bool` | |
| `maximized` | `bool` | |
| `workspace` | `u16` | Zero-based index on the target output |
| `output` | `String` | Connector name or `"MAKE MODEL SERIAL"` |
| `focus` | `bool` | `false` opens the window without stealing focus |
| `opacity` | `f32` | |
| `corner_radius` | `u16` | |

### OutputSetting

| Field | Type | Notes |
|---|---|---|
| `name` | `String` | Connector name (`"eDP-1"`) or `"MAKE MODEL SERIAL"` |
| `enabled` | `bool` | |
| `mode` | `String` | `"2560x1440@144.000"` |
| `scale` | `f64` | |
| `transform` | `OutputTransform` | `Normal` · `R90` · `R180` · `R270` · `Flipped` · `Flipped90` · `Flipped180` · `Flipped270` |
| `position` | `(i32, i32)` | |
| `vrr` | `bool` | |
| `layout` | `LayoutMode` | Overrides the global default for workspaces on this output |

### Example

This exact shape is covered by a test in the crate, so it is guaranteed to parse:

```ron
(
    layout: ScrollingColumns,
    keybinds: [
        (keys: "Super+Q", action: "close-window"),
    ],
    window_rules: [
        (app_id: "Nautilus", floating: true),
        (title: "^(Open|Save)", floating: true, focus: false),
    ],
    outputs: [
        (name: "eDP-1", scale: 2.0, position: (0, 0)),
    ],
)
```

> **Schema skew.** `crownpositor` reads `config.compositor.startup` to launch the
> bar, wallpaper and notification daemon, but there is **no `startup` field** in
> this schema. The two checkouts do not compile together as-is. See
> [Project status](../00-overview/project-status.md).

---

## display.ron

| Field | Type | Default | Notes |
|---|---|---|---|
| `brightness` | `f64` | `80.0` | |
| `night_light` | `bool` | `false` | |
| `night_light_warmth` | `f64` | `50.0` | |
| `scale` | `DisplayScale` | `S100` | `S100` (1.0) · `S125` (1.25) · `S150` (1.5) · `S200` (2.0) |

---

## input.ron

Text input — currently dictation only. Consumed by `crowndictator`, **live**.

| Field | Type | Default | Notes |
|---|---|---|---|
| `dictation_enabled` | `bool` | `true` | Off keeps the daemon resident but releases its keyboard grab |
| `dictation_microphone` | `Option<String>` | `None` | `None` follows the system default device |
| `dictation_hotkey` | `Keybind` | `"Super+Space"` | Held, not struck |
| `dictation_gpu` | `bool` | `true` | The same switch as `--cpu`, from the other direction |

Every field is prefixed because the section is the settings panel's *Input* page
and is expected to gain key-repeat or input-method settings later — those should
not force a rename of what is already there.

`dictation_microphone` naming a device that no longer exists falls back to the
default rather than failing to record.

```ron
(
    dictation_enabled: true,
    dictation_microphone: "Blue Yeti Analog Stereo",
    dictation_hotkey: "Super+Alt+D",
    dictation_gpu: false,
)
```

---

## keybinds.ron

Desktop-wide shortcuts — the ones a compositor grabs globally.

| Field | Type | Default | Notes |
|---|---|---|---|
| `launcher` | `Keybind` | `"Super+Ctrl"` | Shows and hides the launchpad |

One shortcut toggles both directions rather than two, because "the launchpad is a
place you go and come back from". The default is modifier-only so it cannot
collide with an application's own bindings.

> **Nothing reads this section.** `crownpositor`'s `Config::sections()` returns
> only `compositor`, `appearance` and `display`, and
> [`crownlauncher`](../30-components/crownlauncher.md) is a stub. The shortcut is
> defined and connected at neither end.

---

## notifications.ron

| Field | Type | Default |
|---|---|---|
| `enabled` | `bool` | `true` |
| `do_not_disturb` | `bool` | `false` |
| `show_previews` | `bool` | `true` |

> **`crownotify` ignores this entirely**, including Do-Not-Disturb. It has a
> private `do_not_disturb` field and a `toggle_dnd()` that nothing calls.

---

## power.ron

| Field | Type | Default | Notes |
|---|---|---|---|
| `screen_off_minutes` | `u32` | `10` | |
| `sleep_minutes` | `u32` | `30` | |
| `power_profile` | `PowerProfile` | `Balanced` | `PowerSaver` · `Balanced` · `Performance` |

**No reader.**

---

## sound.ron

| Field | Type | Default |
|---|---|---|
| `output_volume` | `f64` | `50.0` |
| `input_volume` | `f64` | `50.0` |
| `muted` | `bool` | `false` |
| `output_device` | `Option<String>` | `None` |

**No reader.** `crownbar`'s volume widget shells out to `wpctl`/`pactl` instead.

---

## wifi.ron

| Field | Type | Default |
|---|---|---|
| `enabled` | `bool` | `true` |
| `network` | `Option<String>` | `None` |

**No reader.** `crownbar`'s wifi widget reads `/sys` and `/proc` instead.

---

## bluetooth.ron

| Field | Type | Default |
|---|---|---|
| `enabled` | `bool` | `false` |

**No reader.** `crownbar`'s bluetooth widget reads `/sys/class/rfkill/` instead.

---

## The Keybind type

Used by `input.dictation_hotkey` and `keybinds.launcher`. Distinct from
`compositor.keybinds`, whose rows are plain strings because the compositor's
action vocabulary is its own.

**Serialised as a Display string**, so files contain
`dictation_hotkey: "Super+Space"` rather than a nested struct.

Structure: `Mods { meta, ctrl, alt, shift }` plus `Option<KeyCode>`.

**Modifier aliases**, all accepted and normalised:

| Canonical | Also accepted |
|---|---|
| `Super` | `Meta`, `Cmd`, `Win` |
| `Ctrl` | `Control` |
| `Alt` | `Option` |
| `Shift` | — |

Order does not matter: `"Ctrl+Super"` parses to the same value as
`"Super+Ctrl"`.

**Key names** are W3C `KeyboardEvent.code` values — `KeyA`, `Space`,
`ArrowLeft`, `Digit1`, `F5`.

**Modifier-only chords are valid** and are fired on the release edge. That is why
`Super+Ctrl` works as a launcher toggle and `Super+Space` works as push-to-talk.

**To unbind**, use the literal string `"None"`.

---

## Adding a section

1. Add `src/schema/<name>.rs` using the `section!` macro.
2. Declare it in `src/schema/mod.rs`.
3. Write a module doc that names **who the consumer is** — every existing section
   does, and it is the only place that contract is recorded.
4. Add a test that parses the RON shape you documented, using
   `crate::parser::options()` so it goes through the same path as `load()`.
   `schema/compositor.rs` is the model.
5. Add the section to this page.
