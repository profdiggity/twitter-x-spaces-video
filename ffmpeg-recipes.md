## FFmpeg Recipes

### 1) Concatenate MP4 files without re-encoding

```bash
# Creates a temporary concat list, merges files by stream copying, then deletes the list.
echo -e "file 'input_01.mp4'\nfile 'input_02.mp4'" > concat_list.txt \
  && ffmpeg -f concat -safe 0 -i concat_list.txt -c copy merged.mp4 \
  && rm concat_list.txt
```

#### What it does

* Writes a concat manifest file (`concat_list.txt`) containing the input file paths.
* Uses FFmpeg’s **concat demuxer** to join the files in order.
* Uses `-c copy` to **stream copy** (no re-encode), which is fast and preserves quality.

#### Key flags

* `-f concat`: Use the concat demuxer (expects a text file list).
* `-safe 0`: Allow “unsafe” paths in the list (absolute paths, special characters). If your paths are simple and relative, you can often omit this.
* `-i concat_list.txt`: The list file.
* `-c copy`: Copy audio and video bitstreams as-is.

#### Notes and gotchas

* For stream copy concat to work reliably, the inputs should match in important ways: container expectations, codec, resolution, frame rate, audio parameters, etc. If they differ, FFmpeg may error or the output may desync.
* If you need to concatenate *many* files, put each on its own line: `file 'input_03.mp4'`, etc.

---

### 2) Loop background music and combine with an existing video

```bash
ffmpeg -y \
  -stream_loop -1 -i bgm.mp3 \
  -i input.mp4 \
  -map 1:v:0 -map 0:a:0 \
  -filter:a "volume=0.2" \
  -c:v copy -c:a aac \
  -shortest -movflags +faststart \
  video_with_bgm.mp4
```

#### What it does

* Loops the MP3 forever, then pairs it with the video.
* Takes **video** from the MP4 and **audio** from the MP3.
* Turns the audio down to 20% volume.
* Encodes audio to AAC and copies the video stream without re-encoding.
* Stops when the shortest input ends (so the output ends when the video ends).

#### Key flags

* `-y`: Overwrite output if it exists.
* `-stream_loop -1 -i bgm.mp3`: Loop the audio input indefinitely.
* `-i input.mp4`: Video input.
* `-map 1:v:0`: Use the first video stream from the second input (the MP4).
* `-map 0:a:0`: Use the first audio stream from the first input (the MP3).
* `-filter:a "volume=0.2"`: Reduce audio volume.
* `-c:v copy`: Copy video without re-encoding.
* `-c:a aac`: Encode audio as AAC (common for MP4).
* `-shortest`: End the output when the shortest stream ends (usually the video).
* `-movflags +faststart`: Moves MP4 metadata to the beginning for better web playback.

#### Notes and gotchas

* If you want to **mix** the MP3 with the video’s original audio (instead of replacing it), you would use `amix` in a `-filter_complex` graph. This command replaces the audio entirely.
* If your MP4 has no video stream at `1:v:0`, adjust the `-map` selectors accordingly.

---

### 3) Mute specific time ranges (Bash script)

This script mutes the audio only during time ranges you provide, while leaving video unchanged.

#### Example usage

```bash
./mute_sections.sh input.mp4 muted_sections.mp4 "5-8,12-13,30-35"
```

That means:

* Mute from 5s to 8s
* Mute from 12s to 13s
* Mute from 30s to 35s

#### Script

```bash
#!/usr/bin/env bash
set -euo pipefail

# mute_sections.sh
#
# Purpose
# - Mute the audio during one or more time ranges (in seconds) while keeping the video unchanged.
#
# Usage
#   ./mute_sections.sh input.mp4 muted_sections.mp4 "5-8,12-13,30-35"
#
# Arguments
#   $1  Input video path
#   $2  Output video path
#   $3  Comma-separated mute ranges in seconds, formatted as "START-END"
#
# Notes
# - Time ranges are inclusive for the filter's between(t,start,end) function.
# - Video is stream-copied. Audio is encoded to AAC for MP4 compatibility.
# - If the input has no audio stream, this will fail.

INPUT="$1"
OUTPUT="$2"
MUTES="$3"

# Build an enable expression like:
#   between(t,5,8)+between(t,12,13)+between(t,30,35)
ENABLE_EXPR=""
IFS=',' read -ra RANGES <<< "$MUTES"

for RANGE in "${RANGES[@]}"; do
  START=${RANGE%-*}
  END=${RANGE#*-}

  if [ -n "$ENABLE_EXPR" ]; then
    ENABLE_EXPR="${ENABLE_EXPR}+between(t,${START},${END})"
  else
    ENABLE_EXPR="between(t,${START},${END})"
  fi
done

# Apply volume=0 only when enable=... evaluates true.
# - The filter runs on the input audio stream [0:a].
# - Output audio is labeled [a] and mapped alongside the copied video stream.
ffmpeg -i "$INPUT" -filter_complex \
  "[0:a]volume=enable='${ENABLE_EXPR}':volume=0[a]" \
  -map 0:v -map "[a]" \
  -c:v copy -c:a aac \
  "$OUTPUT"
```

#### How the audio muting works

* `between(t,START,END)` returns true when the current timestamp `t` is inside that range.
* The script ORs multiple ranges by adding them together:

  * `between(...) + between(...) + ...`
  * Any non-zero result enables the muting.
* The filter itself is:

  * `volume=enable='EXPR':volume=0`
  * When enabled, volume becomes `0` (mute). Otherwise volume stays unchanged.

#### Common tweaks

* Fade mute edges (less abrupt):

  * You can add fades using `afade` and/or replace the volume approach with a more complex filter graph.
* Keep original audio codec:

  * MP4 usually wants AAC, which is why `-c:a aac` is used. If your container supports your original codec, you can change it.

---

## 4) Fit a non-16:9 video into 1920x1080 using a blurred background

```bash
ffmpeg -i input.mp4 -filter_complex \
"[0:v]scale=1920:1080,boxblur=20:1[bg]; \
 [0:v]scale=-1:1080[fg]; \
 [bg][fg]overlay=(W-w)/2:(H-h)/2" \
-c:a copy output_blur.mp4
```

### What it does

Creates a 1080p, 16:9 output even if the source video is vertical or otherwise not 16:9:

* Makes a full frame background by scaling the original to 1920x1080, then blurring it
* Makes a foreground version scaled to height 1080 while preserving aspect ratio
* Overlays the foreground centered on top of the blurred background
* Copies the original audio without re-encoding

### Filter graph breakdown

* `[0:v]scale=1920:1080,boxblur=20:1[bg]`

  * Take the input video stream (`0:v`)
  * Scale to 1920x1080
  * Blur it with `boxblur=20:1`
  * Label the result as `[bg]` for background

* `[0:v]scale=-1:1080[fg]`

  * Take the same input again
  * Scale to height 1080, width auto computed (`-1`) to preserve aspect ratio
  * Label as `[fg]` for foreground

* `[bg][fg]overlay=(W-w)/2:(H-h)/2`

  * Overlay foreground centered on the background
  * `W,H` are background dimensions, `w,h` are foreground dimensions

### Key flags

* `-filter_complex`: Needed because multiple video streams and labels are used
* `-c:a copy`: Copy audio stream as-is into the output container

### Notes and tweaks

* Change blur strength by adjusting `boxblur=20:1`:

  * Larger first number means more blur
* If you want 720p output instead:

  * Replace `1920:1080` with `1280:720`
  * Replace `-1:1080` with `-1:720`

---

## 5) Create a video from a static image plus an audio file, with a waveform overlay

```bash
ffmpeg \
-loop 1 -i cover.png \
-i audio.m4a \
-filter_complex "\
[1:a]aformat=channel_layouts=mono,showwaves=s=1280x200:mode=line:rate=25:colors=white[waveform]; \
[0:v]scale=1280:720[bg]; \
[bg][waveform]overlay=(W-w)/2:(H-h)/2,format=yuv420p[v]" \
-map "[v]" -map 1:a \
-shortest \
-r 30 \
waveform_video.mp4
```

### What it does

* Loops a still image so it lasts for the full audio duration
* Generates a waveform visualization from the audio
* Overlays the waveform on top of the image
* Outputs a standard, widely compatible MP4 video with H.264 style pixel format (`yuv420p`)
* Ends when audio ends

### Filter graph breakdown

* `[1:a]aformat=channel_layouts=mono,...[waveform]`

  * Use audio stream from second input (`1:a`)
  * Convert to mono (cleaner, more predictable waveform)
  * `showwaves` generates a video stream of the waveform:

    * `s=1280x200`: waveform video size
    * `mode=line`: line style waveform
    * `rate=25`: waveform frames per second
    * `colors=white`: waveform color
  * Label waveform as `[waveform]`

* `[0:v]scale=1280:720[bg]`

  * Scale the cover image to 1280x720
  * Label as `[bg]`

* `[bg][waveform]overlay=(W-w)/2:(H-h)/2,format=yuv420p[v]`

  * Center the waveform on the background
  * Convert the pixel format to `yuv420p` for broad compatibility
  * Label final video as `[v]`

### Key flags

* `-loop 1 -i cover.png`: Loop the image indefinitely as a video source
* `-map "[v]"`: Use the filtered video output labeled `[v]`
* `-map 1:a`: Use the original audio stream
* `-shortest`: Stop when the shortest stream ends (usually the audio determines duration)
* `-r 30`: Set output frame rate to 30 fps

### Notes and tweaks

* Move the waveform to the bottom:

  * Replace `overlay=(W-w)/2:(H-h)/2` with:

    * `overlay=(W-w)/2:H-h-40` (40 px margin)
* Make the waveform taller:

  * Increase `showwaves` height, for example `s=1280x300`
* If you need guaranteed H.264 output encoding settings:

  * Add `-c:v libx264 -crf 18 -preset medium`
  * Keep `format=yuv420p` for compatibility

---

## Turn Up Volume by Decibels and Clip Video at Specific Points

### A) Increase volume by N dB for the whole output

Use `volume=XdB` to apply a decibel gain.

```bash
# Example: boost audio by +6 dB and re-encode audio to AAC
ffmpeg -i input.mp4 \
  -filter:a "volume=6dB" \
  -c:v copy -c:a aac \
  output_louder.mp4
```

Notes:

* `+6 dB` is about double the amplitude, which can clip if the peaks are already near 0 dBFS.
* If you hear distortion, add a limiter (example below).

Optional limiter to reduce clipping risk:

```bash
ffmpeg -i input.mp4 \
  -filter:a "volume=6dB,alimiter=limit=0.98" \
  -c:v copy -c:a aac \
  output_louder_limited.mp4
```

### B) Clip the video at certain points

There are two common patterns:

#### Option 1: Single clip (start to end)

```bash
# Keep only 00:00:12 to 00:00:34
ffmpeg -ss 00:00:12 -to 00:00:34 -i input.mp4 \
  -c copy \
  clip_001.mp4
```

Notes:

* This is fast and usually good for rough cuts.
* With `-c copy`, cut points may land on keyframes, so the exact start frame may shift slightly.

If you need frame accurate cuts, re-encode video:

```bash
ffmpeg -ss 00:00:12 -to 00:00:34 -i input.mp4 \
  -c:v libx264 -crf 18 -preset medium \
  -c:a aac \
  clip_001_exact.mp4
```

#### Option 2: Multiple clips stitched together

Example: keep three sections, boost volume by +4 dB, then concatenate.

1. Create the clips:

```bash
ffmpeg -ss 00:00:05 -to 00:00:12 -i input.mp4 -c copy clip_a.mp4
ffmpeg -ss 00:00:20 -to 00:00:28 -i input.mp4 -c copy clip_b.mp4
ffmpeg -ss 00:00:40 -to 00:00:55 -i input.mp4 -c copy clip_c.mp4
```

2. Concatenate using the concat demuxer:

```bash
printf "file 'clip_a.mp4'\nfile 'clip_b.mp4'\nfile 'clip_c.mp4'\n" > concat_list.txt
ffmpeg -f concat -safe 0 -i concat_list.txt -c copy stitched.mp4
rm concat_list.txt
```

3. Apply a volume gain in dB (audio re-encode required if you filter audio):

```bash
ffmpeg -i stitched.mp4 \
  -filter:a "volume=4dB" \
  -c:v copy -c:a aac \
  output_clipped_and_louder.mp4
```

---

## Turn Up Volume Only Between Two Timestamps

This applies gain only within a time window, leaving the rest unchanged.

### A) Single time window with `enable`

```bash
# Boost +8 dB only from 00:00:12 to 00:00:18
ffmpeg -i input.mp4 \
  -filter:a "volume=enable='between(t,12,18)':volume=8dB" \
  -c:v copy -c:a aac \
  output_boosted_window.mp4
```

Notes:

* The timestamps in `between(t,START,END)` are in seconds.
* You can use decimals like `12.5`.

### B) Fade in and fade out the boosted region (optional)

Abrupt gain changes can sound harsh. This adds short fades around the boosted region.

```bash
# Boost window plus gentle fades in/out around the region
ffmpeg -i input.mp4 \
  -filter_complex "\
[0:a]asplit[a0][a1]; \
[a1]volume=8dB,afade=t=in:st=12:d=0.15,afade=t=out:st=17.85:d=0.15[a1f]; \
[a0][a1f]amix=inputs=2:normalize=0[a]" \
  -map 0:v -map "[a]" \
  -c:v copy -c:a aac \
  output_boosted_window_smooth.mp4
```

This pattern duplicates audio, boosts one copy with fades, then mixes it back with the original.
