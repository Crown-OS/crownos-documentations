# Keybindings

The compositor's default bindings, the gesture bindings, and the action
vocabulary you can bind to.

Keyboard chords and trackpad gestures resolve to the **same** `Action` enum and
go down one dispatch path. From the source:

> No second enum for gestures — that is how "swipe left" and "Super+Tab" drift
> apart.

---

## Default keyboard bindings

32 bindings, defined in
`crownpositor/compositor/src/input/shortcuts/bindings.rs`.

### Session

| Chord | Action |
|---|---|
| `Super+Shift+E` | `quit` |
| `Super+Return` | `spawn foot` |
| `Super+Shift+C` | `reload-config` |
| `Super+Q` | `close-window` |

### Focus

| Chord | Action |
|---|---|
| `Super+H` | `focus left` |
| `Super+J` | `focus down` |
| `Super+K` | `focus up` |
| `Super+L` | `focus right` |

### Moving windows

| Chord | Action |
|---|---|
| `Super+Shift+H` | `move left` |
| `Super+Shift+J` | `move down` |
| `Super+Shift+K` | `move up` |
| `Super+Shift+L` | `move right` |

### Workspaces

| Chord | Action |
|---|---|
| `Super+Tab` | `workspace +1` |
| `Super+Shift+Tab` | `workspace -1` |
| `Super+1` … `Super+4` | `workspace 0` … `workspace 3` |
| `Super+Shift+1` … `Super+Shift+4` | `move-to-workspace 0 follow` … `3 follow` |

Note the display numbers are 1-based while the workspace indices are 0-based.

### Window state

| Chord | Action |
|---|---|
| `Super+V` | `toggle-float` |
| `Super+F` | `toggle-fullscreen` |
| `Super+M` | `toggle-maximize` |

### Layout

| Chord | Action |
|---|---|
| `Super+Space` | `cycle-layout` |
| `Super+Shift+Space` | `toggle-layout-mode` |
| `Super+Ctrl+L` | `resize-split 0.05` |
| `Super+Ctrl+H` | `resize-split -0.05` |
| `Super+P` | `promote` |
| `Super+R` | `cycle-size` |
| `Super+Shift+R` | `reset-size` |

> **`Super+Space` collides with `crowndictator`.** The compositor binds it to
> `cycle-layout`; `input.dictation_hotkey` defaults to `"Super+Space"` for
> push-to-talk. Since `crowndictator` reads `/dev/input` directly rather than
> going through the compositor, **both fire**. Rebind one of them.

### Virtual terminals

`Ctrl+Alt+F1` through `Ctrl+Alt+F12` are wired unconditionally in the keyboard
filter — they are not in the bindings table. The `switch-vt <n>` action exists so
a config can put a VT on a different chord.

---

## Default trackpad gestures

| Gesture | Action |
|---|---|
| 3-finger swipe left → right | `workspace -1` |
| 3-finger swipe right → left | `workspace +1` |
| 4-finger swipe bottom → top | `open-workspace-view` |
| 4-finger swipe top → bottom | `close-workspace-view` |

> **The four-finger gestures do nothing.** `OpenWorkspaceView` and
> `CloseWorkspaceView` log `"action is not implemented yet"` —
> `shell/windows_view/` and `shell/workspaces_view/` are zero-byte files. See
> [Project status](../00-overview/project-status.md).

---

## Rebinding

Bindings live in `compositor.ron`:

```ron
(
    keybinds: [
        (keys: "Super+Q", action: "close-window"),
        (keys: "Super+Return", action: "spawn kitty"),
        (keys: "Super+E", action: "spawn nautilus"),
    ],
)
```

Rules:

- **An empty `keybinds` list means "use the built-in defaults"**, not "nothing
  bound" — otherwise a fresh install would have no way to quit.
- **To genuinely bind nothing**, write one row with `keys: "None"`.
- **A malformed row is logged and skipped**, not fatal. The rest of the file
  still loads.
- **Changes apply live.** `crownpositor` watches `compositor.ron`, so a rebind
  takes effect without a restart.

### Chord syntax

`Modifier+Modifier+Key`, with modifiers in any order.

| Canonical | Also accepted |
|---|---|
| `Super` | `Meta`, `Cmd`, `Win` |
| `Ctrl` | `Control` |
| `Alt` | `Option` |
| `Shift` | — |

**Modifier-only chords are valid** — write just `"Super"`. They fire on the
release edge, which is what makes them usable for hold-style shortcuts and
toggles.

---

## The action vocabulary

Actions parse from strings, whitespace-separated: `<name> [args…]`.

### No arguments

| Action | Aliases | Effect |
|---|---|---|
| `none` | | Binds nothing |
| `quit` | `exit` | Ends the session |
| `reload-config` | | Re-reads configuration |
| `close-window` | `close` | Closes the focused window |
| `toggle-float` | `toggle-floating` | Floats or tiles the focused window |
| `toggle-fullscreen` | | |
| `toggle-maximize` | | |
| `toggle-layout-mode` | | Flips the compositor-wide default; per-workspace overrides stay |
| `cycle-layout` | | Cycles this workspace's override: none → master → scrolling → none |
| `open-workspace-view` | | **Not implemented** |
| `close-workspace-view` | | **Not implemented** |
| `promote` | `demote` | Into or out of the master area; full width in a scrolling layout |
| `cycle-size` | | Cycles the focused window through the layout's preset sizes |
| `reset-size` | | |

### With a direction

Directions: `left` · `right` · `up` · `down`.

| Action | Aliases | Effect |
|---|---|---|
| `focus <direction>` | | Moves focus |
| `move <direction>` | `move-window` | Moves the focused window |
| `focus-output <direction>` | | Moves focus to another output |
| `move-to-output <direction>` | | Sends the window to another output |
| `move-workspace-to-output <direction>` | | **Not implemented** |

### With a workspace reference

A workspace reference is one of:

| Form | Meaning |
|---|---|
| `0`, `1`, `2`… | Absolute, zero-based index |
| `+1`, `-2` | Relative — a leading sign means relative |
| `prev`, `previous` | The previously focused workspace |

| Action | Effect |
|---|---|
| `workspace <ref>` | Switches workspace |
| `move-to-workspace <ref> [follow]` | Sends the focused window there. **`follow` is opt-in** — being yanked along with the window is surprising. |

### Other arguments

| Action | Argument | Effect |
|---|---|---|
| `spawn <program> [args…]` | argv, whitespace-split | `spawn foot -e nvim` runs `foot -e nvim`. At least one token is required. |
| `switch-vt <n>` | integer | Switches virtual terminal |
| `set-layout <layout>` | `master-stack`\|`master`, `scrolling-columns`\|`scrolling`, `floating` | Sets this workspace's layout |
| `resize-split <fraction>` | float | Grows or shrinks the layout's primary split. Negative shrinks. |

### Parse errors

A bad action produces a specific message rather than a silent skip:

| Error | Example message |
|---|---|
| Empty | `empty action` |
| Unknown name | ``unknown action `flurb` `` |
| Bad argument | ``invalid direction `sideways` `` |
| Missing argument | ``` `focus` needs a direction ``` |

---

## Desktop-wide shortcuts

Separate from the compositor's own table, `keybinds.ron` holds shortcuts owned by
the desktop rather than by the window manager:

| Field | Default | Effect |
|---|---|---|
| `launcher` | `"Super+Ctrl"` | Shows and hides the launchpad |

Modifier-only by default so it cannot collide with an application's own bindings.

> **Nothing reads this section.** `crownpositor`'s `Config::sections()` returns
> only `compositor`, `appearance` and `display`, and `crownlauncher` is a stub.

---

## Dictation

`crowndictator`'s push-to-talk chord is in `input.ron`, not here, because it is
meaningless with dictation switched off:

| Field | Default |
|---|---|
| `dictation_hotkey` | `"Super+Space"` |

It is **held**, not struck — which is why a modifier-only chord is reasonable
there. `crowndictator` detects it by reading `/dev/input/event*` directly, so it
works under any compositor and is not affected by the compositor's binding table.
See the collision note above.

---

## See also

- [Configuration schema](config-schema.md#compositorron)
- [crownpositor](../30-components/crownpositor.md)
