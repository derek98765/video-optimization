---
name: video-optimization
description: Converts .mov videos in public and src/assets to web-friendly .mp4 (H.264), and compresses .mp4 files that are larger than they need to be, keeping near-original visual quality. Use when you want to reduce video file sizes and improve page load speed.
allowed-tools: Read Glob Grep Bash
---

# Video Optimization Skill

Scan **`public`** and **`src/assets`** for videos and:

1. **Convert every `.mov` to `.mp4`** (H.264 + AAC, web-safe on Chrome, Safari and Firefox).
2. **Compress `.mp4` files that are larger than they need to be**, at near-original visual quality.

Quality comes first: a result that is visibly softer, blockier or stuttery than the source is a failure. When in doubt, keep more bitrate.

## Prerequisites

```bash
brew install ffmpeg   # provides ffmpeg and ffprobe
```

## Steps

### 1. Detect all videos and their properties

```bash
find public src/assets -type f \( -iname "*.mov" -o -iname "*.mp4" \) | sort | while read -r f; do
  info=$(ffprobe -v error -select_streams v:0 \
    -show_entries stream=codec_name,profile,width,height,r_frame_rate,pix_fmt,bit_rate:format=duration,bit_rate,size \
    -of csv=p=0 "$f" | tr '\n' ' ')
  a=$(ffprobe -v error -select_streams a:0 -show_entries stream=codec_name -of csv=p=0 "$f")
  echo "$(du -h "$f" | cut -f1)  $f  [$info]  audio=${a:-none}"
done
```

Record for each file: video codec, resolution, fps, pixel format, duration, overall bitrate, whether it has audio, and whether it is HDR/10-bit (`pix_fmt` like `yuv420p10le`, or `color_transfer` of `smpte2084`/`arib-std-b67`).

Leave `.webm` and files that are already tiny (see rule 1 below) alone.

### 2. Find every usage

Videos are referenced differently by location, so search for both:

- **`public`**: string paths (`"/videos/..."`, `<video src>`, `<source src>`, JSON data, CSS `url()`).
- **`src/assets`**: ES imports (`import hero from "@/assets/images/homepage/hero.mp4"`), `new URL(...)`, `require(...)` and CSS `url(...)`.

```bash
grep -rnE "\.(mov|mp4)" src public/index.html --include="*.jsx" --include="*.tsx" --include="*.js" --include="*.ts" --include="*.css" --include="*.json" --include="*.html"
```

For each usage note:
- **Audio needed?** If every usage is `muted` (autoplay loops, backgrounds, transitions), the audio track can be dropped with `-an`. If any usage plays sound, or usage is unknown, keep audio.
- **Largest rendered size** (CSS px, across breakpoints, counting `object-cover` crops), used only to decide whether the video is oversized (rule 3 below).

### 3. Decide the action per file

| File | Action |
|------|--------|
| `.mov` | **Always convert** to `.mp4`. |
| `.mp4` | **Compress only if** it passes the checks below. Otherwise leave it untouched. |

**Rules for `.mp4` compression** (all must hold):

1. **Skip tiny files**: under ~1 MB, or a bitrate already below ~1 Mbps at 1080p-equivalent. Nothing worth saving.
2. **Skip already-efficient files**: an H.264 encode at roughly CRF 20+ equivalent won't shrink meaningfully. The worthwhile test is the encode itself (step 5): **keep the result only if it is at least 15% smaller** than the original, otherwise discard it and leave the original in place.
3. **Resolution**: never upscale and never change aspect ratio. Only downscale when the video's longest side is more than **1.5×** the largest rendered size × 2 (retina), rounded up to an even number. Within 1.5×, keep the original resolution. Unknown or dynamic usage means keep the original resolution.
4. **Frame rate**: keep the original fps. Never drop frames.
5. Anything that may be downloaded, shown fullscreen, or used as a master/source asset keeps its original resolution.

Show the user a table (file, codec, resolution, size, bitrate, action, reason) before changing anything.

### 4. Encoding settings (near-original quality)

Use constant-quality H.264 so each video gets as many bits as it needs and no more:

```bash
ffmpeg -hide_banner -y -i "$in" \
  -map 0:v:0 -map 0:a:0? \
  -c:v libx264 -crf 18 -preset slow -profile:v high -pix_fmt yuv420p -tag:v avc1 \
  -c:a aac -b:a 160k \
  -movflags +faststart \
  "$out"
```

- **CRF 18** is visually transparent for almost all content. For `.mp4` compression use **CRF 18–20** (start at 18; go to 20 only for very high-bitrate sources, e.g. screen recordings or clean gradients). Never go above 22.
- `-preset slow` gets better compression at the same quality. Use `veryslow` for short clips if time allows.
- `-pix_fmt yuv420p` + `-tag:v avc1` + `+faststart` are required for Safari/iOS compatibility and instant playback.
- Add `-an` (drop audio) only when step 2 showed every usage is muted. Otherwise keep the audio, re-encoding to AAC only if it isn't already AAC (`-c:a copy` when it is).
- Add `-vf "scale=<w>:-2:flags=lanczos"` only for the downscale case in rule 3 (`-2` keeps the height even and the aspect ratio exact).
- Don't set `-r`, `-vsync`, or trim anything. Duration and frame rate must match the source.

**`.mov` shortcut (no re-encode):** if the `.mov` video is already H.264 (`yuv420p`) and the audio is AAC or absent, remux instead of re-encoding. This is lossless and instant:
```bash
ffmpeg -hide_banner -y -i in.mov -c copy -movflags +faststart -tag:v avc1 out.mp4
```
Then apply the 15% compression check only if the result is large.

**HEVC / ProRes / 10-bit / HDR sources** (typical for iPhone `.mov`): re-encode to H.264 with the settings above. For 10-bit or HDR sources, **do not** just force `yuv420p`, which washes out colors. Tonemap to SDR first (`-vf zscale=t=linear:npl=100,format=gbrpf32le,zscale=p=bt709:t=bt709:m=bt709:r=tv,format=yuv420p`) or flag the file to the user and skip it. Also copy color tags: `-colorspace bt709 -color_primaries bt709 -color_trc bt709`.

**iPhone rotation / variable frame rate:** ffmpeg applies rotation metadata automatically. For variable-frame-rate phone footage, keep `-fps_mode passthrough` so timing isn't altered.

### 5. Encode to a temp file, then verify before replacing anything

Encode to `name.optimized.mp4` (never in place), then check:

```bash
# Duration within 0.1s and same resolution/fps (unless deliberately downscaled)
ffprobe -v error -show_entries format=duration -of csv=p=0 "$in" "$out"

# Objective quality check: SSIM should be >= 0.98 (PSNR > 40 dB) vs the source
ffmpeg -hide_banner -i "$out" -i "$in" -lavfi "[0:v][1:v]scale2ref=flags=bicubic[a][b];[a][b]ssim" -f null - 2>&1 | grep SSIM
```

Replace the original only if **all** pass:
- duration matches (±0.1 s) and frame count/fps match,
- SSIM ≥ 0.98 (if lower, re-encode with a lower CRF, e.g. 16–17),
- for `.mp4` compression, the output is **≥ 15% smaller**; otherwise delete the temp file and leave the original untouched.

For `.mov` conversions, the size gate doesn't apply (the point is the container/codec), but still report if the `.mp4` came out larger than the `.mov`.

Spot-check one or two frames visually when the video has fine detail, text, or gradients:
```bash
ffmpeg -ss 00:00:01 -i "$out" -frames:v 1 /tmp/check.png
```

### 6. Update source references

- **`.mov` → `.mp4`**: update static paths, ES import specifiers (`import hero from "@/assets/.../hero.mov"` → `.mp4`), `new URL(...)`, `require(...)`, and CSS `url()`. Update `<source type="video/quicktime">` to `type="video/mp4"`.
- **Dynamic paths built from data**: swap the extension at runtime:
  ```js
  `/videos/${clip.file.replace(/\.mov$/i, ".mp4")}`
  ```
- **Compressed `.mp4` files keep the same name**, so no reference changes are needed.
- Vite fingerprints imports, so a missed one fails the build. Run `yarn build` (or `yarn tsc --noEmit`) afterwards.

### 7. Clean up

Rename `name.optimized.mp4` to the final name (replacing the original `.mp4`), delete converted `.mov` originals only after their `.mp4` verified, and delete leftover temp files.

```bash
find public src/assets -type f \( -iname "*.mov" -o -iname "*.optimized.mp4" \)
grep -rnE "\.mov" src --include="*.jsx" --include="*.tsx" --include="*.js" --include="*.ts" --include="*.css"
```

Both should come back empty (aside from `.mov` files deliberately skipped, which must be reported).

Finish with a before/after table: file, old size, new size, % saved, SSIM.

## Checklist (use TodoWrite)

- [ ] Detect all `.mov` / `.mp4` in `public` and `src/assets` with codec, resolution, fps, bitrate, audio
- [ ] Find every usage (string paths and ES imports); note which are muted and rendered sizes
- [ ] Show the plan table before changing anything
- [ ] Remux H.264 `.mov` losslessly; otherwise re-encode at CRF 18, `yuv420p`, `+faststart`
- [ ] Compress `.mp4` only if the result is ≥ 15% smaller
- [ ] Never change fps, duration or aspect ratio; never upscale
- [ ] Verify duration and SSIM ≥ 0.98 before replacing anything
- [ ] Update `.mov` references (imports, paths, `type=`, dynamic paths)
- [ ] Delete verified originals and temp files
- [ ] Run `yarn build` / typecheck and report the before/after table

## Notes

- Always encode to a temp file first. A bad encode must never overwrite the only copy.
- Don't convert `.webm`. It is already web-friendly.
- Don't re-encode an already-compressed `.mp4` repeatedly. Every generation loses quality, so skip files that were obviously optimized already (small, low bitrate).
- If a file fails the quality check even at CRF 16, leave the original and tell the user.
- Prefer a larger-than-needed result over a visibly degraded one.
