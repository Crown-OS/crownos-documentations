# crowndictator

**Status: Early** · Rust · default branch `main` ·
[repo](https://github.com/Crown-OS/crowndictator)

Push-to-talk voice dictation for Wayland. Hold the shortcut, speak, release: the
recording is transcribed **locally** and typed into the focused window. A small
waveform pill sits at the bottom of the screen while it listens and thinks.

Delivered in a single commit, and internally the most coherent and best-commented
application in the project. It is also the cleanest example of the CrownOS
config-as-IPC pattern.

---

## How it works

```
hold hotkey ──► evdev reads /dev/input/event*  (bypasses the compositor)
                          │
                          ▼
                  cpal captures 16 kHz mono f32
                          │
                          ▼
      ONNX Runtime: nemo128 mel → conformer encoder → greedy TDT decode
                    → SentencePiece detokenise
                          │
                          ▼
      wtype  ──fallback──►  ydotool  ──fallback──►  wl-copy + notify-send
```

Model: **NVIDIA Parakeet TDT 0.6B v2**, run through ONNX Runtime with CUDA when
available.

The ASR engine preloads on demand and **drops the model after 300 s idle** to
free VRAM, so the first transcription after a pause is slower.

---

## Prerequisites

The heaviest in the project. See
[Prerequisites](../10-getting-started/prerequisites.md#crowndictator) for
packages. Beyond those:

- **`input` group membership** — the hotkey is detected by reading
  `/dev/input/event*` directly. `sudo usermod -aG input "$USER"`, then log out
  and back in.
- **ONNX Runtime with CUDA.** `ort` is pinned exactly (`= "2.0.0-rc.12"`) and the
  `cuda` feature is **not optional**, so it is a hard build dependency even on a
  machine with no NVIDIA hardware. The runtime falls back to CPU.
- **A large first-run download** from Hugging Face
  (`istupakov/parakeet-tdt-0.6b-v2-onnx`): roughly **700 MB** for the int8 CPU
  model, **2.5 GB** for fp32 on GPU.
- **Injection tools**: `wtype`, `ydotool`, or `wl-clipboard` plus `libnotify`.
- A `wlr-layer-shell` compositor for the overlay.

Both `crownshell` and `crownos-config` are **path** dependencies, so it needs the
[flat sibling layout](../10-getting-started/workspace-setup.md).

---

## Build and run

```bash
cd crowndictator

cargo run -- --demo               # cycle overlay states with fake audio.
                                  # No model download, no microphone,
                                  # no input group needed.
cargo run                         # the daemon
cargo run -- --transcribe f.wav   # one-shot, 16 kHz wav
cargo run -- --cpu                # skip CUDA
```

**`--demo` is the contributor-friendly path.** Use it for any work on the overlay
UI — it exercises every visual state without downloading a model or touching
`/dev/input`.

---

## Configuration

Reads and live-follows `~/.config/crownos/input.ron` through `crownos-config`.
The manifest comment explains why the dependency exists:

> `~/.config/crownos/input.ron` — the switch, the microphone and the shortcut,
> all written by the settings panel's Input page and followed live from here.

| Field | Default | Effect |
|---|---|---|
| `dictation_enabled` | `true` | Off keeps the daemon resident but releases the keyboard grab |
| `dictation_microphone` | `None` | `None` follows the system default device |
| `dictation_hotkey` | `"Super+Space"` | Held, not struck — a modifier-only chord is reasonable here |
| `dictation_gpu` | `true` | Same switch as `--cpu`, from the other direction |

It subscribes to the **whole section** rather than four individual keys, so the
controller *"sees one consistent snapshot instead of four independent edits"*.

`src/settings.rs` is 40 lines and is the model to copy when bringing another
component onto the config convention.

---

## Source layout

| File | Lines | Role |
|---|---|---|
| `main.rs` | 113 | CLI, creates the `Layer::Overlay` bottom overlay with an **empty input region** so it is click-through |
| `controller.rs` | 263 | The orchestrator |
| `hotkey.rs` | 599 | Global chord detection over evdev |
| `audio.rs` | 192 | cpal capture, 16 kHz mono f32 |
| `asr/mod.rs` | 350 | The ONNX pipeline and the idle-TTL engine thread |
| `asr/download.rs` | — | Hugging Face model fetch |
| `inject.rs` | 51 | The `wtype` → `ydotool` → `wl-copy` fallback chain |
| `settings.rs` | 40 | `load()` and `#[must_use] watch()` |
| `state.rs` | — | `Shared = Arc<Mutex<VizState>>`, `Phase::{Hidden,Listening,Thinking,Success,Error}` |
| `waveform.rs` | 266 | The `crownshell` `SurfaceHandler` |
| `wav.rs` | — | 16 kHz wav reader for `--transcribe` |

### The single-channel design

`controller.rs`'s module header is worth reading in full. The core idea: hotkey
edges *and* settings changes are both `Event`s on one channel, which is what
makes "the user switched dictation off while holding the chord" an ordinary
sequence of messages rather than a race.

### Deliberate compositor bypasses

Both are so it works under any compositor, not only `crownpositor`:

- **Hotkey** — reads `/dev/input/event*` directly rather than asking for a global
  shortcut. Modifier match is exact; `/dev/input` is rescanned every 4 s for
  hotplug.
- **Injection** — shells out rather than using a Wayland input method.

The trade-off is portability at the cost of elevated device access and external
binaries.

---

## Known limitations

- **No tests**, no examples, no benches. There are zero TODO or FIXME markers.
- **Pulls `crownos-config` with default features**, which includes `xilem` — a
  headless daemon dragging in a GUI toolkit. `default-features = false` is likely
  correct.
- **CUDA is not optional** at build time.
- The first run needs network access and multiple gigabytes of disk.

---

## License

`Cargo.toml` declares `license = "MIT"`, but **there is no LICENSE file** in the
repository. See [Project status](../00-overview/project-status.md#licensing).
