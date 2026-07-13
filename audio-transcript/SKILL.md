---
description: Transcribe an audio or video file to plain text locally with whisper.cpp, and print the transcript into the session so the agent can work with it
argument-hint: <audio-or-video-file> [model] [lang]
---

# Audio Transcript

Given a path to an audio or video file, produce a clean plain-text transcript and print it into the session so the agent can work with it (summarize, extract tasks, answer questions about the content).

Everything runs **locally** — the audio never leaves the machine. Transcription is done by [whisper.cpp](https://github.com/ggml-org/whisper.cpp) (the Whisper speech-to-text model). `ffmpeg` is only a preprocessing step: it normalizes any input format into the 16 kHz mono WAV that whisper.cpp expects — `ffmpeg` itself does not transcribe.

## Inputs

- `$1` — path to the audio/video file (required). Supports anything ffmpeg can decode: `.m4a`, `.mp3`, `.wav`, `.ogg`, `.flac`, `.mov`, `.mp4`, `.mkv`, ... For video, the audio track is extracted automatically.
- `$2` / `model` (optional) — Whisper model, defaults to `large-v3`. Alternatives for speed: `medium`, `small`, `base`. Quality is determined by the model size, not by anything else — bigger = more accurate, especially for Russian endings, names, and terms.
- `$3` / `lang` (optional) — language code (`ru`, `en`, ...). Defaults to `auto` (whisper auto-detects). Fix it explicitly for short or noisy clips where auto-detect may guess wrong.

If `$1` is missing, ask the user for the file path.

## Environment

Paths this skill assumes:

- Model dir: `~/.local/share/whisper-cpp/` — a durable data location (XDG data home), **not** `~/.cache`. The model is a ~3 GB asset that is slow to re-download, so it must not live somewhere that cache-cleaners purge.
- Model file for a given model name: `~/.local/share/whisper-cpp/ggml-<model>.bin` (e.g. `ggml-large-v3.bin`)
- Temp WAV: written to the system temp dir, deleted after transcription

## Workflow

### 0. Preflight — check tools and model

Run these checks. Never install anything silently — if something is missing, print the exact command and ask the user to run it (or offer to run it for them).

1. **ffmpeg** — `command -v ffmpeg`. If missing: `brew install ffmpeg`.
2. **whisper-cli** — `command -v whisper-cli`. If missing: `brew install whisper-cpp`.
3. **Model file** — check `~/.local/share/whisper-cpp/ggml-<model>.bin` exists. If missing, download once (warn about size — `large-v3` is ~3.1 GB):

   ```bash
   mkdir -p ~/.local/share/whisper-cpp
   curl -L --fail -o ~/.local/share/whisper-cpp/ggml-<model>.bin \
     https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-<model>.bin
   ```

4. **Input file** — verify `$1` exists and is readable. If not, stop with a clear error.

### 1. Normalize audio with ffmpeg

Convert the input to 16 kHz mono 16-bit WAV in a temp file. This works uniformly for audio files and for video (extracts and downmixes the audio track):

```bash
TMPWAV="$(mktemp -t audio-transcript).wav"
ffmpeg -nostdin -y -i "$1" -ar 16000 -ac 1 -c:a pcm_s16le "$TMPWAV" 2>ffmpeg.log
```

If ffmpeg exits non-zero, the file is likely corrupt or not a media file — show the tail of `ffmpeg.log` and stop.

### 2. Transcribe with whisper.cpp

```bash
whisper-cli \
  -m ~/.local/share/whisper-cpp/ggml-<model>.bin \
  -f "$TMPWAV" \
  -l <lang> \
  -nt \
  -np
```

Flags:
- `-nt` — no timestamps → clean running text (this is what we want; timestamps are intentionally not produced).
- `-np` — no progress prints, keeps stdout clean.
- `-l auto` — auto-detect language, or the fixed code from `lang`.

whisper.cpp handles long files natively (it processes internally in ~30 s windows), so no manual chunking is needed — long recordings just work.

The transcript is whisper-cli's stdout.

### 3. Clean up

Delete the temp WAV (`rm -f "$TMPWAV"`) and the `ffmpeg.log`. The original file is never modified.

### 4. Output

Print the transcript into the session as plain text, prefixed with a short header:

```
Transcript of <filename> (model: <model>, lang: <detected-or-fixed>):

<full transcript text>
```

Then the agent continues with whatever the user actually wanted (summary, action items, translation, Q&A). If the user only asked to transcribe, stop here — the transcript is the deliverable.

## Error handling

- **File not found / unreadable** — stop with the path and the reason.
- **ffmpeg decode failure** — show the tail of `ffmpeg.log`; the input is probably not valid media.
- **Empty transcript** — if whisper returns nothing, tell the user no speech was recognized (silence, music-only, or wrong language guess — suggest fixing `lang`).
- **Large file** — for very long recordings, tell the user it will take a while (large-v3 on Metal is roughly real-time-ish, so a 1-hour file ≈ tens of minutes). Don't block or refuse; just set the expectation.

## Rules

- **Local only.** Never upload the audio anywhere. This is the whole point of the skill.
- **Never install silently.** Missing `whisper-cli`, `ffmpeg`, or a model → print the command and ask first.
- **Don't touch the source file.** All work happens on a temp WAV; clean it up afterward.
- **No timestamps by default.** The deliverable is clean text. (If a task ever needs timecodes, run whisper-cli without `-nt`, but that is out of scope here.)
- **Quality lives in the model.** If the user complains about accuracy, the lever is a bigger model (`large-v3`) or fixing `lang` — not a different backend.
- **Don't dump huge transcripts blindly if the user's real goal is a summary.** Transcribe, then go straight to what they asked for; keep the raw text available but lead with the answer.
