# Provider abstraction

Every paid external capability in the system is modelled as an **axis**. An axis is a
capability the pipeline needs, not a vendor that supplies it:

| Axis | Capability |
|---|---|
| `llm` | Script generation and per-platform metadata generation |
| `image` | Still-frame generation |
| `video` | Short motion-clip generation conditioned on a still |
| `tts` | Narration synthesis with word-level timestamps |
| `audio` | Background music generation |
| `publishing` | Delivery to each destination platform |

## The shape

Each axis has three parts:

1. **An abstract base class** defining the axis contract — what goes in, what comes out, and
   which typed errors may be raised. The pipeline is written against this and nothing else.
2. **One concrete implementation per vendor**, each a self-contained wrapper that translates
   the vendor's API, response shape, and failure modes into the axis contract.
3. **A factory** that reads a config file and constructs the implementation named there.

The pipeline never imports a vendor module. It asks the factory for the provider on an axis
and receives something satisfying the contract.

```
   pipeline stage
        │
        │  "give me the provider for axis X"
        ▼
    factory ─────── reads ──────► providers config (YAML)
        │
        │  constructs
        ▼
   ┌────────────────────────────────────────┐
   │  vendor A impl   vendor B impl   ...   │   all satisfy the axis base class
   └────────────────────────────────────────┘
```

## Swapping a vendor

Edit the config file, restart the daemon. That is the whole procedure. No code change, no
redeploy, no touching the pipeline.

Adding a new vendor is three mechanical steps: subclass the axis's base class, drop the file
into the providers package, and register it in the factory and the config schema. The work is
confined to the new file — no existing provider, stage, or composition is modified. This is a
specific instance of the system's broader additive-by-default posture: a change that only adds
options cannot break running jobs.

## Why this earns its complexity

An abstraction layer over six vendors is not free, and in many systems it would be premature.
Here it pays for itself for reasons specific to the generative-AI vendor landscape:

**Pricing moves constantly, and it moves a lot.** The same capability can differ several-fold
in price between vendors and can reprice on a month's notice. Because the cost model
(see [cost-model.md](cost-model.md)) attributes spend per axis, the system can identify exactly
which axis to reprice — and the abstraction makes acting on that a config edit.

**Direct API versus aggregator is a live tradeoff, per axis, over time.** Some axes route
through a vendor's own API for the lower rate; others route through an aggregator for better
availability or simpler auth, accepting a markup. Which is correct changes. The abstraction
makes it a reversible decision rather than a commitment, and in practice several axes keep both
paths implemented so the choice is one config line.

**Models get deprecated on the vendor's schedule, not yours.** An unattended pipeline meets a
sunset notice as an overnight outage. Being able to fail over to an already-implemented
alternative at 2am without a deploy is the difference between a lost night and a lost week.

**Failure modes differ wildly and must be normalized.** One vendor signals a rate limit with a
429 and a retry-after header; another returns 200 with an error body; a third returns a
successful-looking response containing unusable output. Each wrapper's real job is translating
that mess into the shared typed-error taxonomy, so the orchestrator's failure handling is
written once rather than per vendor.

## Configuration, not credentials

The providers config records **which** vendor and model each axis uses. It contains no
credentials. Secrets are fetched at process start from a hosted secrets manager and injected
into the environment; providers read them on first use. In development the same interface is
satisfied by a local environment file, and the bootstrap is a no-op — so there is exactly one
code path for credentials in both environments, and the production path never has secrets at
rest in the repository.

## One deliberate gap

There is no automatic failover between providers on an axis. A vendor failure raises a typed
error and the job fails loudly; it does not silently retry against the alternative
implementation.

This follows the same reasoning as the no-internal-retry rule in
[orchestration.md](orchestration.md): every provider call is metered, so an automatic failover
is an automatic unrequested charge, potentially at a materially different rate than the one
budgeted. Making the switch an operator decision keeps spend intentional. The alternative
implementation is already there and one config line away — what is withheld is the system's
authority to spend money on its own initiative.
