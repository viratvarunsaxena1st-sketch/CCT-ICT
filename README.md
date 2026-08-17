# aPASAT

Adaptive PASAT (Paced Auditory Serial Addition Test) with a visual-judgment
twist, plus an experimental dichotic attention-switching mode. Single-page
app, no build step, no dependencies.

## Running it

This app fetches its audio as separate `.wav` files, so it needs to be
served over `http://` or `https://` — opening `index.html` directly from
disk (`file://`) will load the page but **the browser will block the audio
fetches** (CORS doesn't allow `fetch()` of local files from a `file://`
page). Any of these work:

```bash
# Python (already on most machines)
python3 -m http.server 8000

# Node
npx serve .
```

Then open `http://localhost:8000` (or whatever port it prints).

## Deploying to GitHub Pages

1. Push this folder's contents to a GitHub repo (`index.html`, `audio/`,
   and this `README.md` at the repo root, or under `/docs`).
2. Repo **Settings → Pages** → set the source to the branch/folder you
   pushed to.
3. GitHub gives you a `https://<user>.github.io/<repo>/` URL — open it.
   No build step, no config file needed; it's a static site.

## How the audio works

Digits are pre-recorded speech, not `speechSynthesis()` — the previous
build used the browser's speech API and had two problems that turned out
to be unfixable rather than just buggy: it has no `AudioNode` to route
through a `StereoPannerNode` (so it can't be panned hard to one ear for
dichotic mode), and it's a single global queue (so it can't play two
digits at once for two ears). Pre-recorded clips loaded through the Web
Audio API sidestep both, and as a side effect are also just plain more
reliable — no per-call startup latency, no dropped/garbled speech at fast
paces.

Each digit (1–9) exists as four separate files, fetched once at startup
and decoded into `AudioBuffer`s that stay in memory for the whole session
— no per-trial network or decode latency:

```
audio/male/1.wav .. 9.wav          normal-speed male voice
audio/female/1.wav .. 9.wav        normal-speed female voice
audio/male-fast/1.wav .. 9.wav     male voice, pre-stretched to 1.5x
audio/female-fast/1.wav .. 9.wav   female voice, pre-stretched to 1.5x
```

The `-fast` sets are **not** the normal clips played back with
`AudioBufferSourceNode.playbackRate = 1.5`. Naively raising `playbackRate`
resamples the audio, which raises pitch right along with speed — the
classic "chipmunk" effect — because it's just replaying the same samples
faster rather than actually shortening the sound. The `-fast` files were
time-stretched offline with a phase vocoder (`ffmpeg`'s Rubber Band
filter, formant-preserved) so only the duration changes, not the pitch —
the fast voice still sounds like the same person talking quickly.
Every clip, either voice, either speed, is always played back at
`playbackRate` 1; "fast" just means "fetch the shorter file," never "play
this file differently." All clips are also silence-trimmed and
loudness-matched (RMS-normalized) so no digit is quieter or harder to
make out than the others.

Non-dichotic mode plays whichever voice is picked in Settings → Voice
(male/female). Dichotic mode always plays both at once — male in the left
ear, female in the right — as a constant cue for which stream is which,
independent of that setting.

## Settings

- **Mode**: Adaptive (speed changes with performance) or Fixed.
- **Starting speed / adaptive limits**: inter-stimulus interval in ms,
  with optional floor/ceiling on how far adaptive mode can push it.
- **Session duration**, **number type probabilities** (sum / off-by-one /
  withhold), **response keys**, **instant feedback beep**.
- **Voice**: male or female (non-dichotic mode only).
- **Voice speed**: normal or 1.5x.
- **Dichotic attention switching** (experimental): two digits at once,
  one per ear; a beep cues which ear to follow, switching every few
  trials.
- **Mobile mode** (on-screen buttons) and **Zen mode** (hide
  non-gameplay UI during play).

Settings persist between sessions (`localStorage`, or the host page's
storage API if running inside one that provides `window.storage`).
