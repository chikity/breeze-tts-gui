# Breeze TTS 2 — web GUI for Apple Silicon

A local web GUI and a small set of launcher scripts for running
[Breeze TTS 2](https://huggingface.co/BreezeBlue/Breeze-TTS-2) on an Apple Silicon
Mac through the MLX backend.

Nothing here is a fork of anything. This repo is a **client**: a single static HTML
file that talks to a separately-installed Breeze TTS server over HTTP. The server
lives in its own checkout and is never modified, so it can be updated freely
without these files drifting.

Measured on an M5 Pro (18-core CPU, 20-core GPU, 64 GB) with the default `mixed`
weights: **~126–158 ms to first audio, real-time factor 0.54–0.58** — comfortably
faster than real time, so it streams without stalling.

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

---

## Setup

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
ln -sfn "$PWD/breeze-tts-gui/bin/breeze-run" ~/.local/bin/breeze-run
ln -sfn "$PWD/breeze-tts-gui/bin/breeze-say" ~/.local/bin/breeze-say
ln -sfn "$PWD/breeze-tts-gui/bin/breeze-gui" ~/.local/bin/breeze-gui
```

---

## Running

Two terminals:

```bash
breeze-run     # the TTS server on 127.0.0.1:8080 — leave it running
breeze-gui     # the GUI on 127.0.0.1:8090 — opens your browser
```

Then open <http://127.0.0.1:8090/>. `breeze-gui` checks that the API is up and
tells you to run `breeze-run` first if it isn't.

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

- **Script** — what to say. The chips insert **vocal events**: `(sigh)`,
  `(laugh)`, `(cough)`, `(clears throat)` in English, `[叹气]`, `[笑]`, `[咳嗽]`,
  `[清嗓子]` in Chinese. Match the event language to the text language.
- **Voice** — `No reference` is voice *design* (pick a voice from a description).
  `Saved voice` reuses something in the library. `Upload a reference` is voice
  *clone*, and needs the reference clip plus its **exact transcript**.
  `Record a reference` does the same from your microphone, and writes the
  transcript for you — see below.
- **Instruction** — optional. Adding one to a reference is what makes it voice
  *direction* rather than a plain clone.
- **cfg_scale** — `1` is plain. Higher follows the instruction more strongly;
  voice design and direction usually want 2–4.
- **Seed** — same seed and settings reproduce the same take.
- **Output** — plays while it generates, `Download WAV`, and reads out first-audio
  time, audio length, generation time and real-time factor.
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

The files in **this** repo (`index.html`, the scripts in `bin/`, this README) are
original work and are not derived from the server's source — they communicate with
it over HTTP, as a separate program.

That is a code boundary, not a licensing one, and it does not loosen anything:

- **Breeze TTS 2 model weights** — including every MLX conversion used here — are
  governed by the [BreezeBlue Research and Non-Commercial
  License](https://huggingface.co/BreezeBlue/Breeze-TTS-2/blob/main/LICENSE):
  **research and non-commercial use only.** That applies to anything you generate
  with this GUI. It is not granted by, and not affected by, any licence on this
  repo.
- The server (`breezeblue-ai/breeze-tts`, and the
  `Shoofio/breeze-tts2-fast-streaming-api` fork) is Apache-2.0.
- The MLX conversions are unofficial community work, not affiliated with or
  endorsed by BreezeBlue.
- You are responsible for the rights to any audio, text or voice you use as input,
  and for how you use the output. Unauthorised voice cloning, impersonation and
  other unlawful uses are prohibited.

Outputs from BreezeBlue's hosted platform or API at <https://breezeblue.ai/> can be
used commercially under an active paid subscription. Self-hosted outputs cannot.

