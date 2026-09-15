# The generation pipeline

The pipeline turns one job into one finished MP4. It is a self-contained package with a
single entry point, and it is **channel-blind**: nothing inside it knows which channel a
job belongs to, what the control plane is, or what the job's lifecycle states are called.

That constraint is load-bearing. The pipeline accepts a typed input object, emits stage
events as it works, and returns a typed result or raises a typed error. Everything about
scheduling, persistence, status vocabulary, and publishing lives on the other side of that
seam. The engine is testable standalone and the orchestration layer around it was replaced
wholesale without the engine noticing.

## The seven stages

Stages are numbered `s01` through `s07`. The numbering is stable even as the set evolves —
`s06` was a procedural-graphics stage in an earlier version, later absorbed into the render
layer as reusable components, and its number was retired rather than reused.

| # | Stage | What it does | Output |
|---|---|---|---|
| s01 | `script` | An LLM expands a format's meta-prompt into a structured multi-scene script as validated JSON. Schema failure is a typed error, not a retry. | Script JSON |
| s02 | `image` | Generates a first-frame still per scene, at a generation resolution that is then resampled to the 1080×1920 delivery frame. | One PNG per scene |
| s03 | `tts` | Synthesizes narration per script segment, returning **word-level timestamps** alongside the audio. | Per-segment audio + alignment manifest |
| s04 | `bgm` | Background music generation. Three modes: generate a track, use a preset, or reuse a track pinned to a channel's identity. | One music track |
| s05 | `video` | Generates a short silent motion clip per scene that needs motion, conditioned on that scene's first frame from s02. | One MP4 per motion scene |
| s06 | *(retired)* | Procedural graphics — countdowns, wheels, score cards. Now implemented as declarative components in the render layer, where they cost nothing to produce. | — |
| s07 | `assemble` | Renders the composition in the Node render layer via subprocess, builds the audio bed by concatenating narration and mixing music underneath, then muxes audio onto the silent render in a single stream-copy pass. | Final MP4 + sidecar metadata |

**Stages s02, s03, and s04 run in parallel.** Images, narration, and music have no
dependency on one another. s05 depends on s02 because each motion clip is conditioned on
its scene's still. s07 depends on everything.

## Scene sources — the cost lever

Not every scene costs the same to produce, and the difference is not marginal. Each scene
declares a `source`, and that single field is the largest cost decision a format author
makes:

| Source | Upstream generation | Relative cost |
|---|---|---|
| `cg` | None — rendered declaratively in the render layer | Free |
| `still` | Image only; motion supplied by a Ken Burns move at render time | Cheap |
| `ai` | Image **and** a generated motion clip | Dominant |

The pipeline generates a clip only for `ai` scenes; `still` and `cg` scenes skip s05
entirely. A scene's source and the composition's component for that slot must agree — a
mismatch either bills for a clip that never gets shown or breaks the render by requesting a
clip that was never generated.

There is a related rule that falls directly out of the cost model: **a motion generation is
always the provider's minimum billable duration**, and a scene that needs to play longer is
stretched at render time — speed ramp, frame interpolation, freeze-frame hold, or loop —
rather than generated longer. A fifteen-second beat is a minimum-length generation plus ten
seconds of stretch. See [cost-model.md](cost-model.md).

## Stage events

Every stage emits a `started` and `completed` event. The orchestrator subscribes to these
and translates them into lifecycle states and an append-only event log — but the translation
lives entirely on the orchestrator's side. The engine never references a lifecycle state
name; doing so would be a contract violation, because it would couple the engine to a
control plane it is specifically designed not to know about.

This is what makes per-stage timing observable for free: the event log records what
happened, when, and how long it took, for every job, without the engine participating in
observability at all.

## The render bridge

Stage s07 is the only place two runtimes meet. The Python side is deliberately a thin shim:
it builds the props the composition expects — script, asset paths, and audio timing including
word-level alignment — invokes the Node renderer as a subprocess, and parses a single line of
structured JSON status from stdout.

Everything visual lives on the Node side as React components: layouts, transitions, overlays,
filters, and the procedural graphics that used to be `s06`. Word-level caption timing comes
from the s03 alignment data, which is why captions land on the word rather than the sentence.

A failure in the render subprocess surfaces as a typed assembly error carrying the exit code,
which internal stage failed, a truncated stderr tail, and the composition identifier — enough
to triage from the error log without reproducing the render.
