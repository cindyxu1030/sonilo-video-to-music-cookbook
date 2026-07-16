# Remotion: render the composition, then add music matched to it

Remotion turns React components into video, and most Remotion projects handle music the manual
way: pick a track, drop it in `public/`, hope the timing fits. This recipe generates the
soundtrack from the rendered cut instead — render once, hand the MP4 to Sonilo, then either mux
with FFmpeg or feed the track back into the composition as an `<Audio/>` asset. Every track is
**licensed and safe for commercial use (terms apply)**.

## Prerequisites

- A Remotion project (`npx create-video@latest`) with a composition to render
- FFmpeg on PATH (for the mux path)
- Sonilo access: the [MCP server](https://github.com/sonilo-ai/sonilo-mcp) (`uvx sonilo-mcp`,
  needs a `SONILO_API_KEY`) or the REST API — docs at
  [sonilo.com](https://sonilo.com/?utm_source=github&utm_medium=oss&utm_campaign=v2m-cookbook)

## 1. Render the cut

```bash
npx remotion render MyComp out/cut.mp4
```

If the master render is heavy, a half-scale proxy uploads faster and the timing matches
identically:

```bash
npx remotion render MyComp out/proxy.mp4 --scale 0.5
```

## 2. Generate the soundtrack

```
video_to_music({
  video_path: "out/cut.mp4",
  prompt: "driving electronic, confident"   // optional single style hint — omit it and Sonilo matches what it sees
})
// → "soundtrack.m4a", matched to the render's length
```

Or via the REST API (async job + webhook → `preview_url`, `final_audio_url`, `license_id`).

## 3a. Fast path: mux with FFmpeg

Video stream copied untouched, done in seconds:

```bash
ffmpeg -y -i out/cut.mp4 -i soundtrack.m4a \
  -map 0:v -map 1:a -c:v copy -c:a aac -b:a 320k -shortest out/final.mp4
```

## 3b. Remotion-native path: feed the track back as an asset

Copy `soundtrack.m4a` into `public/` and wrap the composition:

```tsx
import {AbsoluteFill, Audio, staticFile} from 'remotion';
import {MyComp} from './MyComp';

export const WithMusic: React.FC = () => (
  <AbsoluteFill>
    <MyComp />
    <Audio src={staticFile('soundtrack.m4a')} />
  </AbsoluteFill>
);
```

Then render the wrapped composition:

```bash
npx remotion render WithMusic out/final.mp4
```

Slower than the mux (it's a full re-render), but the audio lives inside the project: preview it
in Remotion Studio, shape it with the `volume` prop, and every future render ships with the
track in place.

## Notes

- **Length matching is automatic.** The track matches the MP4 you uploaded — and since that's a
  render of the same composition, it drops into `<Audio/>` with no trimming.
- **Voiceover in the composition?** Duck the bed under speech with a frame-driven volume so the
  narration stays clear:

  ```tsx
  <Audio
    src={staticFile('soundtrack.m4a')}
    volume={(f) => (f > VO_START && f < VO_END ? 0.25 : 1)}
  />
  ```

  Use `interpolate` for smooth ramps instead of hard steps.
- **Iterating on the composition?** Added scenes and retimed sequences change the cut — re-render
  and generate again so the music matches the new edit. Each run is a fresh interpretation.
- **Caps:** music takes videos up to 6 minutes; sound effects up to 3. The same upload can also
  produce a **royalty-free sound-effects track** (`video_to_sfx`) — see
  [`RECIPES.md`](../RECIPES.md#add-sound-effects-with-sonilo).
