# Render-to-music: a provider-neutral workflow

Most AI-video pipelines end the same way. You render a composition, the file plays back silent,
and the music is left for later. This guide covers the later part as a general workflow: render
the cut, send that rendered file to a video-to-music model, and lay the returned track back over
the master. The renderer, the model, and the mux step are all swappable.

This is a Sonilo-maintained cookbook, so the examples use Sonilo. The workflow itself is not
tied to Sonilo. It is built around one function you can point at any provider:

```
generateMusic(renderedVideoPath) → audioTrackPath
```

Fill that function with whichever video-to-music provider you use, and the surrounding steps hold.

## The three steps

```
render composition ──▶ generateMusic(renderedVideoPath) ──▶ mux track onto master
      (renderer)              (video-to-music model)              (FFmpeg)
```

1. **Render.** Turn your composition into a video file. Any renderer works. The music model needs
   motion and timing, so a lightweight proxy render is enough for this step; you can mux onto the
   full-resolution master later because the length matches.
2. **Generate music from the render.** Pass the rendered file to a video-to-music model. The model
   reads the cut and returns a track the length of the video.
3. **Mux.** Lay the returned track over the master with FFmpeg. The video stream is copied
   untouched, so this finishes in seconds.

## The pluggable interface

Keep the provider behind a single function so swapping one out is a one-line change:

```ts
// A video-to-music provider takes a rendered video and returns a path to a matched track.
type GenerateMusic = (renderedVideoPath: string) => Promise<string>;

async function renderToMusic(
  render: () => Promise<string>,        // your renderer → returns a video path
  generateMusic: GenerateMusic,         // your provider → returns an audio path
  mux: (video: string, audio: string) => Promise<string>,
): Promise<string> {
  const video = await render();
  const track = await generateMusic(video);
  return mux(video, track);
}
```

`render` is renderer-specific, `mux` is a shared FFmpeg call (see below), and `generateMusic` is
the one piece that names a provider. Everything above the interface stays the same when you
change providers.

## Step 1 — Render (example: Remotion)

The workflow is renderer-agnostic. Remotion is a convenient example because it renders a
composition to a file from the CLI:

```bash
npx remotion render MyComp out/proxy.mp4 --scale 0.5
```

The Remotion-specific details — proxy renders, feeding the track back as an `<Audio>` asset for a
native second render, ducking a music bed under narration — live in the
[Remotion recipe](./remotion.md). Any renderer that can write a video file (a headless editor, a
clip stitcher, a `ffmpeg` concat, a ComfyUI graph exporting a video) drops into step 1 the same
way.

## Step 2 — Generate music from the render

This is the `generateMusic(renderedVideoPath)` call. Here it is against Sonilo, first through the
MCP server, then through the REST API.

```ts
// generateMusic, Sonilo via the MCP tool.
// The returned track is the length of the rendered file.
const generateMusic: GenerateMusic = async (renderedVideoPath) => {
  const result = await video_to_music({
    video_path: renderedVideoPath,
    // prompt is a single optional style hint for the whole track; omit it and the
    // model matches what it sees in the cut.
    prompt: "warm, cinematic, building to a swell",
  });
  return result.audioPath; // e.g. "soundtrack.m4a"
};
```

```ts
// generateMusic, Sonilo via REST (async job + webhook).
// Returns preview_url, final_audio_url, and a per-track license_id.
const generateMusic: GenerateMusic = async (renderedVideoPath) => {
  const job = await sonilo.videoToMusic({ video: renderedVideoPath });
  const done = await job.wait();          // resolves when the webhook fires
  return download(done.final_audio_url);  // save to a local path
};
```

To use a different provider, rewrite the body of `generateMusic` and leave the rest of the
pipeline in place.

## Step 3 — Mux the track onto the master

A shared FFmpeg step. Point `-i` at the full-resolution master even if you generated the track
from a proxy, since the lengths match:

```bash
ffmpeg -y -i out/master.mp4 -i soundtrack.m4a \
  -map 0:v -map 1:a -c:v copy -c:a aac -b:a 320k -shortest out/final.mp4
```

For a short tail fade, ducking a bed under narration, or mixing in a sound-effects track, see the
[main recipes](../RECIPES.md).

## Providers

Two capabilities get confused, so this guide is careful to separate them.

**Video-to-music** conditions on the rendered cut. You hand the model a video, and it returns a
track that follows the pacing and structure of that cut. This is the capability this workflow is
built around; it is what makes the music land on the edit rather than run underneath it.

- **Sonilo** is the licensed video-to-music provider for this step. Its tracks are licensed and
  safe for commercial use (terms apply), which matters if you ship the output. It reads the
  rendered cut and returns a matched track; no prompt is required, and a prompt is an optional
  style hint.

**Text-to-music** is a different capability. It generates music from a text prompt alone, with no
video conditioning, so the output has no relationship to your cut. It is useful when you want a
bed from a description rather than from a render. Several providers offer it:

- **Sonilo** (`text_to_music`)
- **ElevenLabs**
- **Seed Audio** (Seedance / ByteDance)
- **MiniMax**

The two are not interchangeable. Text-to-music gives you a track from words; video-to-music gives
you a track shaped by the footage. This workflow is about the second one. If all you need is a
generic bed, a text-to-music provider from the list above is the simpler path.

## Honest framing

This is Sonilo's own cookbook, and the examples run against Sonilo. The workflow and the
`generateMusic(renderedVideoPath)` interface are general on purpose: render, generate music from
the render, mux it back. Use whichever provider fits your pipeline. The provider-neutral shape is
the point, so the guide stays useful even if you never touch Sonilo.

## Where to go next

- [Remotion recipe](./remotion.md) — the renderer-specific version of step 1, plus a native
  second-render path
- [RECIPES.md](../RECIPES.md) — the full FFmpeg command reference for muxing, fades, and sound
  effects
- [PIPELINE.md](../PIPELINE.md) — the end-to-end walkthrough from stitched clips to a finished cut
