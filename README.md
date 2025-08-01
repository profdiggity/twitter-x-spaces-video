## 📼 FFmpeg Script Description

This script generates a video combining a static background image with an animated audio waveform overlay, using `ffmpeg`.

---

### 🎯 Goal

Create a video `Twitter_Spaces_Overlay_Video.mp4` that:

* Uses a static PNG image as the background
* Animates a waveform from the audio track
* Outputs a 1280×720 video at 30 FPS
* Ends when the audio ends

---

### 🧪 Full Command

```bash
ffmpeg \
-loop 1 -i Twitter_Spaces_Graphic_2025-07-31.png \
-i Arcanum_Stream.m4a \
-filter_complex "\
[1:a]aformat=channel_layouts=mono,showwaves=s=1280x200:mode=line:rate=25:colors=white[waveform]; \
[0:v]scale=1280:720[bg]; \
[bg][waveform]overlay=(W-w)/2:(H-h)/2,format=yuv420p[v]" \
-map "[v]" -map 1:a \
-shortest \
-r 30 \
Twitter_Spaces_Overlay_Video.mp4
```

---

### 🔍 Breakdown

#### 🖼️ Image Input

```bash
-loop 1 -i Twitter_Spaces_Graphic_2025-07-31.png
```

* Loops the static PNG image continuously as video background

#### 🔊 Audio Input

```bash
-i Arcanum_Stream.m4a
```

* Input audio file for waveform visualization and final video

#### 🎛️ Filter Complex

```bash
-filter_complex "
[1:a]aformat=channel_layouts=mono,
showwaves=s=1280x200:mode=line:rate=25:colors=white[waveform];
[0:v]scale=1280:720[bg];
[bg][waveform]overlay=(W-w)/2:(H-h)/2,
format=yuv420p[v]"
```

* `[1:a]aformat=channel_layouts=mono`: Ensure mono audio for waveform rendering
* `showwaves`: Generate a white line-style waveform (1280×200) from audio
* `scale`: Resize the background image to 1280×720
* `overlay`: Center the waveform on the image
* `format=yuv420p`: Ensures compatibility with most video players

#### 🎬 Output Configuration

```bash
-map "[v]" -map 1:a
```

* Use the composed video and original audio

```bash
-shortest
```

* End video when audio ends

```bash
-r 30
```

* Set output frame rate to 30 fps

```bash
Twitter_Spaces_Overlay_Video.mp4
```

* Output video filename

---

### ✅ Result

A polished, shareable video:

* Static branded background
* Synchronized waveform animation
* Clean audio
* Great for social media or archives (e.g., Twitter Spaces replay)

---

### 🛡️ License and Attribution

Created by **Sean O'Brien**
📧 [sean@ivycyber.com](mailto:sean@ivycyber.com)
🐦 [@profdiggity](https://twitter.com/profdiggity) for [@IvyCyberEd](https://twitter.com/IvyCyberEd)

#### MIT License

```
MIT License

Copyright (c) 2025 Sean O'Brien

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the “Software”), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
```

