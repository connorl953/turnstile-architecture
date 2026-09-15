# Control-plane migration

The control plane is the system's configuration and state: which channels exist, what they
are configured to produce, what every job did, and what each vendor charges. It started as a
hosted **document database** and was migrated to **PostgreSQL**. This document covers why,
and — more usefully — how the pipeline kept running throughout.

## Why the original choice was right

The document database was not a mistake. It was a good early decision for a specific reason:
it gave the operator a **usable editing UI for free**. Channel configuration, format prompts,
and vendor rates are all operator-edited data, and a spreadsheet-like interface that someone
can open on a phone and edit is genuinely valuable. Building an equivalent admin UI on day one
would have cost weeks and delivered nothing the document database did not already provide.

## Why it stopped fitting

Four problems accumulated, in roughly increasing order of severity:

**Query expressiveness.** Questions the system genuinely needed to answer — which jobs are in
flight for this channel, what has this format already produced, what did the last thirty days
cost by axis — are joins and aggregates. The document API made each of them a paginated fetch
plus client-side filtering.

**Latency at cycle boundaries.** Every cycle read channel configuration and the rate sheet over
HTTP before doing any work. Fine at seven channels; visibly wrong as a foundation.

**No transactional integrity.** A job's lifecycle write and its cost stamp were separate API
calls that could not be made atomic, so an interruption between them left a row that was
internally inconsistent.

**The decisive one: the schema could not be extended programmatically.** The integration layer
accepted requests to add properties to existing databases, returned success, and silently did
nothing. Several fields the design required could not be created except by a human clicking
through a UI. A control plane that cannot be migrated by code is a control plane that blocks
every feature depending on a new field on operator availability.

That last point is what turned a "someday" migration into a necessary one — and it is also what
shaped the fallback strategy, because the same limitation applied *during* the migration.

## The fallback strategy

The constraint was absolute: **the pipeline could not stop.** It runs unattended across several
channels, a stalled cycle means missed posts, and a broken cycle at 2am means nobody notices
until morning. A maintenance window followed by a big-bang cutover was not available.

The strategy was to make every possibly-absent field answerable from two places under one strict
precedence rule:

```
   read a property
        │
        ├── present in control plane?  ──► use it          (control plane always wins)
        │
        └── absent or empty?           ──► use the code-side
                                           YAML fallback
```

Fallbacks lived in version-controlled YAML next to the code, populated with correct values for
every live channel and format. The rule had no exceptions and no merging: if the control plane
had an answer, the YAML was ignored entirely.

Four properties were carried this way — a format-to-channel binding, an acknowledgement flag for
dismissing handled failures, a music-reuse tag pinning a track to a channel's identity, and the
cost-axis mapping that links a vendor's rate row to the accounting taxonomy.

### What this bought

**The pipeline degraded rather than failed.** A missing property produced a sensible default and a
finished video, not an exception and a dead cycle. Some functionality was reduced — an
acknowledgement flag that reads as false makes a dashboard button a no-op — but reduced
functionality with video still shipping is categorically different from a stopped pipeline.

**Migration became per-property and incremental.** Each field could move independently, whenever
the operator got to it, in any order. There was no coordinated cutover, no ordering dependency
between properties, and no moment where the system was half-migrated and therefore broken. The
half-migrated state *was* a supported state, for months.

**Rollback was free.** If a migrated property turned out wrong, clearing it in the control plane
restored the fallback path immediately. No deploy, no revert.

**Retirement was a cleanup, not a breaking change.** Once a property was reliably present, its
fallback stopped being consulted. Removing the dead YAML is a tidying task that can happen at
leisure; it cannot break anything, because nothing reads it any more.

### What it cost

Two sources of truth for a period, and the discipline to actually retire fallbacks instead of
letting them rot into a second, silently diverging configuration system. This is a real
maintenance burden and the reason the pattern should not be reached for casually. It was correct
here because the alternative was an unattended production pipeline failing silently overnight.

## The PostgreSQL shape

The migrated control plane is four tables: **jobs** (one row per episode, lifecycle-constrained at
the database level, carrying cost breakdown and usage), **job_events** (append-only per-stage log,
cascading on job deletion), **channels** (operator-edited configuration, read-only to the daemon),
and **provider_costs** (the vendor rate sheet).

Two decisions are worth naming.

**No ORM, no REST layer.** The adapter speaks to Postgres directly. The query surface is small and
well understood, the schema is owned by this system alone, and an ORM would add a translation layer
and a migration framework in exchange for very little.

**Format definitions stayed in code, deliberately.** Meta-prompts, visual style, and voice selection
live in version-controlled YAML files, not in the database. Only *operational state* — jobs, events,
channel config, rates — is in the control plane. The distinction is between things that are edited
(operational state, belongs in a database with a UI) and things that are authored (format definitions,
belong in version control with review and history). Putting authored content in a database would mean
losing diffs, review, and the ability to ship a format as a reviewable artifact.

Schema changes are numbered SQL migration files applied by a small migrate command — so unlike the
system it replaced, a schema change is now code, executable without any manual operator step. Which
was the whole point.
