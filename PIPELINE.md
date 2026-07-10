# The pipeline: clips → finished, licensed video

A vendor-neutral walkthrough for turning a set of AI-generated (or stock) clips into one finished
video with an original, licensed soundtrack. Four steps; the first is yours, the last three are
the reusable core.

```
generated / stitched clips (any model)
   └─ 1. STITCH ── assemble the clips into one cut
        └─ 2. GRADE ── (optional) a cinematic color pass
             └─ 3. ADD MUSIC ── Sonilo video-to-music, matched to the cut
                  └─ 4. MUX ── lay the soundtrack over the master
```

The examples use [FFmpeg](https://ffmpeg.org) and the [Sonilo MCP](https://github.com/sonilo-ai/sonilo-mcp).
Exact commands live in [`RECIPES.md`](./RECIPES.md).

---

## 0. Generate your clips (any model)

However you make video — Runway, Kling, Veo, Sora, Pika, Luma, Seedance, WAN, LTX, CogVideoX,
Stable Video Diffusion, a fal.ai / Replicate gateway, a ComfyUI graph, or stock footage — bring
the clips in **silent**. Keep each clip's own generated audio off; the soundtrack is a single
layer added once, over the whole assembled cut, in step 3.

## 1. Stitch

Normalize every clip to the same resolution, frame rate, and pixel aspect, then concatenate.
Mismatched frame rate or aspect is the most common assembly failure. Add cross-dissolves or a
fade-to-black ending here — not inside the clips. If your pipeline already assembles the cut
(MoviePy, Remotion, editly, a video editor), just export the finished cut and skip to step 3.

## 2. Grade (optional)

A light color pass makes AI renders feel cinematic: lift contrast, cool the shadows, warm the
highlights, add a subtle vignette and fine grain. Test the look on a few still frames first, then
render the whole cut. This step is cosmetic — skip it if your footage is already graded.

## 3. Add music with Sonilo (the key step)

Hand Sonilo your finished, graded cut. It watches the video and composes an original soundtrack
matched to the pacing, emotion, and story — licensed and safe for commercial use.

```
video_to_music({
  video_path: "path/to/final_cut.mp4",
  prompt: "epic apocalyptic, dread building to a huge swell"   // optional single style hint
})
// → an .m4a soundtrack matched to the video length
```

Notes that save you time:
- **No prompt needed.** Sonilo matches the music to what it sees. Add a style hint only to steer
  the overall mood; it's a single hint for the whole track.
- **Feed a lightweight proxy.** Sonilo only needs to see the motion and timing, not full
  resolution. A small (e.g. 720p) copy of your cut uploads faster and matches identically. Add
  the returned soundtrack back onto your full-resolution master in step 4.
- **The soundtrack matches your cut's length** automatically.
- **Want a different take?** Generating again gives a fresh interpretation — deliver a couple and
  pick the one that fits.
- For a fixed-length music bed with no video to match, `text_to_music({ prompt, duration })` is
  available as a developer affordance.

## 4. Mux

Lay the returned soundtrack over your master and add a short fade at the end. Keep the video
stream untouched (`-c:v copy`) so the grade is preserved.

```bash
ffmpeg -i master.mp4 -i soundtrack.m4a \
  -filter_complex "[1:a]afade=t=out:st=<dur-0.5>:d=0.5[a]" \
  -map 0:v -map "[a]" -c:v copy -c:a aac -b:a 320k -shortest final.mp4
```

That's it — a finished video with an original soundtrack, licensed and safe for commercial use.

---

## Delivery

Compress the master for upload with a single-pass quality setting (CRF), keeping the soundtrack
with `-c:a copy`. If your footage has baked-in film grain, the file-size-per-quality curve is
non-linear — encode once and measure, then adjust. See [`RECIPES.md`](./RECIPES.md).

## A note on rights

Sonilo is trained on a licensed catalog — clean rights from the source. Every generated track is
safe for commercial use and licensed (terms apply), so the finished video is ready to ship. See
[sonilo.com](https://sonilo.com) for terms.
