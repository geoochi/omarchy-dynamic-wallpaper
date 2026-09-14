# omarchy-dynamic-wallpaper

A time-of-day wallpaper for Omarchy: the desktop shows the frame of a 24-hour
timelapse that matches the current minute, so it follows wall-clock time instead
of a fixed image.

This repository holds the wallpaper *scheme*. The render it reads lives in
`indoor/` and is gitignored.

## How it works

The source is a single video: 1440 frames, 1 fps, one frame per minute of the
day, and the frame for the current minute is simply

```
index = hour * 60 + minute
```

That video was encoded with **one keyframe, its first frame**, so there is no
random access — reaching minute 900 means decoding frames 0 through 900, and a
seek costs as much as a full decode (measured: 17.9 s to reach frame 1439 versus
17.5 s for the whole file). Decoding from scratch every minute would therefore
burn 8–18 s of CPU each time, which is why this is **not** a timer.

`omarchy-time-wallpaper` is instead a long-lived service. It keeps one `ffmpeg`
open and reads exactly one frame per minute from it:

```
ffmpeg -threads 6 -filter_threads 1 -i <video> \
  -vf "select=gte(n\,TARGET)" -fps_mode passthrough \
  -c:v png -pix_fmt rgb24 -compression_level 3 -pred none -f image2pipe -
```

`select` makes ffmpeg decode frames `0..TARGET` internally (unavoidable) but
push only `TARGET` onward, so catch-up costs decoding and nothing else. Once the
live stream is at the current minute, ffmpeg blocks on the pipe between reads —
that back-pressure *is* the pacing, and steady-state cost is ~0.3 s per minute.

Each frame is written as a PNG file and handed to Omarchy with
`omarchy theme bg set`, the same path the built-in background picker uses, so
transitions, the lock screen and theme retinting all behave normally.

PNG rather than WebP: the source is gbrp, planar RGB, and libwebp's lossy mode
would subsample that to 4:2:0 and add a second generation of loss on top of the
video's own. `-pred none` with `-compression_level 3` measured both faster and
smaller than the encoder defaults on sampled frames, and encoding is cheap
enough (tens of milliseconds) that the per-minute cost is still dominated by
reading the frame, not by the encoder.

Three details make it robust:

- **The clock is the only source of truth.** The frame index is recomputed from
  the wall clock every tick, never incremented, so a suspend, a clock jump or a
  missed minute cannot make it drift — it fast-forwards, or restarts the stream
  when the index goes backwards (midnight).
- **It yields the background.** It acts only while the current background is one
  of its own frames or the picker entry described below. Choosing any other
  wallpaper — a static image, a video, a theme default — hands the background
  back within seconds. There is no state file: the background pointer *is* the
  state, so restarting the shell, switching themes or crashing cannot leave it
  stuck claiming a wallpaper it no longer shows.
- **Decoding is software, and that is not a fallback.** The stream is HEVC Rext
  coded in gbrp to keep the render's colour exact, and NVDEC cannot decode it.
  `-hwaccel cuda` does not fail — it silently drops to the software decoder
  while still honouring the narrowed thread count, which is the slowest possible
  combination — and forcing `hevc_cuvid` decodes the RGB bitstream as if it were
  YUV, returning green frames. So there is no hardware path to choose between,
  and the thread count is the only dial: see the caveats.

## Layout

| Path | Installs to |
| --- | --- |
| `bin/omarchy-time-wallpaper` | `~/.local/bin/omarchy-time-wallpaper` |
| `bin/omarchy-time-wallpaper-cover` | `~/.local/bin/omarchy-time-wallpaper-cover` |
| `systemd/omarchy-time-wallpaper.service` | `~/.config/systemd/user/omarchy-time-wallpaper.service` |

The service is `Type=simple` because it is long-lived, `After=`/`PartOf=`
`graphical-session.target` so it starts and stops with the session, and
`Restart=always` as a backstop only — the daemon handles its own errors and does
not exit on a bad tick. `ConditionEnvironment=WAYLAND_DISPLAY` skips sessions
with no compositor, where there is nothing to hand a background to.

Decoded frames are written to `~/.local/state/omarchy/time-wallpaper/frames/`,
three at a time (a few MiB each, so about a dozen MiB in total). They live under
`state` rather than `cache` because the
background symlink must still resolve after a reboot, and must not be deleted by
a cache cleaner while it is being displayed. Inside the theme background folders
they would instead grow the picker's thumbnail cache without bound.

## Install

```sh
install -Dm755 bin/omarchy-time-wallpaper       ~/.local/bin/omarchy-time-wallpaper
install -Dm755 bin/omarchy-time-wallpaper-cover ~/.local/bin/omarchy-time-wallpaper-cover
install -Dm644 systemd/omarchy-time-wallpaper.service ~/.config/systemd/user/omarchy-time-wallpaper.service

bin/omarchy-time-wallpaper-cover          # the picker entry, see below
systemctl --user daemon-reload
systemctl --user enable --now omarchy-time-wallpaper.service
```

The scheme reads `~/Work/omarchy-dynamic-wallpaper/indoor/t_g1440_qp6_gbrp.mkv`
and writes to `~/.local/state/omarchy/time-wallpaper/frames/`. Override with:

| Variable | Default |
| --- | --- |
| `OMARCHY_TIME_WALLPAPER_VIDEO` | `~/Work/omarchy-dynamic-wallpaper/indoor/t_g1440_qp6_gbrp.mkv` |
| `OMARCHY_TIME_WALLPAPER_DIR` | `~/.local/state/omarchy/time-wallpaper/frames` |

Uninstall:

```sh
systemctl --user disable --now omarchy-time-wallpaper.service
rm -f ~/.local/bin/omarchy-time-wallpaper ~/.local/bin/omarchy-time-wallpaper-cover \
      ~/.config/systemd/user/omarchy-time-wallpaper.service
rm -f ~/.config/omarchy/backgrounds/*/time-lapse.png
rm -rf ~/.local/state/omarchy/time-wallpaper
```

## Choosing it

Open **Style → Background** (or double-click an empty part of the desktop) and
pick the time-lapse entry, then pick any other wallpaper to hand the background
back.

The picker renders no labels — entries are identified by thumbnail alone — so
the scheme needs a file inside the theme's background folder to be selectable at
all. `omarchy-time-wallpaper-cover` decodes one frame from the video and installs
it there as `time-lapse.png`:

```
~/.config/omarchy/backgrounds/<theme>/time-lapse.png
```

That file is only a handle: selecting it makes the background pointer land on
it, which is what the daemon watches for. Its content is the thumbnail, so the
frame chosen should read well as a still. Frame `420` (07:00) is the default;
pass another index to change it, and re-run the helper after adding a theme.

The handle is **not** committed to this repository. Publishing one frame of the
artwork is a different decision from publishing the scheme.

## Interactions

**tenzin.live-wallpaper** (video/static wallpaper picker). The two share the
single `~/.local/state/omarchy/current/background` pointer, so they are
mutually exclusive by construction: whichever was chosen last wins. Choosing a
video makes the time-lapse stand down, and choosing the time-lapse entry stops a
running video through tenzin's own change detection.

**theme-sync** (scheduled light/dark switching). Set its `keepBackground` option
to `true` so a scheduled switch retints the desktop without touching the
wallpaper:

```json
{ "keepBackground": true }
```

Without it, `omarchy theme set` swaps the background along with the palette and
the daemon puts the frame back on its next tick — a visible fight twice a day. A
theme switched to **by hand** still changes the background either way; only the
plugin's own switches honour `keepBackground`.

## Caveats

- **Booting late in the day costs a catch-up.** Starting at minute 1069 means
  decoding 1069 frames before the first frame is shown — measured at **14 s**,
  and roughly 19 s for a minute near midnight. Until then the desktop keeps
  showing the previous frame — the background symlink survives a reboot — so
  there is no black flash, but the frame is briefly stale.
- **A decoder stays resident, and software decoding makes it the only real
  dial.** One frame a minute is nearly free, but the process holds its decode
  buffers all day, so the thread count is chosen rather than left at the default
  pool. Measured, on the 4K gbrp source: **1265 MB at six threads (14 s
  catch-up)**, 1108 MB at four (34 s), 1387 MB at eight (18 s) and 1901 MB at
  the default pool (12 s). Six threads is the balance. It also uses **no VRAM at
  all**, which matters more here than the memory: the GPU stays free for renders.
  Freeing the RAM entirely would mean decoding on demand instead of staying
  resident, which costs 12–34 s up to several times a day rather than once.
- **Skipping ahead is cheap; re-decoding is not.** After a sleep, the frame
  index is recomputed and the gap is usually skipped by reading frames off the
  live stream. That costs ffmpeg a decode *and* a re-encode per frame worth
  skipped (~184 ms, mostly the 3.5 MB PNG), while restarting the stream decodes
  without encoding (~12 ms a frame) — so the daemon restarts once the gap passes
  about a fifteenth of the way to the current minute, and skips below that.
- Activating takes up to ~5 s: ownership is polled, not watched.
- Every change plays Omarchy's ~420 ms background reveal. That is the background
  plugin's own behaviour, not something this scheme controls.
- The wallpaper is a still per minute. Motion within a minute would need a
  different renderer entirely.
- `omarchy theme set` picks a theme's first background on a manual theme switch,
  and the picker entry sorts ahead of the theme's own images, so a manual switch
  can land on it and start the time-lapse. With `keepBackground` set this only
  affects hand-made switches.

## Development

```sh
python3 -m py_compile bin/omarchy-time-wallpaper
bin/omarchy-time-wallpaper --dry-run    # video, frame dir, ownership, minute
bash -n bin/omarchy-time-wallpaper-cover
journalctl --user -u omarchy-time-wallpaper.service -f
```

`--dry-run` reports `owned: no` whenever something other than this scheme owns
the background, which is the expected state on a fresh install.
