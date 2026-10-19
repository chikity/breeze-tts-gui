# Breeze TTS 2 — web GUI for Apple Silicon

A local web GUI and a small set of launcher scripts for running
[Breeze TTS 2](https://huggingface.co/BreezeBlue/Breeze-TTS-2) on an Apple Silicon
Mac through the MLX backend.

Nothing here is a fork of anything. This repo is a **client**: a single static HTML
file that talks to a separately-installed Breeze TTS server over HTTP. The server
lives in its own checkout and is never modified, so it can be updated freely
without these files drifting.

Measured on an M5 Pro (18-core CPU, 20-core GPU, 64 GB) with the 8-bit weights:
**~200 ms to first audio, real-time factor 0.49–0.55** — roughly twice faster than
real time, so it streams without stalling.

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
  backend refuses to start below that. 8-bit weights need roughly 7–13 GB.
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

Pinned to a revision so a re-upload can't change the weights under you:

```bash
uvx --from huggingface_hub hf download mlx-community/Breeze-TTS-2-mlx-8bit \
  --revision c6e4a2ff6ab9afba68b7853de802273ffe23fb49
```

That's 4.6 GB into the standard Hugging Face cache (`~/.cache/huggingface/hub`),
which is where the server looks.

> **Downloading was slow from Hong Kong** (~0.5 MB/s, roughly 2 hours). The xet
> backend's adaptive concurrency controller collapses to a single connection and
> never recovers — its log filled with 760 repeats of
> `Decreased concurrency from 1 to 1`. Pinning it fixes the decay:
> `HF_XET_FIXED_DOWNLOAD_CONCURRENCY=4 hf download ...` produced zero such events
> and a steady rate. It does not raise peak throughput; your link is the ceiling.

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
| `BREEZE_GUI_PORT` | `8090` — GUI port (and the allowlisted origin) |
| `BREEZE_CORS` | the GUI origin on both `127.0.0.1` and `localhost` |
| `BREEZE_REPO` | `~/Github/breeze-tts2-fast-streaming-api` |
| `BREEZE_MODEL` | the pinned 8-bit snapshot path |
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

Measured here on an M5 Pro, 8-bit MLX weights, 24 kHz output:

| Mode | First audio | Real-time factor |
| --- | --- | --- |
| Plain text | ~200 ms | 0.49–0.51 |
| Voice design / direction (CFG 2–4) | ~210–460 ms | 0.52–0.53 |
| Voice clone (inline reference) | ~590 ms | 0.62 |

Real-time factor below 1 means audio is produced faster than it plays. The GUI's
"first audio" is measured client-side as the moment the first audio bytes arrive,
which is the same quantity the model's own figures describe; it deliberately does
**not** wait for playback to start, which would fold in browser audio-device
startup.

### Optional: better voice-direction fidelity

The 8-bit build quantizes the backbone as well as the depth decoder. A *mixed*
build — bf16 backbone, int8 depth decoder — keeps the backbone bit-exact and
measures faster still (~139 ms first audio on an M4 Pro, backbone KL of 0).
It needs a 7.6 GB download, though the `audio_tokenizer` dedupes against the 8-bit
cache:

```bash
uvx --from huggingface_hub hf download mlx-community/Breeze-TTS-2-mlx \
  --revision 3c8829fb7fd335818f085cd2ef49b4100c0e46c8
```

Then run with the bf16 snapshot and `--precision mixed`-equivalent settings
(`BREEZE_MLX_QUANT=depth:8`, `BREEZE_MLX_COMPILE=1`, `BREEZE_MLX_FAST_FIRST=6`,
`BREEZE_MLX_CACHE_GB=1`, `--chunk-first 1 --chunk-max 1`), or point
`BREEZE_MODEL` at the pre-built [`Shoof/Gust-MLX`](https://huggingface.co/Shoof/Gust-MLX)
checkpoint.

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

