---
name: compress-video
description: Compress existing opaque videos with the user's standard MP4/H.264 profile while preserving resolution, frame rate and duration. Remove audio by default; preserve it when explicitly requested. Use for compression, not animation editing or export from design apps.
---

# Compress video

Execute compression and deliver the verified file. This skill is self-contained and requires no conversation history.

## Standard profile

- MP4 container; H.264 video via `libx264`.
- `-crf 23 -preset slow -pix_fmt yuv420p -movflags +faststart`.
- Preserve source dimensions, frame rate and full duration. Do not add scaling, `-r`, trimming or speed changes unless requested.
- **Default: completely remove audio**, using `-map 0:v:0 -an`. Muting playback or encoding silence is not equivalent.
- If the user explicitly requests sound, map audio and use `-c:a copy` for MP4-compatible audio. This preserves encoded audio bit for bit. If incompatible, encode AAC at 128 kb/s and disclose the re-encoding. Do not claim audio was preserved when the source has none.
- Explicit user parameters override defaults. CRF 23 is lossy video compression: do not promise mathematically lossless video or a fixed file size.

## Workflow

1. Inspect the actual local source: video/audio streams, dimensions, frame rate, duration and file size. Wait for downloads to finish. Use existing FFmpeg/FFprobe or an available trusted runtime; do not hardcode another machine's temporary binary path. Request the source if unavailable.
2. Use a new filename such as `<source-stem>-compressed.mp4`, at the requested destination or beside the source when unspecified. Preserve source and existing outputs with `-n`. Quote paths and avoid shell interpolation of filenames.
3. Encode using the standard profile and selected audio mode. Do not silently reduce resolution/FPS or increase CRF to chase a smaller size. Report if output is larger instead of claiming a reduction.
4. Wait for successful completion. Inspect output streams and fully decode the file to check for errors. Confirm dimensions/FPS and duration match the source, allowing normal container rounding within one frame. Default mode must have **zero audio streams**. For copied audio, compare hashes of encoded audio streams. Check representative frames for orientation, clipping and visible artifacts.
5. Deliver a clickable file link, source → output sizes, and any material tradeoff. State whether audio was removed, copied unchanged or re-encoded. Do not delete or relocate the source.

## Commands

Replace example paths with actual paths. Run only the chosen audio mode.

Default — no audio stream:

```bash
ffmpeg -hide_banner -nostdin -n -i "input.mp4" \
  -map 0:v:0 -c:v libx264 -crf 23 -preset slow \
  -pix_fmt yuv420p -an -movflags +faststart "output-compressed.mp4"
```

Explicitly preserve sound:

```bash
ffmpeg -hide_banner -nostdin -n -i "input.mp4" \
  -map 0:v:0 -map '0:a?' -c:v libx264 -crf 23 -preset slow \
  -pix_fmt yuv420p -c:a copy -movflags +faststart "output-compressed.mp4"
```

Inspect source and output independently, then decode the output:

```bash
ffprobe -v error -show_streams -show_format -of json "output-compressed.mp4"
ffmpeg -v error -nostdin -i "output-compressed.mp4" -f null -
```

For copied audio, run on both files and compare results. Select each audio stream separately when several exist:

```bash
ffmpeg -v error -i "output-compressed.mp4" -map 0:a:0 \
  -c:a copy -f hash -hash sha256 -
```

This profile targets opaque SDR videos. When transparency, HDR or multiple video streams must be preserved, choose the appropriate output/profile before converting; ordinary H.264/yuv420p does not preserve alpha.
