---
name: douyin-transcript
description: Turn a Douyin (抖音) video link into a clean, readable Chinese markdown transcript -- fully local, free, on Apple Silicon macOS. The user pastes a Douyin link or the full "复制打开抖音…" share text; the skill downloads the video, extracts audio, transcribes locally with Whisper large-v3 (Apple Neural Engine via whisperkit-cli), light-cleans the raw ASR into readable prose (fixing the homophone / proper-noun errors Chinese Whisper makes, using the video's own title/topic as context), saves ONE markdown file to an output folder, deletes the downloaded media, and stops. MUST trigger when the user provides a Douyin / 抖音 link or share-text, or says "抖音转写", "转写这个抖音", "把这个抖音转成文字", "抖音转文字", "抖音 transcript", "douyin transcript", "transcribe this douyin", "这个抖音讲什么", "douyin-transcript", or pastes a v.douyin.com / douyin.com/video link. Not for YouTube or for local meeting-audio files.
---

# Douyin Transcript -- 抖音链接 → 干净中文逐字稿

## What this skill does

Given a Douyin link (or pasted share-text), produce a readable Chinese transcript and save it as one markdown file:

1. Resolve the URL out of the pasted text (handles `v.douyin.com` short links + full `douyin.com/video/…` + the `复制打开抖音…` blob)
2. Download metadata + extract 16kHz mono WAV (`yt-dlp` + `ffmpeg`)
3. Transcribe locally with **Whisper large-v3** via `whisperkit-cli` (Apple Neural Engine, `--language zh`)
4. **Light-clean** the raw ASR into readable prose -- punctuation, paragraphs, and fixing obvious homophone / proper-noun errors **using the video's own title + description as topic context** (faithful, never inventing content)
5. Save **one** markdown file to the output folder
6. **Delete the downloaded media** (the temp work dir) -- only the `.md` survives
7. **Stop and await the next instruction** (no auto-summary, no auto-repurpose)

Pipeline is 100% local and free (~12s for a 44s clip on Apple Silicon). Same local-Whisper lineage as `meeting-scribe`; Douyin just adds the yt-dlp front step.

## When to load

The user pastes a Douyin link / share-text, or asks to transcribe / "讲什么" a 抖音 video. One link = one run.

## Requirements (Mac-only)

Runs on **Apple Silicon macOS**. Transcription is `whisperkit-cli` (Apple Neural Engine); the front steps use `yt-dlp` + `ffmpeg` + `python3`. Install with Homebrew:

```bash
brew install whisperkit-cli ffmpeg yt-dlp
```

Not portable to Intel mac / Windows / Linux -- if you need that, use a cross-platform Whisper engine (e.g. faster-whisper / whisper.cpp) instead.

> Throughout, `<skill-dir>` is this skill's install directory (where this SKILL.md lives, e.g. `~/.claude/skills/douyin-transcript`). The two helper scripts ship in `<skill-dir>/scripts/`.

---

## Phase 0 -- Ensure the Whisper model is present (first run only)

Transcription needs the **large-v3** CoreML model on disk (~1.5 GB). Before the first run, confirm it exists -- scan, don't assume:

```bash
bash "<skill-dir>/scripts/ensure-model.sh"
```

- `MODEL_PATH=<dir>` -- a model was found and is reused. Export it so Phase 1 uses it directly:
  ```bash
  export WHISPER_MODEL_PATH="<dir>"
  ```
  (If you forget, `dl-transcribe.sh` re-scans via `ensure-model.sh` and still finds it -- the export just skips that.)
- `MODEL_NOT_FOUND` -- no large-v3 model anywhere. **Ask the user before downloading** (a ~1.5 GB Hugging Face fetch, a few minutes). On their OK:
  ```bash
  bash "<skill-dir>/scripts/ensure-model.sh" --download
  ```
  Downloads the `openai_whisper-large-v3-v20240930` snapshot, then prints `MODEL_PATH=<dir>` -- export it as above. (Choose where it lands with a positional dir after `--download`, or the `WHISPER_MODELS_DIR` env var; default `~/.cache/whisperkit-models`.)

Once present, later runs find it via the scan and skip straight to Phase 1.

---

## Phase 1 -- Download + transcribe (one command)

Run the helper (it does pre-flight, resolve, download, audio extract, Whisper, and prints everything):

```bash
bash "<skill-dir>/scripts/dl-transcribe.sh" "<paste the Douyin link or full share text here>"
```

The script prints:
- `WORKDIR=/tmp/douyin-transcript.XXXXXX` -- the temp dir (remember it for cleanup)
- `----META----` … a JSON block (title / uploader / duration / upload_date / counts / webpage_url)
- `----RAW----` … the raw verbatim ASR transcript (one block, tokens stripped)
- `----END----`

**If it prints an error marker instead**, handle and stop:
- `MISSING_TOOL=<t>` -- a binary is absent (`yt-dlp` / `ffmpeg` / `whisperkit-cli` / `python3`); tell the user the Homebrew install line from Requirements.
- `MISSING_MODEL=<path>` -- the model isn't at the resolved path; go back to **Phase 0** (`ensure-model.sh`) to locate or download it, then re-run with `WHISPER_MODEL_PATH` exported.
- `NO_URL` -- couldn't find a URL in the input; ask the user to re-paste the link.
- `DOWNLOAD_FAILED` -- yt-dlp couldn't fetch it (Douyin a_bogus / risk-control). The script already retried with cookies from several browsers (chrome / safari / firefox / edge / arc / brave). Tell the user to open the video once in **their logged-in browser** then retry (optionally `DOUYIN_COOKIE_BROWSER=<that-browser>`), update `yt-dlp`, or use a parse-API fallback; don't fake a transcript.
- `TRANSCRIBE_FAILED` -- the download worked but whisperkit-cli failed or produced no text (corrupt/partial model, OOM, or a clip with no speech). The script keeps a `whisper.log` in the printed WORKDIR; point the user there. Often a re-run of Phase 0 `--download` (model didn't fully assemble) fixes it.

> The script also prints `WORKDIR=` on these failure paths, so you can inspect logs and clean up. Don't pre-empt with cookies unless the first try fails.

---

## Phase 2 -- Light-clean into readable markdown

Take the `----RAW----` text and produce a **readable** version. This is a judgment pass, not a rewrite:

- **Add punctuation and paragraph breaks.** Group by idea; short clip = a few short paragraphs.
- **Fix obvious ASR errors using the META title/description as topic context** -- Chinese Whisper mangles homophones (谐音) and proper nouns. Typical failure pattern: `Kithub→GitHub`, `cloud.md→CLAUDE.md`, `卡帕西→Karpathy`, `采过→踩过`. Correct only what's clearly wrong given the topic.
- **Preserve meaning and code-switching** (中英混排 as spoken). Do **not** add facts, opinions, or summary that aren't in the audio.
- **Flag genuine uncertainty** with `[方括号]` and one short 校对说明 line listing the fixes you made + any guesses. If the title is clickbait that contradicts the spoken content (e.g. 40万 vs 4.4万), note it.
- Keep it faithful enough that the raw meaning is intact -- this is the base for the user's *next* step, not a finished artifact.

### File to write

Output folder: `${DOUYIN_OUTPUT_DIR:-$HOME/Downloads/douyin-transcripts}` (`mkdir -p` it first).

Path: `<output-folder>/<YYYY-MM-DD>-douyin-<title-slug>.md` (today's date; slug = title kebab-cased, drop emoji / `#hashtags` / punctuation, ~6 words max).

Formatting: prefer `--` over em-dashes, preserve 中英混排, hyphen-no-space filename.

Template:

```markdown
---
title: <video title>
date: <YYYY-MM-DD>
type: douyin-transcript
source: <webpage_url>
uploader: <uploader>
duration: <duration>s
status: awaiting-next-step
tags: [douyin, transcript]
---

# <video title>

- **Source:** <webpage_url>
- **Uploader / date:** <uploader> · <upload_date>
- **Duration:** <duration>s · 点赞 <like_count> · 转发 <repost_count>

## 逐字稿（轻清洗版）

<cleaned, punctuated, paragraphed Chinese transcript>

> **校对说明：** <one line: what homophone/proper-noun fixes were made; any [bracketed] guesses; title-vs-content mismatch if any>
```

---

## Phase 3 -- Clean up + hand off

1. Remove the downloaded media (only the temp work dir from this run):
   ```bash
   rm -rf "<the WORKDIR printed in Phase 1>"
   ```
   Only ever `rm -rf` the `/tmp/douyin-transcript.XXXXXX` path the script printed -- never the skill folder or any other path.
2. Report to the user: the output file path + a one-line gist of what the video is about.
3. **Stop. Await the next instruction.** Don't auto-summarize, translate, or repurpose unless asked.

---

## Notes / gotchas

- `whisperkit-cli` must use `--model-path <local model>`; bare `--model large-v3` hangs downloading from Hugging Face during a transcribe run. The scripts pin the local model path, so this is handled.
- Douyin's `music` / `audio` URL is background BGM, not the voiceover -- must download the MP4 and extract the mixed track (the script does `-x`).
- Douyin exposes no native subtitle track (`--list-subs` = empty), so ASR is mandatory.
- yt-dlp's Douyin extractor is occasionally broken by risk-control changes; fresh browser cookies / a recent `yt-dlp` usually fixes it.
- Everything stays on your machine. The only network access is the Douyin download and the one-time model fetch from Hugging Face.
