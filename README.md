# FFmpeg (Windows x64) — SVT-AV1-PSY + Opus

Standalone, **statically linked** FFmpeg for Windows 64-bit with the
**SVT-AV1-PSY** encoder ([5fish/SVT-AV1](https://github.com/5fish/SVT-AV1)),
**libopus**, **libdav1d** and **libvmaf** built in.
No DLLs, no install — just run `ffmpeg.exe`.

Built automatically by GitHub Actions (see `.github/workflows/build.yml`).
Grab `ffmpeg-svtav1-psy-win64.zip` from the
[Releases](../../releases/tag/latest) page.

Verify your download against the `SHA256SUMS.txt` published next to it:

```powershell
(Get-FileHash ffmpeg-svtav1-psy-win64.zip -Algorithm SHA256).Hash.ToLower()
```

## Contents
- `ffmpeg.exe` — encoder/transcoder (SVT-AV1-PSY is in here)
- `ffprobe.exe` — file/stream analysis
- `BUILDINFO.txt` — exact versions and commits this binary was built from

## Versions
- FFmpeg: `n9.0.1`
- SVT-AV1-PSY: [`5fish/SVT-AV1`](https://github.com/5fish/SVT-AV1) `v2.3.260719`
- libopus: `1.6.1`
- libvmaf: `3.2.0` (VMAF scoring, models built in)
- libdav1d: `1.5.4` (fast AV1 decoding)
- Toolchain: mingw-w64 GCC, static (SSE2…AVX512)

Every source is pinned to an immutable tag, so any release can be rebuilt
byte-for-byte from the recipe. `BUILDINFO.txt` inside the zip records what
went into that specific build.

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

## VMAF scoring

Compare an encoded file against the original (distorted first, reference second):

```bat
ffmpeg.exe -i encoded.mkv -i original.mkv -lavfi libvmaf -f null -
```

Faster with threads + a specific model, and write a log:

```bat
ffmpeg.exe -i encoded.mkv -i original.mkv ^
  -lavfi "libvmaf=n_threads=8:model=version=vmaf_v0.6.1:log_path=vmaf.json:log_fmt=json" -f null -
```

If resolutions differ, scale the distorted to match the reference first:

```bat
ffmpeg.exe -i encoded.mkv -i original.mkv ^
  -lavfi "[0:v]scale=1920:1080:flags=bicubic[d];[d][1:v]libvmaf=n_threads=8" -f null -
```

The final `VMAF score: NN.NN` line is printed at the end (0–100, higher = better).

AV1 inputs decode through **libdav1d**, so scoring an AV1 encode against an
AV1 reference is fast rather than painfully slow.

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
The runner cross-compiles everything from source, smoke-tests the resulting
`.exe` under wine (real encode → decode → VMAF run), and publishes a dated
release plus the rolling `latest` tag.

To bump a component, edit the pinned version in the `env:` block of
`.github/workflows/build.yml`. Dependency builds are cached and keyed on those
versions, so unrelated changes only rebuild FFmpeg.

License: **GPLv3** (`--enable-gpl --enable-version3`). Full license text and the
list of upstream sources ship inside the zip.
