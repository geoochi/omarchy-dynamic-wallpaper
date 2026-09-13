# omarchy-dynamic-wallpaper

A time-of-day wallpaper for Omarchy: the desktop shows the frame of a 24-hour
timelapse that matches the current minute, so it follows wall-clock time instead
of a fixed image.

This repository holds the wallpaper *scheme*. The render it reads lives in
`indoor/` and is gitignored.

## How it works

One frame per minute of the day, so 1440 frames, named as zero-padded 4-digit
indices: `<frames-dir>/NNNN.webp`. The frame for the current minute is simply

```
index = hour * 60 + minute
```

`omarchy-time-wallpaper` runs on a systemd timer every minute, resolves that
frame, and hands it to Omarchy with `omarchy theme bg set`, which is the same
path the built-in background picker uses -- so transitions, the lock screen and
theme retinting all behave normally.

Two details make it robust:

- **Fallback while rendering.** If the wanted frame is not rendered yet, the
  highest rendered frame at or below it is used. A half-finished render still
  shows the latest scene available instead of a gap.
- **It yields the background.** The script does nothing unless the current
  background is one of the frame files or the picker entry described below.
  Choosing any other wallpaper -- a static image, a video, a theme default --
  hands the background back within a minute. There is no state file: the
  background pointer *is* the state, so restarting the shell, switching themes
  or crashing cannot leave it stuck claiming a wallpaper it no longer shows.

## Layout

| Path | Installs to |
| --- | --- |
| `bin/omarchy-time-wallpaper` | `~/.local/bin/omarchy-time-wallpaper` |
| `bin/omarchy-time-wallpaper-cover` | `~/.local/bin/omarchy-time-wallpaper-cover` |
| `systemd/omarchy-time-wallpaper.service` | `~/.config/systemd/user/omarchy-time-wallpaper.service` |
| `systemd/omarchy-time-wallpaper.timer` | `~/.config/systemd/user/omarchy-time-wallpaper.timer` |
| `hooks/post-boot.d/time-wallpaper` | `~/.config/omarchy/hooks/post-boot.d/time-wallpaper` |

The timer fires on `*:*:00` with `AccuracySec=1s`: the default one-minute
accuracy would let systemd coalesce runs across minute boundaries and skip
frames. The service is `oneshot` and carries
`ConditionEnvironment=WAYLAND_DISPLAY`, so it is skipped in sessions with no
compositor. The post-boot hook applies the right frame on login instead of
waiting up to a minute for the timer.

## Install

```sh
install -Dm755 bin/omarchy-time-wallpaper       ~/.local/bin/omarchy-time-wallpaper
install -Dm755 bin/omarchy-time-wallpaper-cover ~/.local/bin/omarchy-time-wallpaper-cover
install -Dm644 systemd/omarchy-time-wallpaper.service ~/.config/systemd/user/omarchy-time-wallpaper.service
install -Dm644 systemd/omarchy-time-wallpaper.timer   ~/.config/systemd/user/omarchy-time-wallpaper.timer
install -Dm755 hooks/post-boot.d/time-wallpaper ~/.config/omarchy/hooks/post-boot.d/time-wallpaper

bin/omarchy-time-wallpaper-cover          # the picker entry, see below
systemctl --user daemon-reload
systemctl --user enable --now omarchy-time-wallpaper.timer
```

The frame directory defaults to `~/Work/omarchy-dynamic-wallpaper/indoor/out`.
Point it somewhere else with `OMARCHY_TIME_WALLPAPER_DIR`; if the render moves,
that is the only thing to change.

Uninstall:

```sh
systemctl --user disable --now omarchy-time-wallpaper.timer
rm -f ~/.local/bin/omarchy-time-wallpaper ~/.local/bin/omarchy-time-wallpaper-cover \
      ~/.config/systemd/user/omarchy-time-wallpaper.{service,timer} \
      ~/.config/omarchy/hooks/post-boot.d/time-wallpaper
rm -f ~/.config/omarchy/backgrounds/*/time-lapse.webp
```

## Choosing it

Open **Style → Background** (or double-click an empty part of the desktop) and
pick the time-lapse entry, then pick any other wallpaper to hand the background
back.

The picker renders no labels -- entries are identified by thumbnail alone -- so
the scheme needs a file inside the theme's background folder to be selectable at
all. `omarchy-time-wallpaper-cover` copies one frame there as
`time-lapse.webp`:

```
~/.config/omarchy/backgrounds/<theme>/time-lapse.webp
```

That file is only a handle: selecting it makes the background pointer land on
it, which is what the script watches for. Its content is the thumbnail, so the
frame chosen should read well as a still. Frame `0420` (07:00) is the default;
pass another index to change it, and re-run the helper after adding a theme.

The handle is **not** committed to this repository. It is a copy of a render
frame, and publishing one frame of the artwork is a different decision from
publishing the scheme.

## Interactions

**tenzin.live-wallpaper** (video/static wallpaper picker). The two share the
single `~/.local/state/omarchy/current/background` pointer, so they are
mutually exclusive by construction: whichever was chosen last wins. Choosing a
video makes the time-lapse stand down on the next tick, and choosing the
time-lapse entry stops a running video through tenzin's own change detection.

**theme-sync** (scheduled light/dark switching). Set its `keepBackground` option
to `true` so a scheduled switch retints the desktop without touching the
wallpaper:

```json
{ "keepBackground": true }
```

Without it, `omarchy theme set` swaps the background along with the palette and
the next timer tick puts the frame back -- a visible fight twice a day. A theme
switched to **by hand** still changes the background either way; only the
plugin's own switches honour `keepBackground`.

## Caveats

- Activating takes up to a minute: the script polls on the timer tick rather
  than watching for the selection.
- The wallpaper is a still image per minute. Animation within a minute would
  need a different renderer entirely.
- `omarchy theme set` picks a theme's first background on a manual theme
  switch, and the picker entry sorts ahead of the theme's own images, so a
  manual switch can land on it and start the time-lapse. With `keepBackground`
  set this only affects hand-made switches.

## Development

```sh
bash -n bin/omarchy-time-wallpaper
bin/omarchy-time-wallpaper --dry-run   # report active state and the frame it would set
```

`--dry-run` prints `active=0` whenever something other than this scheme owns the
background, which is the expected state on a fresh install.
