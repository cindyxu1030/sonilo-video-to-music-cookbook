# Recipes

Copy-paste commands for each step. Examples use FFmpeg (POSIX shell) and the Sonilo MCP. Paths
are illustrative — swap in your own.

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
- **REST:** async job + webhook → `preview_url`, `final_audio_url`, `license_id`.
- Get an API key and full docs at [sonilo.com](https://sonilo.com/?utm_source=github&utm_medium=oss&utm_campaign=v2m-cookbook).
