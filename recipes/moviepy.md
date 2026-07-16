# MoviePy: assemble the cut, then add music matched to it

MoviePy pipelines assemble clips programmatically — concat, crossfade, text overlays — and then
usually ship the result silent, because the video side got automated and the audio side never did.
This recipe adds one step after your existing assembly: export the cut, hand it to Sonilo, and mux
the returned soundtrack back. The music is composed from the finished cut and is **licensed and
safe for commercial use (terms apply)**.

## Prerequisites

- Python 3.9+ with `moviepy` (v2 imports shown; v1 notes at the bottom)
- FFmpeg on PATH (MoviePy uses it under the hood)
- Sonilo access: the [MCP server](https://github.com/sonilo-ai/sonilo-mcp) (`uvx sonilo-mcp`,
  needs a `SONILO_API_KEY`) or the REST API — docs at
  [sonilo.com](https://sonilo.com/?utm_source=github&utm_medium=oss&utm_campaign=v2m-cookbook)

## 1. Assemble and export the cut (silent)

Whatever your assembly already looks like — the only requirement is a single rendered file at the
end, exported without audio:

```python
from moviepy import VideoFileClip, concatenate_videoclips

paths = ["clip1.mp4", "clip2.mp4", "clip3.mp4"]
clips = [VideoFileClip(p).without_audio() for p in paths]

cut = concatenate_videoclips(clips, method="compose")
cut.write_videofile("cut.mp4", fps=30, codec="libx264", audio=False)
```

Export silent on purpose: the soundtrack is one layer added over the whole assembled cut, not
per clip.

## 2. Generate the soundtrack

```
video_to_music({
  video_path: "cut.mp4",
  prompt: "warm hopeful cinematic"   // optional single style hint — omit it and Sonilo matches what it sees
})
// → "soundtrack.m4a", matched to the cut's length
```

Or via the REST API (async job + webhook → `preview_url`, `final_audio_url`, `license_id`).

## 3. Mux the track back

Staying in MoviePy (re-encodes the video):

```python
from moviepy import VideoFileClip, AudioFileClip

video = VideoFileClip("cut.mp4")
music = AudioFileClip("soundtrack.m4a")
video.with_audio(music).write_videofile("final.mp4", codec="libx264", audio_codec="aac")
```

Or skip the re-encode with FFmpeg — video stream copied untouched, done in seconds:

```bash
ffmpeg -y -i cut.mp4 -i soundtrack.m4a \
  -map 0:v -map 1:a -c:v copy -c:a aac -b:a 320k -shortest final.mp4
```

## Notes

- **Length matching is automatic.** The returned track matches the cut you uploaded — nothing to
  trim or loop.
- **Narration in the cut?** Keep the dialogue on its own layer and pull the music bed down so
  speech stays clear:

  ```python
  from moviepy import CompositeAudioClip

  bed = AudioFileClip("soundtrack.m4a").with_volume_scaled(0.25)
  video.with_audio(CompositeAudioClip([dialogue, bed])).write_videofile("final.mp4")
  ```

- **Big master?** Feed Sonilo a 720p proxy instead — it only needs motion and timing, and the
  track drops onto the full-resolution master identically (see
  [`RECIPES.md`](../RECIPES.md#add-a-matched-soundtrack-with-sonilo)).
- **Caps:** music takes videos up to 6 minutes; sound effects up to 3.
- The same upload can also produce a **royalty-free sound-effects track** (`video_to_sfx`) — see
  [`RECIPES.md`](../RECIPES.md#add-sound-effects-with-sonilo).
- **MoviePy v1:** import from `moviepy.editor`, and use `set_audio` / `volumex` in place of
  `with_audio` / `with_volume_scaled`.
