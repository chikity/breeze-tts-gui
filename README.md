# Breeze TTS 2 — web GUI for Apple Silicon

An unofficial local web GUI and a small set of launcher scripts for running
[Breeze TTS 2](https://huggingface.co/BreezeBlue/Breeze-TTS-2) on an Apple Silicon
Mac through the MLX backend.

It speaks a line, clones a voice, reads an EPUB aloud with follow-along text, or
renders a whole book to audio from the terminal — locally, with no account and no
upload.

Nothing here is a fork of anything. This repo is a **client**: a single static HTML
file that talks to a separately-installed Breeze TTS server over HTTP. The server
lives in its own checkout and is never modified, so it can be updated freely
without these files drifting.

Measured on an M5 Pro (18-core CPU, 20-core GPU, 64 GB) with the default `mixed`
weights: **~126–158 ms to first audio, real-time factor 0.54–0.58** — comfortably
faster than real time, so it streams without stalling.

---

## Quick start

The one-time setup — the server checkout, Python 3.12, and ~7.6 GB of weights — is
under [Setup](#setup). After that, one terminal:

```bash
breeze-run
```

then either open the GUI:

```bash
breeze-gui          # http://127.0.0.1:8090
```

or render a book without it:

```bash
breeze-epub book.epub --list                  # chapters, lengths, no audio
breeze-epub book.epub --voice alice           # WAVs: one per chapter + whole book
breeze-epub book.epub --voice alice --m4b     # one audiobook (needs ffmpeg)
```

In the GUI, press **Read a book**, drop an EPUB, pick a saved voice, and press
**Read aloud**; the text follows along sentence by sentence.

---

## Architecture

```
  browser  ──http://127.0.0.1:8090──▶  index.html            (this repo, static)
     │
     ├── <audio src="http://127.0.0.1:8080/v1/audio/speech.wav?...">
     │        progressive WAV, plays as it is generated — needs no CORS
     │
     └── fetch() ──▶ http://127.0.0.1:8080/v1/{voices,speech}
              needs the GUI origin allowlisted on the server

  breeze-run  ──▶  breeze_infer.api on 127.0.0.1:8080
                   loopback only · WebSocket off · CORS = the GUI origin
```

Two details drive the whole design:

- **`<audio>` is not subject to CORS**, and the server's `GET` route skips the
  origin check, so playback streams natively with no configuration.
- **`fetch` POSTs are subject to CORS.** With CORS off, the server rejects any
  `Origin`-bearing *unsafe* request (`POST`/`DELETE`) with `403 origin_not_allowed`
  *before* the endpoint runs. The GUI is served from a different port, so it is a
  cross-origin client and its origin must be allowlisted by `breeze-run`.

`127.0.0.1` and `localhost` are **separate origins** to a browser, so both are
allowlisted.

The copy served from `:8090` is a static file server; the GUI has no build step, no
bundler, no package manager and no external requests. It is one HTML file.

---

## Requirements

- An Apple Silicon Mac (M1 or later) with **at least 16 GB** of memory — the MLX
  backend refuses to start below that. The default `mixed` profile needs ~8.2 GB.
- macOS 14 (Sonoma) or later.
- [`uv`](https://docs.astral.sh/uv/) and Python 3.12.
- Optional: [`ffmpeg`](https://ffmpeg.org/) for `--m4b` (`brew install ffmpeg`).

---

## Setup

```bash
git clone <this repo> breeze-tts-gui
./breeze-tts-gui/bin/breeze-setup
```

`breeze-setup` checks the Mac, clones the server, creates the Python 3.12 venv,
installs the server's dependencies, downloads the pinned weights, links the scripts
into `~/.local/bin`, and verifies that the server package imports. It is
idempotent: an existing checkout, venv or weights snapshot is reused.
`breeze-setup --check` prints what is missing without changing anything, and
`--no-model` skips the download. `breeze-doctor` reports the result.

<details>
<summary><strong>Manual setup — exactly what the script does</strong></summary>

### 1. The server

```bash
git clone https://github.com/Shoofio/breeze-tts2-fast-streaming-api.git
cd breeze-tts2-fast-streaming-api
uv venv --python 3.12
```

Install its dependencies **with an explicit interpreter path**:

```bash
uv pip install --python .venv/bin/python \
  -r requirements.txt --overrides requirements-mac-overrides.txt
```

> **Do not skip `--python`.** `uv pip install` prefers the `VIRTUAL_ENV`
> environment variable over `./.venv` if it is set, so if you have any virtualenv
> active it will silently install into *that* one instead, leaving the server's
> `.venv` empty. Setting `--python .venv/bin/python` (or running with
> `env -u VIRTUAL_ENV`) removes the ambiguity.

### 2. The model weights

One download covers both the `mixed` (default) and `bf16` profiles — `mixed` only
quantizes the depth decoder at load:

```bash
HF_XET_FIXED_DOWNLOAD_CONCURRENCY=4 \
  uvx --from huggingface_hub hf download mlx-community/Breeze-TTS-2-mlx \
  --revision 3c8829fb7fd335818f085cd2ef49b4100c0e46c8
```

That's 7.6 GB into the standard Hugging Face cache (`~/.cache/huggingface/hub`),
which is where the server looks. The `8bit` profile is a *separate* 4.6 GB
checkpoint — read [Weight options](#weight-options-precision) before pulling it.

> **The download is the slow part, and the xet backend degrades on its own.** An
> early attempt fell from 0.67 to 0.22 MB/s and never recovered: the adaptive
> concurrency controller collapses to a single connection and ratchets down
> forever — its log filled with 760 repeats of `Decreased concurrency from 1 to 1`
> while reporting `success_ratio = 1.000` next to a threshold of `0.500`. Pinning
> it fixes the decay rather than the speed:
> `HF_XET_FIXED_DOWNLOAD_CONCURRENCY=4 hf download ...` produced zero such events
> and held steady. Peak throughput is your link, not a setting.

### 3. This repo

```bash
git clone <this repo> breeze-tts-gui     # or just keep the checkout
ln -sfn "$PWD/breeze-tts-gui/bin/breeze-run"    ~/.local/bin/breeze-run
ln -sfn "$PWD/breeze-tts-gui/bin/breeze-say"    ~/.local/bin/breeze-say
ln -sfn "$PWD/breeze-tts-gui/bin/breeze-gui"    ~/.local/bin/breeze-gui
ln -sfn "$PWD/breeze-tts-gui/bin/breeze-epub"   ~/.local/bin/breeze-epub
ln -sfn "$PWD/breeze-tts-gui/bin/breeze-doctor" ~/.local/bin/breeze-doctor
ln -sfn "$PWD/breeze-tts-gui/bin/breeze-setup"  ~/.local/bin/breeze-setup
```

</details>

---

## Running

Two terminals:

```bash
breeze-run     # the TTS server on 127.0.0.1:8080 — leave it running
breeze-gui     # the GUI on 127.0.0.1:8090 — opens your browser
```

Then open <http://127.0.0.1:8090/>. `breeze-gui` checks that the API is up and
tells you to run `breeze-run` first if it isn't. If anything looks wrong,
`breeze-doctor` prints the hardware, install and server state in one pasteable
report.

There is **no** command-line synthesis on the MLX backend — the server's own
`infer.py` is CUDA-only. `breeze-say` fills that gap:

```bash
breeze-say "The birch canoe slid on the smooth planks." out.wav
breeze-say "(sigh) It is good to hear your voice again." out.wav
breeze-say "Read this slowly." out.wav -F cfg_scale=4 \
  --form-string "instruction=Speak with a restrained, serious tone."
breeze-say "Clone this line." out.wav \
  -F "ref_audio=@reference.wav" --form-string "ref_text=the exact transcript"
```

### Environment overrides

| Variable | Default |
| --- | --- |
| `BREEZE_PORT` | `8080` — server port |
| `BREEZE_PRECISION` | `mixed` (default), `bf16` or `8bit` — picks weights *and* speed settings |
| `BREEZE_MLX_CACHE_GB` | `1` — caps MLX's buffer cache; see the memory note below |
| `BREEZE_MLX_QUANT` | `depth:8` under `mixed`; parts are `backbone`/`depth`, bits `2,3,4,5,6,8` |
| `BREEZE_MLX_GROUP` | `64` — quantization group size; smaller is more accurate |
| `BREEZE_CHUNKS` | streaming chunk bounds, set per precision |
| `BREEZE_GUI_PORT` | `8090` — GUI port (and the allowlisted origin) |
| `BREEZE_CORS` | the GUI origin on both `127.0.0.1` and `localhost` |
| `BREEZE_REPO` | `~/Github/breeze-tts2-fast-streaming-api` |
| `BREEZE_MODEL` | the pinned snapshot path for the chosen precision |
| `BREEZE_API` | `http://127.0.0.1:8080` (GUI side) |
| `BREEZE_PY` | the server venv's `python` |

---

## Using the GUI

Above the form is a **Speak / Read a book** switch. Both modes share the Voice and
Delivery panels, so one saved voice or reference clip covers either a typed line or
an entire EPUB.

The form reads top to bottom as **1 · Script → 2 · Voice → 3 · Delivery**;
**Generate** sits in a bar at the end of it, and the output panel is pinned in a
rail on the right. `⌘↵` generates from anywhere.

- **Script** — what to say. The chips insert **vocal events**: `(sigh)`,
  `(laugh)`, `(cough)`, `(clears throat)` in English, `[叹气]`, `[笑]`, `[咳嗽]`,
  `[清嗓子]` in Chinese. Match the event language to the text language.
- **Voice** — `No reference` is voice *design* (pick a voice from a description).
  `Saved voice` reuses something in the library. `Upload a reference` is voice
  *clone*, and needs the reference clip plus its **exact transcript**.
  `Record a reference` does the same from your microphone, and writes the
  transcript for you — see below.
- **Gender** — voice *design* only, so it appears only under `No reference`. Picking
  `Female` or `Male` folds `"A female voice."` / `"A male voice."` onto the front of
  the Instruction, because the model has no gender field of its own — it reads
  gender out of the instruction. It needs **cfg_scale 2–4** to land; at `1` the
  instruction is largely ignored. With a reference clip, the clip already fixes the
  voice, so gender is not offered.
- **Instruction** — optional. Adding one to a reference is what makes it voice
  *direction* rather than a plain clone.
- **Delivery** — `cfg_scale` (`1` is plain; higher follows the instruction more
  strongly, and voice design and direction usually want 2–4) and **Seed** (same seed
  and settings reproduce the same take). **Advanced sampling** exposes the rest.
- **Output** — plays while it generates, offers `Download WAV`, and reads out
  first-audio time, audio length, generation time and real-time factor.
- **Voice library** — list, use, delete. Named voices persist to the server's
  `voices/` directory; unnamed (`v_…`) ones live in memory, capped at 64.

**Reference clips:** WAV, FLAC or OGG; 1–8 channels; 8–192 kHz; **≤ 30 s** and
≤ 25 MiB. Clean, non-looping speech with minimal background noise works best.
`ref_text` must transcribe the clip *verbatim*, including any repeated words —
getting this wrong is the usual cause of a poor clone.

### Recording a reference

This is the recommended way to make a clone reference, because it removes the step
people get wrong. Choose **Record a reference** and the GUI shows a passage to read
aloud. You read it, it records you, and it sets the transcript to *exactly* that
passage — no transcription, and nothing to mistype. The transcript is correct by
construction because the GUI already knew what you were going to say.

- Pick **English** or **中文** and `↻` for a different passage. Each is roughly
  15–20 seconds of speech, long enough to characterise a voice.
- The highlight moves at a fixed pacing cue (~2.5 words/s). It is a *guide for you*
  to read along with, not a measurement of what you actually said — nothing is
  listening or tracking your position. Read naturally; ignore it if it drifts.
- Watch the level meter. If it barely moves, your input is too quiet and the clone
  will be poor. Aim for a healthy spread without hitting the end.
- Recording stops itself at 28 s. The server rejects anything over 30 s. Aim for
  15–25 s; under about 2 s is not usable and the GUI will say so.
- Microphone processing (echo cancellation, noise suppression, auto gain) is
  deliberately **disabled** — that chain is tuned for phone calls, and its
  artefacts are exactly what a clean reference should not contain.
- If several input devices exist, a picker appears after the first recording.

**Two caveats.** Recording needs a *secure context*: `http://127.0.0.1:8090/` and
`http://localhost:8090/` qualify, but if you reach the GUI over a LAN address such
as `http://192.168.1.20:8090/` the browser will refuse microphone access outright.

And the browser's `MediaRecorder` only produces webm/opus or mp4, neither of which
the server accepts, so the GUI decodes the take and re-encodes it to 16-bit PCM
WAV in the page. That is a second-generation encode — harmless in practice, since
the model's codec tokenises the audio to discrete codes anyway, but if you already
have a good clean recording, uploading it directly avoids the round trip.

Reshuffling the passage after a take **discards the recording** on purpose: an old
clip paired with a new transcript would build a voice from mismatched text, which
clones badly.

---

## Reading an EPUB

`breeze-epub` turns a text EPUB into audio through the running server: chapters in
spine order, one voice throughout, no extra dependencies beyond Python 3 and the
standard library.

```bash
breeze-epub book.epub --list                    # chapters, lengths, no audio
breeze-epub book.epub --voice alice             # reuse a saved voice
breeze-epub book.epub --voice alice --m4b       # one audiobook, chapter marks
breeze-epub book.epub --ref me.wav \
  --ref-text "the exact words in me.wav"        # clone a reference for the whole book
breeze-epub book.epub --instruction "A calm, low female narrator." --cfg 3
```

Output goes to `<epub dir>/<stem>/` (or the second positional argument): one WAV per
chapter, a whole-book WAV and an `.m3u` playlist. `--m4a` adds per-chapter AAC via
macOS's `afconvert`; `--m4b` writes a single audiobook with chapter marks, metadata
and the EPUB cover. Everything is 24 kHz mono, the model's own rate.

**`--m4b` is the file you want for listening or publishing.** It needs
[`ffmpeg`](https://ffmpeg.org/) on `PATH` (`brew install ffmpeg`). The title, author
and cover come from the EPUB's package document; `--title`, `--author` and
`--cover` override them. Audio is AAC at 64 kbps, loudness-normalised to about
−19 LUFS (`--no-loudnorm` turns that off). It is muxed from the chapter files, so it
works with `--no-book` and past the 4 GiB WAV limit. Chapters are joined with no
inserted silence; the chapter marks are the separation.

**The GUI has the same reader** under **Read a book**. Drop an EPUB on the panel and
the page parses it itself — the ZIP is opened with `DecompressionStream`, the OPF
spine is read, and the XHTML becomes plain text; nothing is uploaded. Chapters appear
with their approximate length, and playback runs sentence by sentence, advancing
through parts and chapters on its own with pause, stop, previous/next chapter, and
part navigation inside a chapter (the arrows beside the `2 / 5` counter). Jumping to a
part while playing continues seamlessly; while stopped it only moves the reading
position, and Read starts from there.
The chapter text is shown while it plays, with the current sentence highlighted and
the pane scrolled to follow it (the **Follow** button turns that off if you want to
read ahead). The API returns no timestamps, so book playback synthesizes **one
sentence per request**: the sentence being spoken is then known exactly rather than
interpolated, and the highlight cannot drift. The next sentences are fetched and
buffered while the current one plays, so the costs are a short wait for the first
sentence (~0.55× its length) and some per-request overhead. The voice is snapshotted
when playback starts and reused for every request, so a chapter boundary can never
change speaker. With **No reference** there
is no stored speaker to reuse, so the GUI anchors one first: it synthesizes a short
passage (~15 s of audio) with the current instruction, registers that as a reference,
and reads the book from the resulting `voice_id`. The anchor is cached until the
instruction, cfg or seed changes. Without it, each part would be read in a different
voice — that is the model, not the GUI. Saving a chapter re-renders it (the browser
never buffers the stream it played); writing a whole book is still a job for
`breeze-epub`, since hours of PCM do not belong in a tab.

Saving is not instant, because it regenerates: on this hardware it runs at the model's
RTF of ~0.55, so a 13-minute chapter takes about 7 minutes (the whole of *Alice*,
3 h 14 m of audio, takes roughly 1 h 46 m). The save panel in the output rail reads the
streaming response as it arrives, so it shows generated time against the **whole
chapter's** estimate, the part counter, an ETA and a **Cancel** button (Escape also
cancels). The estimate uses the characters-per-second rate measured during playback
and refines itself as bytes arrive; it is clamped short of 100% until the chapter
actually closes.

**Why a reference voice is the consistent one.** The model keeps no speaker between
requests. Each request is conditioned from scratch, so "the same voice" only survives
if the same speaker conditioning goes in every time. A reference clip (or a saved
voice built from one) fixes the speaker. Voice design has no speaker of its own, so
`breeze-epub` anchors it: it synthesizes one short sample (~18 s) with the instruction,
registers that as a reference, and uses the resulting `voice_id` for every chunk.
Either way the clip is registered **once** with `POST /v1/voices`, and the server
caches the built prefix, so there is no per-chapter encode cost. Use
`--save-voice NAME` to keep the anchor for later runs. Anchoring costs one extra
~18 s sample the first time; later runs reuse the voice id from
`.breeze-epub.json` (or the saved name).

Two knobs are per-run and deterministic on purpose: `--seed` (default `42`) and
`--cfg` (default `2` with `--instruction`, else `1`). Same text, voice and settings
reproduce the same take, which is also what makes the cache sound.

**Chapters and chunks.** A chapter becomes one request when it fits under the
server's 10,000-character text limit (`--chunk-chars`, default 9,000), so the server's
own segmenter handles pacing continuity inside it. Longer chapters split at paragraph
breaks, falling back to sentence boundaries and then hard cuts; the splitter never
drops or reorders text. Between chapters the whole-book file gets `--gap` seconds of
silence (default 0.8).

**Resume.** Each chunk's PCM is cached under `<outdir>/.parts/` and deleted once its
chapter WAV is written, so an interrupted run restarts at the chapter it was on and
only repeats the one chunk in flight. The cache is keyed by a signature of the voice,
settings and text, and the run's signature is recorded in `.breeze-epub.json`: change
the voice and the tool refuses to mix old audio into new, until you pass `--fresh` or
pick another outdir.

`--start` / `--end` restrict the range (1-based, inclusive) — useful to skip front
matter on the first pass. `--no-book` skips the whole-book file, which matters for a
long novel: 24 kHz 16-bit mono WAV is about 173 MB per hour, so `--m4a` is worth
turning on, and `--m4b` is smaller still. At the measured RTF of ~0.55, an hour of
finished audio costs roughly 33 minutes of generation, and one request runs at a
time, so leave `breeze-run` to it.

The tool POSTs `application/x-www-form-urlencoded`, not multipart, so the
[`curl -F` parenthesis trap](#the-curl-trap) below cannot bite it.

---

## Design

The interface was rebuilt against the **UI UX Pro Max** design skill (vendored in
`.opencode/skills/ui-ux-pro-max/`). Its `--design-system` search resolves this
product as a *Developer Tool / IDE* and returns *Dark Mode + Minimalism* with a
*Real-Time Monitor + Terminal* app style, a "code dark + run green" palette and a
mono+sans type pair. Those tokens live at the top of `index.html` as custom
properties — colour, a 4/8px space scale, radius, elevation and motion durations.

Two deliberate departures from what the skill recommends, both to protect
properties this repo already had:

- **System fonts, not Google Fonts.** The skill's pair is JetBrains Mono + IBM Plex
  Sans loaded by `@import`. That is an external request on every load, and "no
  external requests… it is one HTML file" is a feature here, so the type keeps the
  mono+sans *character* with `-apple-system` and `ui-monospace` instead.
- **Light by default.** The skill's anti-pattern list says avoid a light-mode
  default. Reading the teleprompter aloud in a room is a real use, so light stays
  the default and `prefers-color-scheme: dark` gets the OLED treatment.

Accessibility is part of the system rather than a pass bolted on afterwards:
`:focus-visible` rings on every control (previously only inputs had one), inline
Lucide SVG icons in place of the `🎲` / `↻` / `●` / `■` glyphs, labelled
`role="group"` chip rows, a `<fieldset>`-grouped radio set for the voice source,
`role="alert"` on the message region, `role="progressbar"` on the recorder,
`aria-label` on every icon-only button, a skip link, and `prefers-reduced-motion`
honoured. The teleprompter marks the current word with weight and underline as well
as colour. Verified with no horizontal overflow at 390 / 768 / 1440 px.

---

## The curl trap

Worth knowing even if you only use the GUI, because it cost real debugging time.

**`curl -F` gives special meaning to a value that *starts with* `(`**, wrapping the
field as a nested `multipart/mixed` part. Breeze's vocal-event syntax is exactly
`(sigh)`, `(clears throat)` and friends, so:

```bash
# BROKEN — the server sees a malformed text field
curl -X POST .../v1/audio/speech -F "text=(sigh) Hello." ...

# plain ASCII is fine, and the parens are only a problem at the START of a value
curl -X POST .../v1/audio/speech -F "text=Hello (world)." ...
```

It fails in two different ways: `400 text must be UTF-8 text` when a reference file
is also present (the `text` part's content-type stops being UTF-8), or a **60-second
ramble** when it isn't (the model never emits EOS and runs to the 750-frame cap).

Fixes:

```bash
# 1. --form-string sends the value verbatim  (what breeze-say and the GUI use)
curl -X POST .../v1/audio/speech --form-string "text=(sigh) Hello." ...

# 2. or use the GET route, which is query-encoded and has no such trap
curl -G .../v1/audio/speech.wav --data-urlencode "text=(sigh) Hello." -o out.wav
```

The GUI only ever builds query-encoded `.wav` URLs, so this cannot bite there. Its
**Copy as curl** button emits `--data-urlencode`, and `breeze-say` uses
`--form-string`.

---

## Performance

Measured here on an M5 Pro with the default `mixed` weights, 24 kHz output, warm —
the first request after each start pays an `mx.compile` warmup and reads high:

| Mode | First audio | Real-time factor |
| --- | --- | --- |
| Plain text | 126–153 ms | 0.54 |
| Voice design / direction (CFG 4) | 143–158 ms | 0.57–0.58 |

A voice clone pays a one-off reference encode and prefix build on top, roughly
300 ms for the first request against a given voice.

Real-time factor below 1 means audio is produced faster than it plays. The GUI's
"first audio" is measured client-side as the moment the first audio bytes arrive,
which is the same quantity the model's own figures describe; it deliberately does
**not** wait for playback to start, which would fold in browser audio-device
startup.

### Weight options (precision)

`breeze-run` takes `BREEZE_PRECISION`, which selects the weights *and* the matching
speed settings. The server accepts exactly three checkpoint kinds and refuses
everything else — notably 4-bit, which is why the popular 4-bit MLX conversion
can't be used here at all.

All three measured here, identical speed settings and generation sequence, so the
comparison isolates the quantization:

| `BREEZE_PRECISION` | Backbone | Depth decoder | First audio | RTF | Memory | Download |
| --- | --- | --- | --- | --- | --- | --- |
| `mixed` *(default)* | **exact** | int8 | 126–158 ms | 0.54–0.58 | 8.2 GB | 7.6 GB |
| `bf16` | **exact** | **exact** | 152–204 ms | 0.75–0.80 | 8.5 GB | same file |
| `8bit` | mxfp8 | mxfp8 | 107–143 ms | 0.48–0.51 | 5.7 GB | 4.6 GB |

**Only `8bit` quantizes the backbone**, and that is the part that matters: the
backbone sets prosody and is where a voice direction lands, and CFG *amplifies* its
error by multiplying the gap between the two guidance rows. In the fork author's
blind listening test, `8bit` placed worst on seven of ten lines and best on none,
while `bf16` and `mixed` tied.

**That is why `mixed` is the default.** Against `8bit` it trades ~13% throughput and
2.5 GB of memory for a bit-exact backbone. Against `bf16` it gives up only the depth
decoder's exactness — the part their blind test could not hear — for ~45% less
generation time. On this hardware `bf16` is the wrong purchase: slowest, and only
marginally the most faithful.

All three are comfortably real-time here, and that is machine-specific. On a 16 GB
M5, `bf16` measured RTF 1.47 — *slower* than real time — where streamed playback
starves. The 20-core GPU is what makes full precision viable.

**Memory: the buffer-cache cap matters more than the precision.** MLX keeps freed
buffers by default, so an uncapped `bf16` run climbs from ~9 GB toward ~15 GB. All
three profiles set `BREEZE_MLX_CACHE_GB=1`, which is the difference between those
numbers — the fork's 8.9 GB figure for `mixed` assumes it too. Override
`BREEZE_MLX_CACHE_GB` to trade memory for speed.

`BREEZE_MLX_GROUP` (default 64) tunes the quantized layers: a smaller group size is
more accurate and costs a little memory and time.

`bf16` uses `--chunk-first 1 --chunk-max 4` rather than the narrower bounds the
other profiles set, because wide chunks starve playback on the slowest build. Raise
`BREEZE_CHUNKS` if your machine has headroom.

---

## Limitations and security

Deliberate, upstream design decisions — not bugs:

- **One request at a time.** There is a single GPU and no queueing beyond the
  `.wav` route's 60 s wait. The GUI disables Generate while a request is in flight.
- **No authentication anywhere**, and the API is HTTP. It is bound to loopback and
  must stay there. Do not expose the port.
- **`breeze-run` allowlists the GUI origin, which is a wider posture than CORS-off.**
  Any page served from that origin can drive the API. Only this GUI is served
  there, so the boundary holds — but the flag is deliberately kept visible in one
  place rather than buried in a launcher.
- **The `.wav` GET route is reachable by any web page you visit**, even with CORS
  off, because media requests carry no `Origin` header. Any site can make your Mac
  speak. That is upstream behaviour; the GUI neither causes nor can fix it.

Do **not** launch the server with the fork's own `scripts/start_breeze_mac.sh`. It
hardcodes `--host 0.0.0.0 --cors '*'`, and because `--cors` is `nargs="?"` with a
default of `None`, there is **no way to turn CORS off through that launcher** —
`0.0.0.0` plus CORS `*` lets any device on your network, and any web page, upload
and delete voices and run synthesis. `breeze-run` exists to avoid that.

---

## Licence

The client code in this repository — `index.html`, the scripts in `bin/`, this
README — is **MIT licensed**; see [`LICENSE`](LICENSE) and [`NOTICE`](NOTICE). It is
original work, not derived from the server's source: the client communicates with
the server over HTTP, as a separate program.

That is a code boundary, not a licensing one, and it does not loosen anything:

- **Breeze TTS 2 model weights** — including every MLX conversion used here — are
  governed by the [BreezeBlue Research and Non-Commercial
  License](https://huggingface.co/BreezeBlue/Breeze-TTS-2/blob/main/LICENSE):
  **research and non-commercial use only.** That applies to anything you generate
  with this GUI. It is not granted by, and not affected by, any licence on this
  repo.
- The server (`breezeblue-ai/breeze-tts`, and the
  `Shoofio/breeze-tts2-fast-streaming-api` fork) is Apache-2.0.
- `.opencode/skills/ui-ux-pro-max/` is vendored **unmodified** from
  [nextlevelbuilder/ui-ux-pro-max-skill](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill),
  MIT licensed (© 2024 Next Level Builder); its `LICENSE` is kept alongside it. It is
  a design-time tool only and is never loaded or executed by the GUI.
- The MLX conversions are unofficial community work, not affiliated with or
  endorsed by BreezeBlue.
- You are responsible for the rights to any audio, text or voice you use as input,
  and for how you use the output. Unauthorised voice cloning, impersonation and
  other unlawful uses are prohibited.

Outputs from BreezeBlue's hosted platform or API at <https://breezeblue.ai/> can be
used commercially under an active paid subscription. Self-hosted outputs cannot.

