# video-optimization

A Claude Code skill that converts `.mov` videos to web-friendly `.mp4` and
compresses oversized `.mp4` files, keeping near-original visual quality. It
scans both `public` and `src/assets`.

## What's in this repo

```
skills/
  video-optimization/SKILL.md
```

Install by copying `skills/video-optimization` into `~/.claude/skills/`
(or a project's `.claude/skills/`).

## Prerequisites

- [`ffmpeg`](https://ffmpeg.org/) (includes `ffprobe`):
  ```bash
  brew install ffmpeg
  ```

## What it does

1. **Detects** every `.mov` / `.mp4` under `public` and `src/assets`, with
   codec, resolution, fps, bitrate and audio.
2. **Finds every usage** (string paths and ES imports) to learn whether the
   video is always `muted` and how large it renders.
3. **Converts `.mov` to `.mp4`**: lossless remux when already H.264/AAC,
   otherwise H.264 at CRF 18 with `yuv420p` and `+faststart`.
4. **Compresses `.mp4` only when worthwhile**: the result is kept only if it
   is at least 15% smaller; never upscales, never changes fps, duration or
   aspect ratio.
5. **Verifies before replacing**: duration matches and SSIM >= 0.98 against
   the source.
6. **Updates references** (imports, paths, `type=`, dynamic paths), deletes
   verified originals, and reports a before/after table.

HDR / 10-bit sources are tonemapped to SDR or flagged and skipped. `.webm`
files are never touched.

## How to invoke

```
/video-optimization
```

Or ask in conversation, e.g. "optimize the videos in this project".
