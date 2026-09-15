# Cost model

The system spends real money on every job, across six vendors, on a schedule nobody is
watching. Cost accounting is therefore not a reporting feature bolted on at the end — it is a
first-class part of the architecture, and it changes how content formats get designed.

## Canonical cost axes

Every metered operation is attributed to a **canonical axis**. Axes are named after the unit
that is actually billed, not after the vendor or the pipeline stage:

```
llm.script.tokens_in       llm.script.tokens_out
llm.metadata.tokens_in     llm.metadata.tokens_out
image                      video.seconds
tts.chars                  bgm.seconds
upload
```

Naming by billed unit is what makes the taxonomy survive vendor changes. Swapping the image
vendor does not add, remove, or rename an axis — it changes the rate attached to one. The axis
set is stable; the numbers behind it are not.

Note that script and metadata generation are separate axes despite both being LLM calls on
possibly the same vendor. They are separate because they answer different questions: one scales
with script complexity, the other with the number of destination platforms, and conflating them
would hide which one moved.

## Rate sheet and caching

Rates live in a **control-plane table**, one row per provider, carrying the axis, the price, the
unit, and a unit size — the divisor that turns a quoted price like "per million tokens" into a
per-unit rate.

At the start of each cycle the orchestrator loads that table once into an in-memory rate sheet
keyed by axis, and every job in the cycle stamps against that snapshot. This is a deliberate
consistency choice as much as a performance one: every job in a cycle is priced against
identical rates, so a rate edit mid-cycle cannot produce two jobs that disagree about what an
image cost.

Missing rates are surfaced as a warning at cycle start rather than as a failure at stamping
time. An unpriced axis should not stop a pipeline from producing video.

**Rates are data, not code.** They live in the database and are edited by the operator from each
vendor's pricing page — no deploy, no code review, no release to reprice. The one thing not
permitted is a rate card checked into the repository, because a stale rate card is worse than no
rate card: it looks authoritative and quietly lies.

Cost tracking is itself toggleable. When disabled, a null tracker satisfies the same interface —
no rate query, no cost writes, no per-job stamp — so the pipeline runs identically during testing
phases without accumulating meaningless cost data.

## Per-job stamping

As a job completes, its row receives a full **cost breakdown** and a **usage record** alongside
the total. Usage is kept separately from cost, and that separation matters: usage is a physical
fact about the job that stays true forever, while cost is that usage multiplied by a rate that was
current on a particular day. Keeping both means historical jobs can be repriced against new rates
to answer "what would last month have cost at today's prices" — which is exactly the question a
vendor's price change raises.

## The cost hierarchy

The accounting exists to support one insight, which is the thing that actually drives design:

```
CG      — FREE        Rendered declaratively. Costs nothing per episode.
LLM     — TRIVIAL     Script and metadata generation. Never worth optimizing.
image   — MODEST      A generated still. Real, but small.
video   — DOMINANT    A generated motion clip. More than everything
                      else in the episode COMBINED, by multiples.
```

The load-bearing sentence is the last one. **A single generated motion clip costs more than all
the graphics, language-model calls, and stills of an episode put together.** Once that is
measured rather than assumed, the correct instincts follow without further analysis:

1. **The count of motion scenes is a format's entire cost shape.** Estimating what a format costs
   means counting one thing.
2. **Never optimize language-model or image spend.** They are rounding error, and they are where
   the viewer's actual experience lives. Spend freely there.
3. **Motion is opt-in, never default.** The question for each scene is whether the beat *needs*
   generated motion — not whether it could have it. Intros, outros, call-to-action cards, held
   reveals, and establishing shots do not.

## Design consequences

**Formats are authored down a cost ladder.** For every scene: can this be free graphics? If not,
can it be a still with a camera move? Only if neither works does it generate motion. A new
reusable render component that lets a beat drop from motion to graphics is a permanent cost win
across every future episode, and gets proposed on those grounds.

**Motion generations are always the provider's minimum billable duration.** Longer generations cost
proportionally more for the same beat, so a scene that plays longer than the minimum is stretched
at render time — speed ramp, frame interpolation, freeze-frame hold, or loop. A timeline gap is
never closed by generating more video. This rule alone is worth a large fraction of the per-episode
cost.

**A new pipeline stage that calls a paid API is a standing tax.** It applies to every episode, on
every channel, forever. A format runs hundreds of times, so a choice that adds one paid call per
episode is not a small decision, and it gets stated in those terms when proposed.

**Reasoning happens in orders of magnitude, not dollars.** The hierarchy above is durable; specific
figures rot within months and invite arguing over cents. Design conversations use the hierarchy.
When an actual dollar figure is genuinely needed, it is queried live from the rate table rather
than recalled.
