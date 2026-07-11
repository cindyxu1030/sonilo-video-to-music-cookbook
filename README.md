# Sonilo Video-to-Music Cookbook

**Fully licensed. Safe for commercial use.**

You generated the clips. You stitched them together. Now they need music that fits the edit
and the story — and that's the hard part. Stock libraries and prompt-based generators don't
know where your cuts are, so the music never quite lands on your edit.

**[Sonilo](https://sonilo.com) turns your video into music.** Point it at your finished cut and
it composes an original soundtrack matched to the pacing, emotion, and story — in seconds, no
prompt needed. Every track is **licensed and safe for commercial use**, so creators and brands
can actually ship it.

This cookbook shows how to add that music step to **any** AI-video pipeline.

**▶ [Watch the reference demo](./assets/demo-trailer.mp4)** — a 15-second trailer assembled from
AI-generated clips (stitched → graded → music by Sonilo). The soundtrack was composed from the
finished cut: the swell lands on the reveal, the impact lands on the edit.

---

## Why this exists

Almost every open-source AI-video pipeline today generates clips, stitches them, and then either
ships **silent** or bolts on a **stock/prompt-picked track that ignores the edit**. The music is
the last mile — and it's the part that makes a cut feel finished.

Sonilo is built for exactly this: an original soundtrack **matched to your video**, licensed and
safe for commercial use. It's vendor-neutral about how the video was made — Runway, Kling, Veo,
Sora, Pika, Luma, Seedance, WAN, LTX, CogVideoX, Stable Video Diffusion, a fal.ai / Replicate
gateway, a ComfyUI graph, or plain stock footage. Sonilo sits **downstream** of all of them.

## Who this is for

Builders of AI-video tools — text-to-video orchestrators, clip stitchers, faceless-video
factories, agentic film systems, and video-editing frameworks — who want a music layer that
matches the finished cut instead of a random library track.

---

## Two ways to use Sonilo

| Surface | Best for | What you get |
|---|---|---|
| **MCP server** ([`sonilo-mcp`](https://github.com/sonilo-ai/sonilo-mcp)) | Agentic workflows (Claude, Codex, any MCP client) | A `video_to_music` tool — hand it a video, get back a matched audio file |
| **REST API** | Platforms & backends | Async job + webhook → `preview_url`, `final_audio_url`, and a per-track `license_id` (an auditable license record) |

> Sonilo also offers sound effects (SFX) and section-level control on the [web platform](https://sonilo.com).
> This cookbook focuses on the developer surfaces above.

## 60-second quickstart (MCP)

```jsonc
// Register the Sonilo MCP server (needs a SONILO_API_KEY from sonilo.com)
// claude mcp add sonilo --env SONILO_API_KEY=sk-... -- uvx sonilo-mcp
```

Then, in an agent session, hand Sonilo your finished cut:

```
video_to_music({
  video_path: "path/to/your/final_cut.mp4",
  prompt: "warm hopeful cinematic, building to an uplifting swell"   // optional style hint
})
// → an .m4a soundtrack, matched to your video's length
```

`prompt` is a single **optional** style hint for the whole track — Sonilo works with no prompt at
all, matching the music to what it sees. Then mux the returned audio onto your master (see
[`RECIPES.md`](./RECIPES.md)).

---

## The full pipeline (vendor-neutral)

```
generated / stitched clips (any model)
   └─ 1. STITCH ── assemble the clips into one cut
        └─ 2. GRADE ── (optional) a cinematic color pass
             └─ 3. ADD MUSIC ── Sonilo video-to-music, matched to the cut
                  └─ 4. MUX ── lay the soundtrack over the master
```

Full walkthrough with copy-paste commands: **[`PIPELINE.md`](./PIPELINE.md)** ·
Command reference: **[`RECIPES.md`](./RECIPES.md)** ·
Worked example: **[Higgsfield clips → finished trailer](./examples/higgsfield.md)**

## Integration patterns by tool type

- **Silent-output pipelines** (they stitch clips and ship no audio) → add a music step after the
  final assembly so the output arrives with an original soundtrack.
- **Stock / prompt-picked-music pipelines** → offer Sonilo as an alternative so the music is
  matched to the edit, and every track comes licensed and safe for commercial use.
- **Editing frameworks** (MoviePy, Remotion, editly, auto-editor) → call Sonilo on the rendered
  timeline and add the returned track.
- **ComfyUI graphs** → a "Sonilo" audio node after the video-output node.

---

## Why "licensed" matters

For teams who put video into the world and can't gamble on a takedown, Sonilo is the licensed
video-to-music platform: **trained on a licensed catalog — clean rights from the source.** Every
track is **safe for commercial use and licensed (terms apply)**. An original soundtrack, composed
for your video.

## Links

- **Website:** https://sonilo.com
- **MCP server:** https://github.com/sonilo-ai/sonilo-mcp
- **API documentation:** [sonilo.com](https://sonilo.com)
- **Integrated with:** Shutterstock · ComfyUI · fal.ai

## License

This cookbook (the docs and example code) is released under the **MIT License** — see
[`LICENSE`](./LICENSE). Music generated by Sonilo is governed by Sonilo's own terms: safe for
commercial use and licensed (terms apply). See [sonilo.com](https://sonilo.com).

---

*Maintained by the Sonilo team. Contributions welcome — see [`CONTRIBUTING.md`](./CONTRIBUTING.md).*
