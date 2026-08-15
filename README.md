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

The PSY features of this fork are driven by two parameters, passed through
`-svtav1-params` (colon-separated, names are the encoder's long options
without the leading `--`):

```bat
ffmpeg.exe -i input.mkv ^
  -c:v libsvtav1 -preset 2 -crf 20 -pix_fmt yuv420p10le ^
  -svtav1-params "lineart-psy-bias=5:texture-psy-bias=4" ^
  -color_primaries bt709 -color_trc bt709 -colorspace bt709 ^
  -c:a libopus -b:a 160k output.mkv
```

Smaller "mini" encode, still with PSY detail retention:

```bat
ffmpeg.exe -i input.mkv ^
  -c:v libsvtav1 -preset 4 -crf 32 -pix_fmt yuv420p10le ^
  -svtav1-params "lineart-psy-bias=3:texture-psy-bias=3" ^
  -c:a libopus -b:a 128k output.mkv
```

Encoding to 10-bit (`-pix_fmt yuv420p10le`) is worth it even for 8-bit
sources — it reduces banding and costs almost nothing.

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

## PSY params

> This is the **5fish** fork, not the older psy-ex SVT-AV1-PSY. Its PSY system
> is different, and the upstream authors explicitly warn against mixing it with
> parameters from other PSY variants — setting a `-psy-bias` already turns off
> `enable-variance-boost` and tunes a whole set of features for you. Start with
> the two below on their own.

- `lineart-psy-bias=0..7` — lineart / edge retention
- `texture-psy-bias=0..7` — texture and grain retention
- `noise-psy-bias=0..7` — noise retention (auto-set from `texture-psy-bias` ≥ 5)

Set **both** psy-bias values together; omitting one is not recommended.

| source | starting point |
| --- | --- |
| high quality / high fidelity | `lineart-psy-bias=5:texture-psy-bias=4` |
| weak lineart | `lineart-psy-bias=6` or `7` |
| heavy texture or noise | `texture-psy-bias=5..7` |
| flat, little texture | `texture-psy-bias=3` or lower |
| mini encodes | `lineart-psy-bias=3:texture-psy-bias=3` |

Below CRF 24 the encoder enables its high-quality bias automatically, below
CRF 16 the high-fidelity bias — you do not need to set those yourself.

Full reference: [Docs/Parameters.md](https://github.com/5fish/SVT-AV1/blob/main/Docs/Parameters.md)
in the encoder repo (more detailed than `--help`).

## preset / crf rules of thumb
- **preset** 0–13: lower = slower + better. This fork recommends `2` for the
  best quality, `4` or at most `6` when you want speed. `0` is *not*
  recommended — `2` uses smarter candidate filtering and often beats it.
- **crf**: lower = better + bigger. ≤`12` high fidelity, `16–20` high quality,
  `24–28` larger mini, `32–40` tiny mini. Fractional values work
  (`-svtav1-params "crf=26.5"`).

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
