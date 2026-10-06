# dota-minimap-mirror

Mirror a cropped region of your main display (e.g. the Dota 2 minimap) onto a
second physical screen in real time, automatically turning that screen on
when the game launches and off when it closes.

Built for **Linux + KDE Plasma (Wayland)**, specifically for setups like:

- A 3440x1440 ultrawide running the game natively (e.g. via Steam/Proton on
  Bazzite/SteamOS-like systems).
- A small secondary panel (e.g. a 720x720 square display) plugged into
  another output, used purely to show the minimap.

There is no Windows dependency and no DWM thumbnail API involved — this is a
from-scratch implementation for Wayland compositors that don't expose
`wlr-screencopy` or `ext-image-copy-capture-v1` (which includes KWin).

## How it works

```
Steam launch options
        │
        ▼
dota-minimap-wrapper          (enables the output, sets brightness/position)
        │
        ├── starts dota-minimap-mirror (background)
        │        │
        │        ▼
        │   xdg-desktop-portal ScreenCast  →  PipeWire node
        │        │
        │        ▼
        │   GStreamer: pipewiresrc → videocrop → waylandsink
        │        │                                  │
        │        ▼                                  ▼
        │   crop region reloaded live       fullscreened on target output
        │   from config file
        │
        └── runs the actual game (%command%)
                 │
                 ▼
        on exit: kill mirror, disable the output (kscreen-doctor)
```

Because KWin doesn't support the lightweight wlroots screencopy protocols,
capture goes through **xdg-desktop-portal's `ScreenCast` interface**, which
hands back a PipeWire video stream. GStreamer crops that stream to the
configured rectangle and renders it fullscreen on the target output via
`waylandsink`. When DMABuf buffers are available the crop/scale happens
entirely on the GPU — no frame is copied to the CPU.

The one-time screen-sharing permission prompt is answered once; a **restore
token** is saved so every subsequent launch is silent.

## Requirements

- KDE Plasma on Wayland (tested on Plasma 6.7, Bazzite/Kinoite).
- `xdg-desktop-portal-kde` and PipeWire (standard on Plasma).
- GStreamer with `pipewiresrc`, `videocrop`, `waylandsink` (part of the
  standard `gstreamer1-plugins-*` set on most distros, including Bazzite).
- Python 3 with `PyGObject` (`python3-gobject` / `python-gobject`), providing
  `gi.repository.Gst` and `gi.repository.Gio`.
- `kscreen-doctor` (ships with Plasma) for enabling/disabling/positioning the
  secondary output and setting its brightness.

Nothing here needs to be installed on an immutable base if the above
packages are already part of your Plasma desktop — they are on Bazzite.

## Installation

```bash
git clone https://github.com/xeroxis-xs/dota-minimap-mirror.git
cd dota-minimap-mirror

mkdir -p ~/.local/bin ~/.config/dota-minimap
cp bin/dota-minimap-mirror ~/.local/bin/
cp bin/dota-minimap-wrapper ~/.local/bin/
chmod +x ~/.local/bin/dota-minimap-mirror ~/.local/bin/dota-minimap-wrapper

cp config/region.conf.example ~/.config/dota-minimap/region.conf
```

Edit `~/.config/dota-minimap/region.conf` to match your layout (see below).

### Steam launch options

In Steam, open the game's **Properties → Launch Options** and set:

```
~/.local/bin/dota-minimap-wrapper %command%
```

Launching the game from Steam now:

1. Enables the target output and moves it into position.
2. Starts the mirror (prompting for screen-share permission once, ever).
3. Runs the game normally.
4. On exit, stops the mirror and disables the output again.

### First run

The first launch triggers a KDE "Share screen" dialog — pick your main
display (the one the game runs on) and tick **"Remember this decision"**.
After that it's silent on every future launch.

## Configuration

`~/.config/dota-minimap/region.conf`:

```ini
# Region of the source display to mirror, in pixels, relative to the
# top-left of the captured output.
CROP_X=0
CROP_Y=1070
CROP_W=370
CROP_H=370

# Wayland output name to show the mirror on (see `kscreen-doctor -o`).
TARGET_OUTPUT=HDMI-A-1
```

This file is watched live — while the mirror is running, edits are picked up
immediately, so you can nudge `CROP_X/Y/W/H` to pixel-perfect alignment
without restarting anything.

Environment variables for `dota-minimap-wrapper` itself (set them inline in
the Steam launch options, before the script path):

| Variable             | Default        | Purpose                                   |
|----------------------|----------------|--------------------------------------------|
| `MINIMAP_OUTPUT`     | `HDMI-A-1`     | Output to enable/disable                   |
| `MINIMAP_POSITION`   | `1377,1440`    | Output position passed to `kscreen-doctor` |
| `MINIMAP_BRIGHTNESS` | *(unset)*      | 0-100, sets panel brightness on launch     |

Example:

```
MINIMAP_BRIGHTNESS=60 ~/.local/bin/dota-minimap-wrapper %command%
```

## Finding your crop rectangle

1. Take a screenshot of your main display while in-game (e.g. `spectacle -f`).
2. Crop/zoom into the minimap corner to find its pixel bounds.
3. Set `CROP_X`, `CROP_Y`, `CROP_W`, `CROP_H` accordingly in `region.conf`.

For a standard Dota 2 HUD on a 3440x1440 display with the minimap in the
bottom-left corner, a region close to `0,1070,370,370` is a good starting
point — adjust to taste since HUD scale settings shift it slightly.

## Running without Steam

You can run the mirror standalone, e.g. via `systemd --user`:

```bash
systemd-run --user --unit=dota-minimap --description="Dota minimap mirror" \
  --setenv=WAYLAND_DISPLAY="$WAYLAND_DISPLAY" \
  --setenv=XDG_RUNTIME_DIR="$XDG_RUNTIME_DIR" \
  ~/.local/bin/dota-minimap-mirror

systemctl --user status dota-minimap
journalctl --user -u dota-minimap -f
systemctl --user stop dota-minimap
```

Remember to enable the output first with `kscreen-doctor output.<NAME>.enable`.

## Adjusting brightness manually

The target panel's brightness can be changed at any time, independent of the
mirror:

```bash
kscreen-doctor output.HDMI-A-1.brightness.50   # 0-100
```

## Known limitations

- **No zero-cost path on Wayland.** Capturing the source output via PipeWire
  prevents KWin from giving the game a direct scanout path, so there's a
  small performance cost while the mirror runs. A true hardware-level region
  mirror (near-zero cost) is only possible under X11 with overlapping
  `xrandr` outputs, which requires giving up the Wayland session.
- **Requires KDE's portal implementation.** This relies on
  `xdg-desktop-portal-kde`'s `ScreenCast` interface and `kde_output_management_v2`.
  Other compositors (GNOME, wlroots-based) would need a different capture
  path (e.g. `wl-mirror`, `grim`/`slurp`-based tools) and different output
  control commands than `kscreen-doctor`.
- **Portal may capture the full workspace**, not a single output, depending
  on your compositor version. The crop coordinates are relative to that
  captured image, not necessarily a single monitor's own coordinate space —
  verify alignment with a screenshot if your layout differs from the example.

## License

MIT — see [LICENSE](LICENSE).
