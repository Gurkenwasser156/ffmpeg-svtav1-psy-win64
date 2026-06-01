# FFmpeg (Windows x64) — SVT-AV1-PSY + Opus

Standalone, **statically linked** FFmpeg for Windows 64-bit with the
**SVT-AV1-PSY** encoder ([5fish/svt-av1-psy](https://github.com/5fish/svt-av1-psy))
and **libopus** built in. No DLLs, no install — just run `ffmpeg.exe`.

Built automatically by GitHub Actions (see `.github/workflows/build.yml`).
Grab the latest `ffmpeg-svtav1-psy-win64.zip` from the
[Releases](../../releases/tag/latest) page or the workflow artifacts.

## Contents
- `ffmpeg.exe` — encoder/transcoder (SVT-AV1-PSY is in here)
- `ffprobe.exe` — file/stream analysis

## Versions
- FFmpeg: `n7.1`
- SVT-AV1-PSY: `5fish/svt-av1-psy` (2.3.0-C fork, branch `main`)
- libopus: `1.5.2`
- Toolchain: mingw-w64 GCC, static (SSE2…AVX512)

## Quick start

AV1 video + Opus audio:

```bat
ffmpeg.exe -i input.mp4 -c:v libsvtav1 -preset 6 -crf 28 -c:a libopus -b:a 128k output.mkv
```

Use PSY features via `-svtav1-params` (colon-separated):

```bat
ffmpeg.exe -i input.mp4 -c:v libsvtav1 -preset 4 -crf 26 ^
  -svtav1-params "tune=3:enable-variance-boost=1:variance-boost-strength=2" ^
  -c:a libopus -b:a 160k output.mkv
```

List all encoder options:

```bat
ffmpeg.exe -h encoder=libsvtav1
```

## Common PSY / SVT params
- `tune=0|1|2|3` — 0=VQ, 1=PSNR, 2=SSIM, 3=subjective/PSY
- `enable-variance-boost=1`
- `variance-boost-strength=1..4`
- `enable-qm=1` / `qm-min` / `qm-max`
- `sharpness=-7..7`
- `film-grain=0..50`

## preset / crf rules of thumb
- **preset** 0–13: lower = slower + better. Start at `6`; `2–4` for archival.
- **crf**: lower = better + bigger. Start at `28`; ~`20` (great) … `40` (small).

## Build it yourself
Push to `main` or run the **workflow_dispatch** trigger in the Actions tab.
The runner cross-compiles everything from source and publishes the zip.

License: **GPLv3** (`--enable-gpl --enable-version3`).
