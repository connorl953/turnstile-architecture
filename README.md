# Turnstile — Architecture Notes

Turnstile is an autonomous short-form video pipeline. A single Mac Mini, unattended,
generates roughly seven finished 9:16 videos a day — one per channel — and publishes them
to four platforms without a human in the loop. Each episode is assembled from an
LLM-written script, generated stills, generated motion clips, synthesized narration with
word-level timing, and a background music bed, then rendered and muxed into a final MP4.
The operator's only routine interaction is editing configuration and reading a dashboard;
the system schedules, generates, fails loudly, and publishes on its own.

**This repository is documentation only.** The implementation is private. Nothing here is
runnable code — it describes the architecture, the tradeoffs, and the reasoning.

The interesting engineering here is not the media generation. It is the daemon
orchestration, the provider abstraction that lets any vendor be swapped from config, the
per-axis cost accounting that makes unit economics visible per asset, and the graceful
degradation strategy that kept the pipeline running through a live control-plane
migration.

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
   │  Orchestrator daemon                   (10-minute cycle)   │
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
   │  Generation pipeline      (channel-blind, seven stages)    │
   │                                                            │
   │      script ──► image ──┐                                  │
   │                  tts  ──┼── parallel ──► video ──► assemble│
   │                  bgm  ──┘                                  │
   │                                                            │
   │  emits stage events ────────────────────► control plane    │
   └─────────────────────────────┬──────────────────────────────┘
                                 ▼
                       ready/<job_id>.mp4
                                 │
                                 ▼
   ┌────────────────────────────────────────────────────────────┐
   │  Upload pass — each platform independently fallible        │
   └────┬──────────────┬──────────────┬──────────────┬──────────┘
        ▼              ▼              ▼              ▼
    YouTube        Instagram      Facebook        Reddit
     Shorts          Reels          Reels
```

The pipeline is **channel-blind by design**: nothing inside it knows which channel a job
belongs to, or that a control plane exists at all. It accepts a typed input, emits stage
events, and returns a typed result or raises a typed error. That single seam is what makes
the orchestration layer replaceable — and it is why the control-plane migration described
in [docs/migration.md](docs/migration.md) never required touching the engine.

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

**Scale:** one orchestrator daemon · seven pipeline stages · four publishing platforms ·
~7 channels · ~210 videos/month.

---

## Documentation

| Document | Contents |
|---|---|
| [docs/pipeline.md](docs/pipeline.md) | The seven generation stages and what each produces |
| [docs/orchestration.md](docs/orchestration.md) | Daemon model, scheduling, job state machine, failure handling |
| [docs/providers.md](docs/providers.md) | Provider abstraction — how vendors swap via config |
| [docs/cost-model.md](docs/cost-model.md) | Per-axis cost accounting and rate caching |
| [docs/migration.md](docs/migration.md) | Control-plane cutover and the fallback strategy |

---

## Design decisions

Three choices that shaped the system, and the reasoning behind each.

### 1. Fallbacks maintained code-side during the control-plane migration

The control plane moved from a hosted document database to PostgreSQL. The complication
was that the old system's schema could not be extended programmatically — its API accepted
property additions and silently no-opped, so several fields the design called for could not
be created without manual UI work by the operator.

Rather than block the pipeline on operator availability, every field that might be absent
was given a **code-side fallback in version-controlled YAML**, under a strict precedence
rule: the control plane wins when the property is present, and the YAML answers when it is
not.

The pipeline therefore degraded gracefully instead of failing on missing schema properties.
A half-migrated system stayed fully operational, each property could be migrated
independently on the operator's own schedule, and there was no coordinated cutover and no
downtime. When a property finally landed, its fallback became dead weight rather than a
breaking change.

The cost of this decision is worth naming: two sources of truth for a period, and fallbacks
that must be deliberately retired or they rot. That was the right trade against an
unattended pipeline failing silently overnight.

### 2. Provider abstraction allowing per-stage vendor substitution without code changes

Each paid capability is an **axis** — script LLM, image, video, text-to-speech, background
music, publishing. Every axis has an abstract base class, one concrete implementation per
vendor, and a factory that reads a config file to decide which to construct.

Switching the vendor behind any stage is a config edit and a daemon restart. No code change,
no redeploy. Adding a vendor means subclassing that axis's base class, dropping the file in,
and registering it.

This mattered more than it sounds. Generative-AI vendors change pricing, deprecate models,
and suffer outages on their own schedule. The abstraction made a vendor problem an
operations decision rather than an engineering project, and it made it cheap to route the
same axis through a direct API or through an aggregator depending on which was currently
cheaper or more reliable.

### 3. Per-axis cost stamping making unit economics visible per generated asset

Every metered operation is attributed to a **canonical cost axis** — tokens in, tokens out,
images, video-seconds, speech characters, music-seconds, uploads. Rates live in a
control-plane table, are cached once per cycle into an in-memory rate sheet, and each job is
stamped with a full cost breakdown and usage record as it completes.

This turns a vague monthly invoice into per-asset unit economics. It is what makes it
possible to establish that a single generated video clip costs more than every other
component of an episode combined, by multiples — and that fact, once measurable, drives the
design of every content format built on the system. Formats are authored to push each scene
down a cost ladder, and the accounting is what proves whether that worked.

Rates are data, not code, precisely because rates rot. The axis taxonomy is stable; the
numbers behind it are expected to change without a deploy.

---

## A note on scope

The implementation — including all content formats, prompts, provider credentials, and
channel configuration — lives in a private repository. This repository exists to document
the architecture for people who want to understand how the system is put together. There is
no code here, and no attempt to make the system reproducible.
