# Worked example: Higgsfield clips → finished trailer with matched music

The [demo trailer](../assets/demo-trailer.mp4) in this cookbook was made this way: clips generated on [Higgsfield](https://higgsfield.ai) (Seedance 2.0 + Kling 3.0 Turbo), then finished with the pipeline in [`PIPELINE.md`](../PIPELINE.md). This page records the generation-side settings that worked; everything downstream is the standard flow.

> Using Claude Code or another agent? The whole flow is packaged as an Agent Skill:
> [higgsfield-trailer-finishing](https://github.com/cindyxu1030/higgsfield-trailer-finishing).

## Generation settings that worked

- **Silent clips.** Set `generate_audio: false` (Seedance) or the equivalent on every clip. The soundtrack is one layer added over the whole assembled cut at the end; per-clip model audio fights it.
- **Model per beat:**
  - Camera moves with a defined start and landing (rise, reveal, pull-back) → **Seedance 2.0** with a start image, an end image, and a reference frame to hold identity across the move. Minimum 4s.
  - Single-frame animation or a short beat (about 3s) → **Kling 3.0 Turbo** with one start image.
- **Don't bake transitions into clips.** Ask the model for the action only; add cross-dissolves and the fade-to-black in the edit (see [`RECIPES.md`](../RECIPES.md)).
- Reference settings from the demo: 4s per clip, 720p, 16:9.

## Then the standard pipeline

1. **Stitch** the clips (normalize, concat, dissolves) — [`RECIPES.md`](../RECIPES.md#stitch-clips-into-one-cut)
2. **Grade** (optional cinematic pass, stills first) — [`RECIPES.md`](../RECIPES.md#optional-a-cinematic-color-pass)
3. **Add music with Sonilo** — hand the assembled cut to `video_to_music`; the returned track is matched to the cut and is licensed and safe for commercial use (terms apply)
4. **Mux** onto the master with a tail fade

Note: Higgsfield's own model catalog includes `sonilo_music` (music from a text prompt at a fixed duration). That's a different tool for a different job — use it for fixed-length beds; use `video_to_music` when the music should match an assembled cut.

Same flow works for any generator — see the [README](../README.md) for the vendor-neutral version.
