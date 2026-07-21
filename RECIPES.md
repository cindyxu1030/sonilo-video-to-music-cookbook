# Recipes

Copy-paste commands for each step. Examples use FFmpeg (POSIX shell) and the Sonilo MCP. Paths
are illustrative — swap in your own.

**Working inside a specific editing framework?** Step-by-step versions of this flow:

- [MoviePy](./recipes/moviepy.md) — assemble in Python, export the cut, add the matched track back
- [Remotion](./recipes/remotion.md) — render the composition, then mux or feed the track back as an `<Audio>` asset
- [auto-editor](./recipes/auto-editor.md) — tighten the cut first, then generate music that matches the new timing

---

## Stitch clips into one cut

Normalize each clip identically, then concatenate (prevents frame-rate / aspect mismatches):

```bash
for f in clip1.mp4 clip2.mp4 clip3.mp4; do
  ffmpeg -y -i "$f" \
    -vf "scale=1920:1080:force_original_aspect_ratio=decrease,pad=1920:1080:(ow-iw)/2:(oh-ih)/2,setsar=1,fps=30" \
    -c:v libx264 -crf 18 -preset slow -pix_fmt yuv420p -an "norm_$f"
done
printf "file 'norm_clip1.mp4'\nfile 'norm_clip2.mp4'\nfile 'norm_clip3.mp4'\n" > list.txt
ffmpeg -y -f concat -safe 0 -i list.txt -c copy stitched.mp4
```

Cross-dissolve between two clips (0.5s) instead of a hard cut:

```bash
ffmpeg -y -i a.mp4 -i b.mp4 -filter_complex \
 "[0:v][1:v]xfade=transition=fade:duration=0.5:offset=<a_dur-0.5>,format=yuv420p" out.mp4
```

Fade to black ending: append `,fade=t=out:st=<dur-0.6>:d=0.6` to the video filter.

---

## Optional: a cinematic color pass

Test on stills first, then render the whole cut. A neutral cinematic look:

```bash
GRADE="eq=contrast=1.12:saturation=1.14:gamma=0.97,curves=r='0/0 0.5/0.53 1/1':b='0/0.06 0.5/0.47 1/0.88',vignette=PI/4.5,unsharp=5:5:0.5:5:5:0.0,noise=alls=5:allf=t"
ffmpeg -y -ss 2 -i stitched.mp4 -frames:v 1 test.png
ffmpeg -y -i test.png -vf "$GRADE" test_graded.png     # look at it, tune, repeat
ffmpeg -y -i stitched.mp4 -vf "$GRADE" -c:v libx264 -crf 16 -preset slow -pix_fmt yuv420p -tune film -an graded.mp4
```

The `curves` term warms highlights and cools shadows; `noise=…:allf=t` is temporal grain. Tune
`contrast` / `curves` to your footage.

---

## Add a matched soundtrack with Sonilo

**Make a lightweight proxy** (Sonilo only needs motion + timing):

```bash
ffmpeg -y -i graded.mp4 -vf "scale=1280:720" -c:v libx264 -crf 30 -preset veryfast -pix_fmt yuv420p -an proxy.mp4
```

**Call Sonilo** (MCP tool; the returned file is matched to the video length):

```
video_to_music({
  video_path: "path/to/proxy.mp4",
  prompt: "warm hopeful cinematic, building to an uplifting swell"   // optional single style hint
})
// → "soundtrack.m4a"
```

Or via the REST API (async job + webhook), which returns `preview_url`, `final_audio_url`, and a
per-track `license_id`. Full API docs at [sonilo.com](https://sonilo.com/?utm_source=github&utm_medium=oss&utm_campaign=v2m-cookbook).

**Mux the soundtrack onto the full-resolution master**, with a short tail fade, video untouched:

```bash
MUS="soundtrack.m4a"
DUR=$(ffprobe -v error -show_entries format=duration -of default=nk=1:nw=1 "$MUS")
FADE=$(awk "BEGIN{printf \"%.2f\", $DUR-0.5}")
ffmpeg -y -i graded.mp4 -i "$MUS" \
  -filter_complex "[1:a]afade=t=out:st=${FADE}:d=0.5[a]" \
  -map 0:v -map "[a]" -c:v copy -c:a aac -b:a 320k -shortest final.mp4
```

**Want an alternate take?** Call `video_to_music` again — each run is a fresh interpretation.
Deliver two and let the user pick.

---

## Add sound effects with Sonilo

`video_to_sfx` generates a royalty-free sound-effects track from the video itself: impacts, ambience, and transitions land where the picture needs them. Works alongside the music step or on its own. Examples: [DEMOS.md](./DEMOS.md#video--sound-effects).

Two things that differ from the music call:

- **The video cap is 3 minutes** (music allows 6). Pre-check with `ffprobe` before uploading.
- **The API is task-based, not streaming**: submit returns a task id, poll until it finishes, then download the audio. The MCP tools (`video_to_sfx`, `get_sfx_task` — sonilo-mcp v0.2.1+) handle this for you.

```
video_to_sfx({
  video_path: "path/to/final_cut.mp4",
  prompt: "footsteps in snow, wind, gear rustle"   // optional hint; omit to derive from the picture
})
// → an .m4a sound-effects track matched to the video length
```

`text_to_sfx(prompt, duration)` covers fixed-length one-shot effects with no video to match.

**Mix effects under the existing audio** (music and/or dialogue), ducked so they support rather than dominate:

```bash
ffmpeg -y -i final.mp4 -i sfx.m4a \
  -filter_complex "[1:a]volume=0.5[fx];[0:a][fx]amix=inputs=2:duration=first:normalize=0[a]" \
  -map 0:v -map "[a]" -c:v copy -c:a aac -b:a 320k final_with_sfx.mp4
```

If the video is silent (no music step yet), skip the `amix` and map the effects track directly.

---

## Deliver: compress under a size cap

Single-pass CRF, keep the soundtrack stream as-is:

```bash
ffmpeg -y -i final.mp4 -c:v libx264 -crf 24 -preset slow -pix_fmt yuv420p -c:a copy final_web.mp4
```

Lower CRF = higher quality + bigger file. If your footage has baked-in film grain the size curve
is non-linear, so encode once and measure, then adjust CRF up or down to hit your cap.

---

## Language reference

- **MCP:** `video_to_music(video_path? | video_url?, prompt?, output_directory?)` — `prompt` is a
  single optional style hint for the whole track. `text_to_music(prompt, duration)` generates a
  fixed-length bed with no video to match (a developer affordance).
- **MCP (SFX, v0.2.1+):** `video_to_sfx(video_path? | video_url?, prompt?, output_directory?)` +
  `get_sfx_task(task_id)`; `text_to_sfx(prompt, duration)` for fixed-length effects. 3-minute video cap.
- **REST:** music = async job + webhook → `preview_url`, `final_audio_url`, `license_id`;
  SFX = task pipeline (`POST /v1/video-to-sfx` → poll `/v1/tasks/{task_id}`).
- Get an API key and full docs at [sonilo.com](https://sonilo.com/?utm_source=github&utm_medium=oss&utm_campaign=v2m-cookbook).
