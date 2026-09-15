# Orchestration

The orchestration layer decides *what* runs and *when*, drives each job through the
pipeline, and records everything. It is the half of the system that
knows about channels, schedules, lifecycle states, and platforms — none of which the
generation pipeline knows anything about.

## Daemon model

Production runs on a single Mac Mini with daemons supervised by **launchd**. launchd was
chosen over a container orchestrator for an unglamorous but decisive reason: the workload is
one machine, generation is GPU- and API-bound rather than horizontally scalable, and the
render layer benefits from hardware video encoders that are simplest to reach on bare metal.
A scheduler that keeps processes alive and restarts them on failure is the entire
requirement.

The system runs as a **single orchestrator daemon**, alongside a local operator panel that is
not a daemon. It did not start that way. Generation, publishing, and the operator dashboard
were originally three separate supervised processes; publishing was folded into the
orchestrator's cycle during the control-plane migration described in
[migration.md](migration.md).

Consolidating was the right call. As separate processes, generation and publishing each
opened their own control-plane connection, each built their own rate sheet, and coordinated
only through the database and a shared directory of finished renders — which meant two
processes could hold different views of the same job, and a publish could begin against state
the generator was still writing. Merging them removed an entire class of race condition and a
duplicate configuration load, at the cost of nothing that was actually being used: the two
never ran concurrently in any meaningful sense, because the upload pass was idle whenever
generation was not producing anything.

The general lesson is that process boundaries should follow genuine concurrency or genuine
failure isolation. These followed neither — they were separate because they were written at
different times.

## No queue

The control plane is a **system of record, not a message queue.** This is a deliberate
choice worth explaining, because a job queue is the obvious default here and it was
rejected.

Creating a job record is the first step of that job's lifecycle, not a request for someone
else to pick it up. The same call that records the job drives it forward to completion. There
is no write-a-row-then-poll-it-back round trip, no dispatcher, no worker pool, and no
visibility gap between "queued" and "running."

The reasoning: throughput is roughly seven jobs a day. A queue buys parallel consumers and
backpressure, neither of which this workload needs, and charges for them in components that
can fail independently and states that can disagree. Direct invocation means the scheduler, a
manual CLI trigger, and a tooling-side trigger all enter through exactly one function, and
the database describes what happened rather than coordinating it.

## The cycle

The orchestrator wakes on a fixed interval and does four things in order:

1. **Sweep stale jobs.** Any job left in a mid-generation state past a timeout is marked
   failed — see *Crash recovery* below.
2. **Reload provider configuration.** Vendor selection is re-read each cycle so config edits
   take effect without a restart where possible.
3. **Schedule and generate.** For each channel found due, run one job to completion.
4. **Upload pass.** Intended to publish any finished video whose posting window has
   opened. This step is currently inert — see *Publishing* below.

Direct CLI entry points exist for each of these — run one job now, run one upload pass, run
one full cycle — so an operator or tooling can trigger work without waiting for a tick, using
the identical code path the daemon uses.

## Scheduling

A channel is due if and only if **all** of the following hold: it is active, its format
specification exists and is marked published, no job for it is already in flight, and its
posting cadence has elapsed. Cadences are simple and declarative — daily, every other day,
weekdays only — evaluated in UTC against a per-channel preferred post time.

The in-flight check is the interesting one. "In flight" includes the **finished-but-unposted**
state, not just the generating states. A rendered video awaiting its posting window still
occupies its channel, which prevents the system from stacking up a backlog of renders for a
channel whose window has not opened. One channel, one job, at a time.

## Job state machine

```
QUEUED → SCRIPTED → IMAGE_GEN → VIDEO_GEN → AUDIO_GEN → ASSEMBLED → READY → UPLOADED
                                                                   │
                     any caught exception at any point ────────────┴──► FAILED
```

States are constrained at the database level rather than by convention. The mapping from
pipeline stage events to these names lives in one listener class on the orchestrator side —
the single place that knows both vocabularies. Adding a pipeline stage means teaching that
listener about it; the stage itself stays ignorant.

Alongside the current state, every stage event is appended to an **immutable event log** with
its phase, timestamp, and structured detail. The job row answers "where is this now"; the
event log answers "what happened to it, when, how long did each stage take, and why did it
fail." The log cascades on job deletion, so it cannot outlive its subject.

## Failure handling

**Fail loud, never retry internally.** Every external API failure raises a typed error that
propagates to the top of the job runner, which records the failure and stops. The engine does
not retry, and this is intentional: every call is metered, so an automatic retry is an
automatic unrequested charge. A retry policy that spends money must be an explicit decision,
so the only path back into the queue is an operator action.

Errors are a typed tree rather than strings, which lets the orchestrator decorate each class
usefully — configuration errors, transient provider errors, authentication errors, rate limits
carrying a retry-after, content refusals, output that succeeded but is unusable, assembly
failures, and schema-validation failures. Assembly errors in particular carry enough structured
detail to triage a render failure without reproducing it.

### Publishing is operator-gated, and currently single-platform

**The daemon does not publish.** This is the largest gap between the system's design and its
current state, and it is worth stating plainly rather than describing the intent.

The design called for an upload pass inside the daemon cycle, publishing to each platform
independently so that one platform's failure would not block the others — a video reaching
three of four platforms would keep its successes and retry only the failure. Per-platform
metadata generation for four platforms exists and works.

What actually runs is narrower. The daemon's upload path constructs a publisher that was
retired during a vendor SDK migration and now raises on construction, so that path is dead. The
working publisher lives behind a **foreground console tool the operator runs deliberately**, and
its supported-platform tuple currently contains **one entry: Instagram**.

The practical effect is a clean split: generation is unattended and runs on a timer; publishing
is a manual step. The independent-per-platform failure model described above is the design the
code is being moved toward, not a property the system has today.

### Crash recovery

Generation is **not resumable.** A job interrupted mid-generation has partial artifacts, partial
spend, and no safe resumption point, so at the start of every cycle any job sitting in a
generation state past a timeout is swept to failed with an explicit reason. The operator re-runs
it deliberately.

Finished-but-unposted jobs are never swept. They are completed renders legitimately waiting for
a posting window, and sweeping them would destroy work that has already been paid for.
