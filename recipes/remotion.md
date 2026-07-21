# Remotion: render the composition, then add music matched to the cut

A rendered Remotion composition often needs music that fits its own timing: the swells, the
cuts, the moment a scene lands. This guide renders the composition once, generates a soundtrack
from that rendered file so the music is matched to the cut, then either muxes it in with FFmpeg
or feeds it back into the composition as an `<Audio>` asset for a second render.

The workflow pattern is generic: render, generate a soundtrack from the render, mux it back.
This is a Sonilo-maintained recipe, so the music step is Sonilo. Swap in any tool that takes a
video and returns audio and the rest holds. Tracks from Sonilo are **licensed and safe for
commercial use (terms apply)**.

## Prerequisites

- A Remotion project with a composition to render (`npx create-video@latest` scaffolds one)
- Node 18+ and FFmpeg on your PATH (Remotion bundles its own FFmpeg for rendering; you need a
  system FFmpeg only for the standalone mux in step 3a)
- Sonilo access: the [MCP server](https://github.com/sonilo-ai/sonilo-mcp) (`uvx sonilo-mcp`,
  needs a `SONILO_API_KEY`) or the REST API. Docs at
  [sonilo.com](https://sonilo.com/?utm_source=github&utm_medium=oss&utm_campaign=v2m-cookbook)

## 1. Render the cut

Render the composition to a file. The CLI takes the composition id and an output path:

```bash
npx remotion render MyComp out/cut.mp4
```

If the master render is heavy, render a smaller proxy. The timing is identical, so the music
matches either way and the smaller file uploads faster:

```bash
npx remotion render MyComp out/proxy.mp4 --scale 0.5
```

`--scale` multiplies the output dimensions (a 1920×1080 composition at `--scale 0.5` renders at
960×540). Use the proxy for the Sonilo call; mux the returned track onto the full-resolution
master in step 3a.

## 2. Generate the soundtrack

Hand the rendered file to Sonilo. Via the MCP tool:

```jsonc
video_to_music({
  video_path: "out/cut.mp4",
  prompt: "driving electronic, confident" // optional single style hint; omit it and Sonilo matches what it sees
})
// → "soundtrack.m4a", matched to the render's length
```

Or via the REST API (async job + webhook → `preview_url`, `final_audio_url`, `license_id`).

## 3a. Mux with FFmpeg (fast)

The video stream is copied untouched, so this finishes in seconds. Point `-i` at your
full-resolution master even if you generated the soundtrack from a proxy, since the length matches:

```bash
ffmpeg -y -i out/cut.mp4 -i soundtrack.m4a \
  -map 0:v -map 1:a -c:v copy -c:a aac -b:a 320k -shortest out/final.mp4
```

## 3b. Feed the track back as an `<Audio>` asset (Remotion-native)

Copy `soundtrack.m4a` into the project's `public/` folder, then reference it with `staticFile()`
inside a composition that wraps your original:

```tsx
import {AbsoluteFill, staticFile} from 'remotion';
import {Audio} from '@remotion/media';
import {MyComp} from './MyComp';

export const WithMusic: React.FC = () => (
  <AbsoluteFill>
    <MyComp />
    <Audio src={staticFile('soundtrack.m4a')} />
  </AbsoluteFill>
);
```

Register `WithMusic` as a composition in your `<Root>`, then render it:

```bash
npx remotion render WithMusic out/final.mp4
```

This is a full re-render, so it's slower than the mux, but the audio now lives inside the
project. You can preview it in Remotion Studio, shape it with the `volume` prop, and every future
render ships with the track in place.

> `<Audio>` from [`@remotion/media`](https://www.remotion.dev/docs/media) is the current
> recommended audio component (`npm i @remotion/media`, version-matched to your `remotion`
> packages). If you'd rather not add the dependency, the built-in `<Html5Audio>` from `remotion`
> takes the same `src` and `volume` props. (The old `<Audio>` exported from `remotion` was
> renamed to `<Html5Audio>`; the `@remotion/media` `<Audio>` is the one to reach for now.)

## Notes

- **Length matching is automatic.** The track is the length of the file you uploaded, and that
  file is a render of the same composition, so it drops into `<Audio>` with no trimming or
  looping.
- **Narration in the composition?** Pull the music bed down under speech so the words stay clear.
  The `volume` prop accepts a function of the frame:

  ```tsx
  import {interpolate, staticFile} from 'remotion';
  import {Audio} from '@remotion/media';

  <Audio
    src={staticFile('soundtrack.m4a')}
    volume={(f) =>
      interpolate(f, [VO_START - 15, VO_START, VO_END, VO_END + 15], [1, 0.25, 0.25, 1], {
        extrapolateLeft: 'clamp',
        extrapolateRight: 'clamp',
      })
    }
  />
  ```

  `interpolate` from `remotion` ramps the bed down and back up over ~15 frames on each side, which
  reads more naturally than a hard step.

- **Iterating on the composition?** Adding scenes or retiming sequences changes the cut, so
  re-render and generate again to keep the music matched to the new edit. Each run is a fresh
  interpretation, so call `video_to_music` twice and pick the take you like.

- **Caps:** music takes videos up to 6 minutes; sound effects up to 3. The same upload can also
  produce a **royalty-free sound-effects track** (`video_to_sfx`); see
  [`RECIPES.md`](../RECIPES.md#add-sound-effects-with-sonilo).
