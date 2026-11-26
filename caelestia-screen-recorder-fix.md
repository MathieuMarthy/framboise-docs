# Caelestia Screen Recorder: Recording Dies After ~3 Seconds

## 1. Overview

- **Problem**: Starting a recording from the Caelestia shell UI (Utilities panel → *Fullscreen* / *Region* buttons) shows "Running…", then stops on its own after ~3 seconds. Nothing is saved to `~/Videos/Recordings`, and `~/.local/state/caelestia/record/recording.mp4` is left as an 88-byte stub.
- **Not affected**: Running `caelestia record` from a terminal or through the Hyprland keybinds (`CTRL+ALT+R`, `SUPER+ALT+R`, `SUPER+SHIFT+ALT+R`) worked.
- **Components**: `caelestia-cli` 1.1.3 (`caelestia/subcommands/record.py`), `caelestia-shell` 2.5.0 (`services/Recorder.qml`), `gpu-screen-recorder` 6.1.7.

## 2. Context / Root Cause

`Recorder.qml` runs `caelestia record` through a Quickshell `Process`. `record.py` then spawns `gpu-screen-recorder` with `subprocess.Popen(..., start_new_session=True)` and **does not redirect its stdout/stderr**, so the recorder inherits the Quickshell `Process` pipe.

`record.py` returns after `proc.wait(1)`. Quickshell then closes the read end of the pipe. `gpu-screen-recorder` writes a `gsr info: update fps: …` line to stderr every second, the write hits a closed pipe, and the process dies from **SIGPIPE** before it has written the MP4 trailer.

Minimal reproduction (no fix applied):

```bash
(caelestia record 2>&1 | sleep 1); sleep 4; pgrep -a gpu-screen-recorder || echo "recorder dead"
```

There are no coredumps and no kernel messages, because SIGPIPE is a normal signal death.

## 3. Solution Implemented

`gpu-screen-recorder -v no` turns off the periodic stderr output. Caelestia passes `record.extraArgs` straight to the recorder, so the fix only needs config and survives package updates.

`~/.config/caelestia/cli.json`:

```json
"record": {
    "extraArgs": ["-v", "no"]
}
```

No shell restart is needed, because `cli.json` is read on every `caelestia record` call.

## 4. Verification & Status

- Tested a closed pipe with `-v no`: the recorder stayed alive, and stopping it with SIGINT produced a valid MP4 (`ffprobe` duration ~4.8 s).
- The user tested the Utilities panel *Fullscreen* button: the recording was saved to `~/Videos/Recordings/` and is valid.

```bash
ffprobe -v error -show_entries format=duration -of csv=p=0 ~/Videos/Recordings/<file>.mp4
```

## 5. Maintenance / Handy Commands

```fish
# Is a recording running?
pgrep -a gpu-screen-recorder

# Shell logs (look for Recorder / Process errors)
qs log -c caelestia

# Validate config
python3 -c "import json;print(json.load(open('$HOME/.config/caelestia/cli.json'))['record'])"
```

- **Pitfall when debugging**: `caelestia record` stops recordings with `pkill -f gpu-screen-recorder`. It kills **any** process whose command line contains that string, including a debugging shell that ran a command mentioning it.
- If an upstream `record.py` update redirects the recorder's output (`stdout/stderr=DEVNULL`), this workaround can be removed. It is harmless either way.
- If a future `gpu-screen-recorder` still writes to stderr with `-v no` (for example warnings during audio recording), the symptom will come back. In that case the proper fix is a patch in `record.py` that redirects the `Popen` output.
