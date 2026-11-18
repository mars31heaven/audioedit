# audioedit

A terminal tool for recording and quickly editing audio. It records from
your default source, applies tailored edits to WAV and FLAC files, lets
you layer other files or overdub live, and plays every result in mpv so
you can judge it straight away.

Each edit is written to a new numbered file. Your starting file or
recording is never changed or deleted.

## What it can do

- Record a take with a live level meter
- Apply edits from a searchable menu, one after the other
- Layer another file on top of the current one
- Overdub: record live over the current sound, like a looper
- Preview files by sound while you browse them
- Show levels and a spectrum in mpv after every render
- Master a finished album with one shared gain

## Requirements

- Python 3.9 or newer
- Python packages: `numpy`, `soundfile`
- Programs: `fzf`, `mpv`, `ffmpeg`
- For recording and overdub: `parec` (package `libpulse`) and `pactl`

On Arch Linux:

```bash
sudo pacman -S python python-numpy python-soundfile fzf mpv ffmpeg libpulse
```

The recording and meter features use PulseAudio tools and a terminal, so
the script is aimed at Linux.

## Install

```bash
chmod +x audioedit.py
cp audioedit.py ~/.local/bin/audioedit
```

Make sure `~/.local/bin` is in your `PATH`. Run `audioedit --help`
for the built-in instructions.

## How it starts

### Without a file or folder: record

```bash
audioedit
```

The script records a new take from the current default audio source.
Check the source first with:

```bash
pactl get-default-source
```

1. Press any key to start, any key to stop. Esc at the first prompt cancels.
2. The take plays in mpv.
3. Choose `k` keep and edit, `d` discard, `r` re-record, or `l` listen again.

Recordings are 24-bit stereo FLAC at 48 kHz. Without `parec`, it falls
back to 16-bit through ffmpeg, with a timer but no meter.

Add `--meter` to see the live level meter first, without saving anything.
It helps you set levels on your mixer or interface. Aim for peaks around
-12 to -6 dB.

### With a folder: pick a file

```bash
audioedit .
audioedit ~/Documents/audioedit_recordings/20261111
```

An fzf picker lists the WAV and FLAC files directly inside the folder
(not subfolders), newest first. Moving the cursor over a file **plays it**.
Moving to another file stops the first and plays the new one. The preview
pane shows duration, sample rate, channels, format and size.

| Key    | Action                           |
|--------|----------------------------------|
| Enter  | Select the file                  |
| Ctrl-R | Play the highlighted file again  |
| Esc    | Cancel                           |

### With a file: edit it directly

```bash
audioedit take.flac
```

The file opens straight in the edit menu. Only `.wav` and `.flac` are supported.

## Where files are saved

| What                | Where                                                        |
|---------------------|--------------------------------------------------------------|
| New recordings      | `~/Documents/audioedit_recordings/YYYYMMDD/recording_YYYYMMDD_HHMMSS.flac` |
| Edits               | Next to the file they came from, for example `take_reverse_01.flac` |
| Album masters       | `FOLDER/mastered/`                                           |

Edits keep the format of their source (FLAC stays FLAC, WAV stays WAV) and
are written as 24-bit. Existing files are never overwritten: the number at
the end goes up instead.

To move older recordings into day folders, run once:

```bash
audioedit --organize
```

## Workflow inside the menu

1. **Pick an edit.** An fzf list shows every edit with a one-line
   description. Type to filter, Enter to choose. The header shows the
   version you are working on and how many edits you have kept.
2. **Answer the prompts.** Each edit explains its settings first. Press
   Enter to accept the default shown in brackets.
3. **Listen.** The result is written to a new file and plays in mpv.
4. **Decide:**

| Key | Action                                                   |
|-----|----------------------------------------------------------|
| `k` | Keep it and continue editing                             |
| `f` | Keep it and finish the session                           |
| `d` | Discard it and go back to the previous version           |
| `r` | Run the same edit again with new answers                 |
| `a` | Discard it and choose a different edit                   |
| `l` | Listen again                                             |
| `q` | Discard it and end the session with the previous version |

5. **Go back if needed.** Choose `undo` in the menu to remove the last
   kept edit. Choose `finish` or press Esc to end the session.

When the session ends, intermediate versions are deleted. Only your
starting file and the last version remain, and the final line shows its
path. Use `--keep-all` to keep every render.

## The mpv window

After each render, mpv shows level bars with dB values above a scrolling
spectrum. Steady hiss appears as a bright band near the top, and crackle
shows as vertical streaks. Press `i` in mpv for file information. If the
window stays blank, use `--plain` for a plain window without the display.

## Edits

Every edit explains its settings when you pick it. These are short
invitations to experiment.

### Cut and shape

- **extract**: keep only a time range
- **trim-silence**: remove silence at the start and end
- **reverse**: play the file backwards
- **fade**: fade in and out
- **varispeed**: change speed and pitch together, like tape
- **stretch**: Paulstretch-style stretch into smooth drones
- **loop**: repeat the whole file, with an optional crossfade
- **loop-section**: loop a time range, then continue with the file

### Combine and record

- **layer**: mix another file from the same folder on top
- **overdub**: record live over the current version while it plays.
  Settings: level of the new take, level of the existing sound, latency
  compensation, and an optional loop. Use headphones.

### Space

- **stereo-width**: mid/side width, from mono to wider
- **channel-switch**: one channel only, then the other, then stereo, for
  times you set, with a choice of which side starts

### Level and repair

- **normalize**, **gain-up**, **gain-down**: simple level changes
- **gain-crush**: drive into hard clipping, then trim
- **denoise**: reduce steady hiss using a noise sample
- **declick**: remove short clicks and crackle
- **declip**: soften flat-topped peaks
- **remove-dc**: remove DC offset and very low drift

### Failure and glitch

- **dropout**: punch random or regular chunks of silence into the sound
- **bitcrush**: reduce bit depth and sample rate
- **slice-shuffle**: cut into slices, reorder, repeat or reverse them
- **stutter**: repeat small chunks at random moments

### Texture and synthesis

- **wavefolder**: fold the waveform back on itself for metallic harmonics
- **ring-mod**: multiply by a carrier for tremolo, bells or shimmer
- **spectral-freeze**: hold one moment's spectrum as a drone
- **granular**: a grain cloud built from the file, with ambient and glitch
  presets, tempo-grid timing, repeats and pitch sets

For the random effects, a seed of 0 gives a new result each time. Any
other number repeats the same result, which is useful while you tune the
other settings.

### Mastering

- **peak-report**: shows peak, average level and what cutting peaks would
  do. It makes no file.
- **ceiling-clip**: cut only the peaks above a level, then set the final peak
- **`--album FOLDER`**: applies one shared gain to every track in a folder
  and writes 24-bit copies to `FOLDER/mastered/`. Relative levels between
  tracks stay as you composed them. `--album-peak DB` sets the final peak
  of the loudest track (default -1).

For finished projects, render from your DAW at 32-bit float with no
limiter on the master. Then run `peak-report`, and use `normalize` for a
single track or `--album` for a release.

## Options

| Option           | Meaning                                                  |
|------------------|----------------------------------------------------------|
| `--meter`        | Live level meter before recording                        |
| `--organize`     | Move old recordings into day folders, then exit          |
| `--album FOLDER` | Master a whole folder with one shared gain               |
| `--album-peak DB`| Final peak for `--album` (default -1)                    |
| `--keep-all`     | Keep every render of the session                         |
| `--plain`        | Plain mpv window, no meter or spectrum                   |
| `--help`         | Full built-in instructions                               |

## Time format

Times can be typed as `4`, `4.5`, `17:54`, `00:17:54` or `00:17:54.320`.

## Suggested orders

- **Noisy recording:** extract, declip, declick (start at 15), denoise
  (6 to 10 dB), normalize.
- **Glitch chain:** slice-shuffle, then stutter, then dropout.
- **Always:** run `declip` before `gain-crush`, never after.

## Good to know

- Edits run on the whole file in memory, so very long files with heavy
  edits (`stretch`, `granular`) can take a while.
- Peaks above full scale are clipped when a file is saved. Use `normalize`
  or `ceiling-clip` before heavy gain changes.
- The script has not been run end to end. Check it with
  `python3 -m py_compile audioedit` and report anything that fails.
