# Turnstile — Architecture Notes

Turnstile is a **six-stage media generation pipeline with operator-gated publishing.** A
single Mac Mini runs one scheduling daemon that, unattended, takes a series format and
produces finished 9:16 vertical videos — an LLM-written script, generated stills, generated
motion clips, synthesized narration with word-level timing, an operator-supplied music bed,
and a React-rendered composition muxed into a final MP4. Publishing is a separate,
deliberate step the operator triggers.

**This repository is documentation only.** The implementation is private. Nothing here is
runnable code — it describes the architecture, the tradeoffs, and the reasoning.

The interesting engineering is not the media generation. It is the daemon orchestration, the
provider abstraction that lets vendors be swapped from config, the per-axis cost accounting,
and the graceful degradation strategy that kept the pipeline running through a live
control-plane migration.

---

## Architecture

```
        ┌───────────────────────────────────────────────────┐
        │  Control plane — PostgreSQL (Supabase)            │
        │  jobs · job_events · channels · provider_costs    │
        └───────────────────────┬───────────────────────────┘
                                │  reads config
                                │  writes lifecycle + cost
                                ▼
   ┌────────────────────────────────────────────────────────────┐
   │  Orchestrator daemon  — the only daemon   (10-min cycle)   │
   │  ┌──────────┐   ┌─────────────┐   ┌────────────────────┐   │
   │  │  sweep   │──►│  scheduler  │──►│    job runner      │   │
   │  │  stale   │   │  which      │   │  drives one job    │   │
   │  │  jobs    │   │  channels   │   │  through the       │   │
   │  │          │   │  are due    │   │  pipeline          │   │
   │  └──────────┘   └─────────────┘   └─────────┬──────────┘   │
   └─────────────────────────────────────────────┼──────────────┘
                                                 │
                                                 ▼
   ┌────────────────────────────────────────────────────────────┐
   │  Generation pipeline      (channel-blind, six stages)      │
   │                                                            │
   │      script ──► image ──┐                                  │
   │                  tts  ──┼── parallel ──► video ──► assemble│
   │                  bgm  ──┘                        (Remotion)│
   │                                                            │
   │  emits stage events ────────────────────► control plane    │
   └─────────────────────────────┬──────────────────────────────┘
                                 ▼
                       ready/<job_id>.mp4
                                 │
                                 ▼
   ┌────────────────────────────────────────────────────────────┐
   │  Publishing — OPERATOR-GATED, not part of the daemon cycle │
   │  a foreground console tool the operator runs deliberately  │
   └───────────────────────────┬────────────────────────────────┘
                               ▼
                          Instagram
```

The pipeline is **channel-blind by design**: nothing inside it knows which channel a job
belongs to, or that a control plane exists. It accepts a typed input, emits stage events,
and returns a typed result or raises a typed error. That single seam is what let the
orchestration layer be replaced wholesale without the engine noticing.

---

## Current state — what runs and what doesn't

Architecture documents tend to describe intent. This section describes the code.

| | |
|---|---|
| **Daemons** | **One.** A single `launchd` plist. Generation and scheduling live in one process; an earlier design had three. |
| **Generation stages** | **Six** — script, image, TTS, BGM, video, assemble. Numbered `s01`–`s07` with a gap: the procedural-graphics stage was absorbed into the render layer and its number retired. |
| **Rendering** | **Remotion** (Node + React) invoked as a subprocess, then one ffmpeg pass to mux the audio bed. An earlier three-pass ffmpeg assembler was deleted outright. |
| **Publishing** | **Operator-gated, Instagram only.** The daemon's upload path is not wired to a working publisher; the live path is a foreground console tool. Four-platform metadata generation exists, but only one platform can currently be published to. |
| **Cost accounting** | **Implemented, disabled.** Nine axes are stamped at every paid stage, but the config flag is off and a null tracker is installed in its place. |
| **Retries** | **None, anywhere.** There is no retry logic in the daemon or the engine. Every failure is terminal until an operator re-runs the job. |
| **Tests** | 70 test functions in the engine package. No CI, no linter, no type checking. |

The word *autonomous* applies to generation, not to publishing.

---

## Stack

| Layer | Technology |
|---|---|
| Orchestration, scheduling, pipeline | Python 3.11, asyncio |
| Control plane | PostgreSQL (Supabase), accessed directly — no ORM, no REST layer |
| Render layer | Node + React (Remotion), TypeScript |
| Audio mux, encoding | ffmpeg |
| Process supervision | launchd (macOS) |
| Operator UI | Gradio |
| Secrets | Hosted secrets manager, injected into the environment at process start |

---

## Documentation

| Document | Contents |
|---|---|
| [docs/pipeline.md](docs/pipeline.md) | The six generation stages and what each produces |
| [docs/orchestration.md](docs/orchestration.md) | Daemon model, scheduling, job state machine, failure handling |
| [docs/providers.md](docs/providers.md) | Provider abstraction — how vendors swap via config |
| [docs/cost-model.md](docs/cost-model.md) | Per-axis cost accounting and rate caching |
| [docs/migration.md](docs/migration.md) | Control-plane cutover and the fallback strategy |

---

## Design decisions

### 1. Code-side fallbacks carried the system through the control-plane migration

The control plane moved from a hosted document database to PostgreSQL. The complication was
that the old system's schema could not be extended programmatically — its API accepted
property additions and silently no-opped, so several fields the design required could not be
created without manual UI work by the operator.

Rather than block on operator availability, each missing field was given a **fallback in
version-controlled YAML**, under a strict precedence rule: the control plane won when the
property was present; the YAML answered when it wasn't.

The pipeline therefore degraded gracefully instead of failing on missing schema properties.
A half-migrated system stayed operational, each property could migrate independently, and
there was no coordinated cutover.

**Those fallbacks have since been retired.** PostgreSQL holds the columns directly, so the
`cost_axes` and `channels` YAML fallbacks were removed once the cutover completed — which
was always the intended end state. The pattern is worth describing precisely because it has
a defined end: a fallback that is never retired becomes a second, silently diverging
configuration system.

### 2. Provider abstraction allowing per-stage vendor substitution without code changes

Each paid capability is an **axis** — script LLM, image, video, text-to-speech, publishing.
Every axis has an abstract base class, one concrete implementation per vendor, and a factory
that reads a config file to decide which to construct.

Six abstract base classes are defined; six concrete implementations currently work. Two
axes have genuine alternatives in place — image can route through either a direct API or an
aggregator, and video likewise — so for those, switching vendor is a config edit and a
restart.

This matters because generative-AI vendors change pricing, deprecate models, and suffer
outages on their own schedule. The abstraction makes a vendor problem an operations decision
rather than an engineering project.

The honest limitation: the abstraction is a per-axis `if` chain in the factory rather than a
registry, and the pipeline reaches into the provider config for a few generation parameters
that ought to sit behind the interface. It is a real seam, not a perfect one.

### 3. Per-axis cost stamping making unit economics visible per generated asset

Every metered operation is attributed to a **canonical cost axis** — tokens in, tokens out,
images, video-seconds, speech characters, uploads. Rates live in a control-plane table, are
cached once per cycle into an in-memory rate sheet, and each job is stamped with a full cost
breakdown and usage record as it completes.

This turns a monthly invoice into per-asset unit economics, which is what establishes that a
single generated motion clip costs more than every other component of an episode combined.
That fact drives how content formats get designed — each scene is pushed down a cost ladder
from free graphics to a still to generated motion, and the accounting is what proves whether
it worked.

**It is currently switched off.** The machinery is implemented and wired; a config flag
installs a null tracker instead, and the rate table needs populating before the numbers mean
anything. The design is real; the telemetry is not running.

---

## A note on scope

The implementation — including all content formats, prompts, provider credentials, and
channel configuration — lives in a private repository. This repository documents the
architecture for people who want to understand how the system is put together. There is no
code here, and no attempt to make the system reproducible.
