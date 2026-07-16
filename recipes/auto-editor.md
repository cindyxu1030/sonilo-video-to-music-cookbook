# auto-editor: tighten the cut, then add music matched to it

[auto-editor](https://github.com/WyattBlue/auto-editor) removes the dead air — silence in a
talking-head take, static stretches in a screen recording — and leaves you a tighter cut with the
dialogue still on it. That new cut has new timing, which is exactly when a pre-picked track stops
fitting. This recipe generates the music **after** the tighten, from the cut auto-editor actually
produced, then mixes it under the remaining dialogue. Every track is **licensed and safe for
commercial use (terms apply)**.

auto-editor itself runs fully local by design; the music step here is a separate downstream call
in your own pipeline — nothing changes inside auto-editor.

## Prerequisites

- `pip install auto-editor` (ships with its own FFmpeg)
- FFmpeg on PATH for the mix step
- Sonilo access: the [MCP server](https://github.com/sonilo-ai/sonilo-mcp) (`uvx sonilo-mcp`,
  needs a `SONILO_API_KEY`) or the REST API — docs at
  [sonilo.com](https://sonilo.com/?utm_source=github&utm_medium=oss&utm_campaign=v2m-cookbook)

## 1. Tighten the cut

```bash
# cut on silence (talking-head footage)
auto-editor input.mp4 --edit audio:threshold=4% --margin 0.2sec -o tightened.mp4

# or cut on motion (screen recordings)
auto-editor input.mp4 --edit motion:threshold=2% -o tightened.mp4
```

`--margin` keeps a little breathing room around each kept section so the cut doesn't feel
clipped.

## 2. Generate music from the tightened cut

Order matters: tighten first, then generate. The soundtrack is composed from the cut it sees, so
it should see the final timing, not the rambling original.

```
video_to_music({
  video_path: "tightened.mp4",
  prompt: "understated, steady, stays out of the way"   // optional single style hint
})
// → "soundtrack.m4a", matched to the tightened cut's length
```

Or via the REST API (async job + webhook → `preview_url`, `final_audio_url`, `license_id`).

## 3. Mix the bed under the dialogue

The tightened cut still carries the dialogue, so this is a mix, not a replacement — music pulled
down to bed level, video stream untouched:

```bash
ffmpeg -y -i tightened.mp4 -i soundtrack.m4a \
  -filter_complex "[1:a]volume=0.25[bed];[0:a][bed]amix=inputs=2:duration=first:normalize=0[a]" \
  -map 0:v -map "[a]" -c:v copy -c:a aac -b:a 320k final.mp4
```

Want the bed to dip automatically whenever someone speaks? Duck it against the dialogue instead
of using a fixed volume:

```bash
ffmpeg -y -i tightened.mp4 -i soundtrack.m4a -filter_complex \
 "[0:a]asplit=2[vo][sc];[1:a][sc]sidechaincompress=threshold=0.05:ratio=8:attack=20:release=400[bed];[vo][bed]amix=inputs=2:duration=first:normalize=0[a]" \
 -map 0:v -map "[a]" -c:v copy -c:a aac -b:a 320k final.mp4
```

## Notes

- **Length matching is automatic.** The returned track matches the tightened cut — nothing to
  trim when the edit gets shorter.
- **Keep the narration clear.** A bed around `volume=0.2`–`0.3` supports the dialogue without
  competing with it; start low and raise it only if the mix feels empty.
- **Re-tightened the cut?** New timing means a new cut — run `video_to_music` on the new file so
  the music matches the edit you're actually shipping.
- **Caps:** music takes videos up to 6 minutes. Tightening often brings a long take under the
  cap; if the result is still longer, split it into sections and generate per section.
- The same upload can also produce a **royalty-free sound-effects track** (`video_to_sfx`,
  3-minute video cap) — see [`RECIPES.md`](../RECIPES.md#add-sound-effects-with-sonilo).
