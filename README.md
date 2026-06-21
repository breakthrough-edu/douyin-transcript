# douyin-transcript

> [!NOTE]
> **Superseded by [`social-video-transcript`](https://github.com/breakthrough-edu/social-video-transcript)** -- same local-Whisper engine, now covering Douyin **+ Xiaohongshu (小红书) + Instagram + Facebook + TikTok**, with auto-detected language. This repo still works, but new work happens there:
> ```bash
> npx skills add breakthrough-edu/social-video-transcript
> ```

Turn a **Douyin (抖音)** video link into a clean, readable **Chinese markdown transcript** -- fully local, free, on Apple Silicon macOS.

Paste a Douyin link (or the full `复制打开抖音…` share text). The skill downloads the video, transcribes it locally with **Whisper large-v3** on the Apple Neural Engine, light-cleans the raw speech-to-text into readable prose (fixing the homophone / proper-noun mistakes Chinese Whisper makes, using the video's own title as context), and saves one `.md` file. Nothing leaves your machine.

This is a [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) skill.

## Install

```bash
npx skills add breakthrough-edu/douyin-transcript
```

Then, in Claude Code, paste a Douyin link and it takes over.

## Requirements (Apple Silicon macOS only)

Transcription uses [`whisperkit-cli`](https://github.com/argmaxinc/WhisperKit) (CoreML / Apple Neural Engine), so this runs on **Apple Silicon Macs**. Install the tools with [Homebrew](https://brew.sh/):

```bash
brew install whisperkit-cli ffmpeg yt-dlp
```

| Tool | Why |
|---|---|
| `whisperkit-cli` | local Whisper transcription on the Neural Engine |
| `ffmpeg` | extract + resample the audio to 16kHz mono WAV |
| `yt-dlp` | resolve + download the Douyin video |
| `python3` | strip ASR tokens, collect metadata (Homebrew pulls it in automatically as a `yt-dlp` dependency) |

On first use the skill checks for the **large-v3** model (~1.5 GB). If you don't have it, it asks before downloading `openai_whisper-large-v3-v20240930` from Hugging Face into `~/.cache/whisperkit-models` (override with `WHISPER_MODELS_DIR`). If you already have a large-v3 CoreML model (e.g. from `meeting-scribe`), it is detected and reused.

> Not on Apple Silicon? `whisperkit-cli` won't run. Use a cross-platform Whisper engine (faster-whisper, whisper.cpp) instead.

## Usage

Paste any of these and the skill runs:

- a short link -- `https://v.douyin.com/XXXXXXXX/`
- a full link -- `https://www.douyin.com/video/123…`
- the whole share blob -- `… 复制打开抖音… https://v.douyin.com/XXXXXXXX/ …`

It downloads, transcribes (~12s for a 44s clip), light-cleans, and writes the file -- then stops and waits for your next instruction (it does **not** auto-summarize or repurpose).

## What you get

One markdown file in `~/Downloads/douyin-transcripts/` (override with `DOUYIN_OUTPUT_DIR`):

```markdown
---
title: 普通人也能玩转AI的法宝
date: 2026-05-13
type: douyin-transcript
source: https://www.douyin.com/video/7638865717618042619
uploader: 抖音精选
duration: 127s
status: awaiting-next-step
tags: [douyin, transcript]
---

# 普通人也能玩转AI的法宝

- **Source:** https://www.douyin.com/video/7638865717618042619
- **Uploader / date:** 抖音精选 · 20260513
- **Duration:** 127s · 点赞 33769 · 转发 316

## 逐字稿（轻清洗版）

我敢说，新手学 AI 最没用的，就是那些满是黑话的教程……

> **校对说明：** 用标题作语境修正了明显 ASR 谐音错误；不确定的账号名已用 [方括号] 标出。
```

The transcript is a faithful, lightly-cleaned base for whatever you do next (summary, notes, repurposing) -- not an over-polished rewrite.

## Configuration

| Env var | Default | What |
|---|---|---|
| `DOUYIN_OUTPUT_DIR` | `~/Downloads/douyin-transcripts` | where the `.md` is saved |
| `WHISPER_MODEL_PATH` | (auto-detected) | explicit path to a large-v3 CoreML model dir |
| `WHISPER_MODELS_DIR` | `~/.cache/whisperkit-models` | where to keep / download models |
| `WHISPER_LANG` | `zh` | Whisper language hint |
| `DOUYIN_COOKIE_BROWSER` | (tries several) | force one browser for the download cookie retry |

## When Douyin blocks the download

Douyin occasionally trips its `a_bogus` / risk-control on `yt-dlp`. The skill already retries with cookies from your browsers (it tries Chrome, Safari, Firefox, Edge, Arc, Brave in turn). If it still fails:

- open the video once in your logged-in browser, then retry -- this refreshes the cookies. To pin one browser, set `DOUYIN_COOKIE_BROWSER=safari` (or `chrome` / `firefox` / `edge` / `arc` / `brave`);
- make sure `yt-dlp` is recent (`yt-dlp -U` or `brew upgrade yt-dlp`) -- Douyin's extractor is patched often.

## Privacy

Everything runs on your machine. The video, the audio, and the transcript never leave it. The only network access is the Douyin download and the one-time model fetch from Hugging Face.

## License

MIT -- see [LICENSE](LICENSE).
