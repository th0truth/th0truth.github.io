# ffmpeg-optimizer

Lightweight video optimizer written in Python that uses FFmpeg to compress video files into streaming-ready MP4s while keeping good visual quality.

## Key Features
- **Quality-Driven Encoding**: Uses CRF instead of a fixed bitrate, with selectable speed presets (`ultrafast` … `veryslow`).
- **Streaming Defaults**: H.264 (`libx264`, High profile, `yuv420p`), AAC 96 kbps stereo audio, and `+faststart` for progressive playback.
- **Size Reporting**: Prints file size before and after, plus the percentage saved.
- **Zero Dependencies**: Pure Python standard library; only FFmpeg is required in `PATH`.

## Tech Stack
- **Python**
- **FFmpeg**
- **libx264 / AAC**
