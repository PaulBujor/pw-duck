# pw-duck

Sidechain ducking for PipeWire and PulseAudio. Lower the volume of one app while something else is making sound, like a sidechain compressor on a mixing desk.

The motivating case: you're on a Zoom call with music playing. When someone speaks (or you do), the music dips. When there's a pause, it holds for a moment, then fades back up.

```
pw-duck --output zoom --input zoom --target tidal-hifi
```

- No dependencies beyond Python 3.8+ and the `pactl` / `parec` tools.
- Works per stream, not on the whole mix, so the app you duck never triggers itself.
- Restores your volume on exit, and follows along if you move the slider while it runs.

## Requirements

- Linux with PipeWire (`pipewire-pulse`) or PulseAudio.
- `pactl` and `parec` (package `pulseaudio-utils` / `libpulse`).
- Python 3.8 or newer. No pip packages.

## Install

```sh
git clone https://github.com/PaulBujor/pw-duck.git
ln -s "$PWD/pw-duck/pw-duck" ~/.local/bin/pw-duck
```

## Usage

Three kinds of sidechain source and one kind of target. All are repeatable.

| Option | Meaning |
|---|---|
| `--target APP` | App to duck. |
| `--output APP` | Duck while APP's **playback** is audible (someone talking in a call). |
| `--input APP` | Duck while the **microphone APP is recording from** is loud (you talking). Only active while APP has a capture stream open. |
| `--input-device SOURCE` | Duck while a capture device is loud, e.g. `@DEFAULT_SOURCE@`. Always opened, see [Bluetooth](#bluetooth-headsets). |

`APP` is matched case-insensitively against `application.name` (substring) or the process binary (exact). Find names with:

```sh
pw-duck --list
```

```
Playback streams (--output / --target)
  #11618  name='tidal-hifi'          binary='tidal-hifi'   vol=50%
  #11652  name='ZOOM VoiceEngine'    binary='zoom'         vol=100%
Capture streams (--input)
  #10864  name='ZOOM VoiceEngine'    binary='zoom'         vol=100%
```

### Examples

```sh
# Duck Tidal whenever Zoom plays audio or you speak into it
pw-duck --output zoom --input zoom --target tidal-hifi

# Only react to the other side of the call
pw-duck --output zoom --target tidal-hifi

# Duck music and a browser together while a voice-chat app talks, harder and slower to return
pw-duck --output discord --target spotify --target firefox --duck-db -18 --release 2

# Duck while anything is picked up by the default mic (no call needed)
pw-duck --input-device @DEFAULT_SOURCE@ --target spotify
```

### Options

| Option | Default | |
|---|---|---|
| `--duck-db` | `-12` | How far to duck, in dB. |
| `--hold` | `1.5` | Seconds to stay ducked after sound stops. Bridges pauses between words and sentences. |
| `--attack` | `0.15` | Seconds to duck down. |
| `--release` | `1.0` | Seconds to fade back up. |
| `--easing` | `smooth` | Fade curve: `smooth` (smoothstep, gentle start and end) or `linear` (straight ramp in dB). |
| `--output-threshold-db` | `-50` | Level (dBFS RMS) that counts as playback sound. |
| `--input-threshold-db` | `-40` | Level (dBFS RMS) that counts as you speaking. |
| `--input-sustain-ms` | `100` | The input must stay above its threshold this long to count. Ignores keystroke clicks. |
| `--calibrate-guided` | | Guided two-phase calibration (noise, then speech) with suggested settings. |
| `--calibrate-seconds` | `10` | Length of each guided phase. |
| `--calibrate SECONDS` | | Measure the first `--input` / `--input-device`, print percentiles, exit. |
| `--debug` | | Print levels and gain twice a second. |

## Calibrating the mic threshold

Playback from an app is usually digital silence between sounds, so the `-50` dB default is fine. A microphone has a noise floor and, on a laptop, picks up your own typing, so pick its settings from your own room. Join the call first so the app has a capture stream, then:

```sh
pw-duck --input zoom --calibrate-guided
```

It runs two phases of 10 seconds (`--calibrate-seconds`), announced by a notification and a sound: first your normal background noise (type, but don't talk), then talking. It prints the level percentiles of each phase and suggests settings:

```
sustained level (dBFS RMS):
  noise:  p5= -63.2  p25= -54.1  p50= -51.7  p75= -48.3  p90= -46.6  p99= -44.3  max= -43.0
  speech: p5= -62.3  p25= -54.9  p50= -47.5  p75= -42.4  p90= -38.0  p99= -34.4  max= -31.8

suggested: --input-threshold-db -41.8 --input-sustain-ms 100
  background noise above it: 0.0% of chunks
  speech above it:           23.2% of chunks (--hold 1.5s bridges the gaps between words)
```

The "speech above it" figure is per 20 ms chunk. Speech is full of short gaps, and `--hold` keeps the duck engaged across them, so it doesn't need to be near 100%. The tool warns if almost none of your speech clears the noise.

For a quick single measurement, `--calibrate SECONDS` prints both the instantaneous and the sustained percentiles. Use `--debug` to watch live levels (`instant/sustained`) while you tune.

### Ignoring keystrokes and other clicks

A laptop mic hears typing almost as loud as speech, so a plain level threshold can't tell them apart. They differ in duration: a keystroke is a click of a few milliseconds, speech is sustained. `--input-sustain-ms` (default 100) requires the level to stay above the threshold for that long, which drops the clicks. In the example above, typing peaks at -26 dB instantaneously but only -43 dB once sustained. Raise it if typing still triggers the duck, lower it if the duck reacts too slowly to the start of speech.

## How it works

1. `pactl list sink-inputs / source-outputs` finds the streams by app name once a second, so apps that start, stop or recreate their streams are picked up.
2. For each sidechain source, `parec --monitor-stream` taps that one stream (or `parec --device` taps a capture device) as 16 kHz mono PCM. Every 20 ms chunk gets an RMS level in dBFS, with the DC offset removed first because some laptop mics idle at a constant non-zero level that would otherwise look like permanent sound.
3. If any source was above its threshold within the last `--hold` seconds, the duck is engaged. Gain fades in dB over `--attack` / `--release` along the `--easing` curve, updated every 20 ms while fading. If the duck reverses mid-fade it eases from the current level, and the fade time scales with the distance left, so a short blip doesn't take a full attack or release.
4. The gain is applied with `pactl set-sink-input-volume`. PulseAudio and PipeWire volumes are cubic (`dB = 60·log10(value)`), so `-12 dB` is applied as a factor of `10^(-12/60)` on the raw value and the dB drop is exact.
5. On `SIGINT` / `SIGTERM` the original volumes are restored.

## Run it in the background

A systemd user unit is in [`contrib/pw-duck.service`](contrib/pw-duck.service):

```sh
mkdir -p ~/.config/systemd/user
cp contrib/pw-duck.service ~/.config/systemd/user/
$EDITOR ~/.config/systemd/user/pw-duck.service   # set your ExecStart arguments
systemctl --user daemon-reload
systemctl --user enable --now pw-duck
journalctl --user -u pw-duck -f
```

## Caveats

### Soft mute
Apps like Zoom keep the capture stream open when you mute inside the app and simply discard the audio. `--input` sees the microphone, not the app's mute button, so it will duck when you talk while muted. If that is a problem, use `--output` only, or raise `--input-threshold-db` so only close, loud speech triggers it. A hardware mute that stops the source avoids the issue.

### Bluetooth headsets
Opening a Bluetooth headset's microphone can make the system switch it from the high-quality music profile (A2DP) to the low-quality headset profile (HFP). `--input APP` is safe because it only taps a source the app is already using. `--input-device` opens the device itself, so avoid pointing it at a Bluetooth mic.

### Stream restore
A newly created stream usually starts at its last remembered volume. `pw-duck` treats that as the un-ducked volume. If a target stream appears while the duck is engaged, it is ducked immediately.

### Electron and browser apps
Browser and Electron apps often share one `application.name` across tabs or windows, so `--target firefox` ducks all of them. Use `pw-duck --list` to see what an app actually registers.

## Troubleshooting

| Symptom | Check |
|---|---|
| `--list` doesn't show the app | The app has no stream open right now. Start playback / join the call. |
| Duck never lets go | A source's noise floor is above its threshold. Run with `--debug` and compare. |
| Duck never triggers | Threshold too high, or the app's stream was not matched. Check `--list` and `--debug`. |
| `scan failed` in the log | PipeWire restarted or `pactl` is unavailable. It retries every second. |

## License

MIT, see [LICENSE](LICENSE).
