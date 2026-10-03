# Agent orchestration — design notes

How I run AI coding agents in production, and what measuring them actually showed.

This repository is a **write-up, not the engine**. The engine runs real client work and stays
private. What's here is the architecture, the protocol, and the numbers — enough to judge the
reasoning, and enough to rebuild it.

---

## The problem

A coding agent is a non-deterministic worker that is confidently wrong a predictable fraction
of the time. You cannot fix that from inside the agent. Prompting harder does not fix it;
a bigger model moves the rate without changing the shape.

So the question is not *"how do I get a better agent"* but **"what wrapper makes a
non-deterministic worker safe to depend on?"**

The answer that worked: **a deterministic control plane around non-deterministic workers.**
The agent writes code. The *engine* decides what counts as done.

---

## Architecture

```
          requirements interview
                   │
                   ▼
            north-star doc ──────────────┐  anchors every later decision
                   │                     │  (append-only; cited, not rewritten)
                   ▼                     │
          atomic slice queue ◄───────────┘
                   │           (dynamic: new work appended at batch
                   │            boundaries, never mid-batch)
                   ▼
        ┌──── coordinator ────┐           deterministic
        │   (owns the truth)  │           ───────────────
        └──┬───────────────┬──┘
           │               │              non-deterministic
      ┌────▼────┐    ┌─────▼─────┐        ─────────────────
      │implement│    │ validate  │        separate seats,
      │  seat   │    │   seat    │        never the same model
      └────┬────┘    └─────┬─────┘
           │               │
           └──── gate ◄────┘
                   │
         ┌─────────┴─────────┐
      landed              escalate
                      (new model / smaller slice)
```

### Four rules that do the work

**1. Slices are atomic.** Every non-test unit of work is under ~30 minutes, 1–3 tests, tight
diff. Anything bigger is a split trigger, not a challenge. This is not an aesthetic
preference — see [the context-length measurement](docs/measurements.md#context-length).

**2. The validator is never the implementer.** Structurally, not advisorily. For hard tasks the
topology escalates from two agents to three (coordinator ≠ implementer ≠ validator)
automatically. A model grading its own work is not a review, and this holds whether the
reviewer is a model or a person.

**3. Completion is a protocol, not a sentence.** An agent does not "say" it finished. It emits
a parseable callback with a status, and for a validation it emits a verdict record with a
defect class, the failed gate, and evidence paths that are hashed at submission. A claim of
success is checkable or it is not a claim. See [the protocol](docs/protocol.md).

**4. The report is a consequence of what happened, never a claim about it.** State is derived
from the record — the journal, the commit, the appended callback. Never from a regex over
prose, never from a hardcoded assumption, never from a file that may not exist. An orchestrator
whose self-report cannot be trusted cannot be pointed at a client project, because the
operator's only view of the work *is* the report.

---

## Completion is idempotent, not atomic

The honest version, because the dishonest version loses work.

You cannot atomically "finish" a slice: committing code and recording that you committed it are
two different writes to two different places, and the process can die between them. So:

```
  journal intent  ──►  commit  ──►  single locked append  ──►  repair on rerun
```

On rerun the wrapper looks for a slice-named commit since the recorded base. If it finds one
but no callback, it **appends the missing callback and does not commit again**. Crash-safe
without double-committing, and without the "did it work?" ambiguity that makes an operator
re-run things by hand.

The append is a single locked write, so two seats racing cannot interleave a callback line.

---

## Failure handling

| Failure | Response |
|---|---|
| Agent finishes without calling back | Idle detector catches it in ~1 minute (this was the overnight-stall class of bug) |
| Agent loops | Hard attempt cap — justified by [the flat retry curve](docs/measurements.md#the-retry-curve-is-flat) |
| Agent drifts | Time-budget check-ins with hard caps; drift surfaces instead of burning the night |
| Agent writes outside its slice | Landlock write-fence around the runtime |
| Validator passes a broken change | Validators are pre-scored against frozen known-bad diffs; see [the reviewer finding](docs/measurements.md#my-ai-reviewer-was-rubber-stamping) |
| Model unreachable | Router with failover — a model served on two machines survives one going down |

---

## Models

**Frontier:** Claude (Opus/Sonnet) · OpenAI Codex · Grok.
**Self-hosted:** GLM · DeepSeek · Qwen and other open models — ten seats across two GPU boxes
on vLLM, behind a router.

Every model is an interchangeable **seat** behind one dispatch protocol, so routing is a
configuration decision rather than a code change.

Planners, implementers and validators are benchmarked **separately**, because they are
different jobs. Measured: one model planned 5/5 and validated near-chance; another was the
reverse. Routing is derived from measurement.

---

## What this is not

- **Not an autonomous-agent framework.** The agents are not in charge. If you want an agent
  that decides what to work on, this is the wrong shape.
- **Not a prompt library.** Prompts here are versioned artifacts with a linted invariant
  (see [protocol](docs/protocol.md#role-cards)), but prompting is the small part.
- **Not a benchmark claim about models in general.** Every number in
  [measurements](docs/measurements.md) came off my own harness, on my own task distribution,
  and is reported as such.

## Read next

- **[docs/protocol.md](docs/protocol.md)** — the callback and verdict grammar, role cards, gates
- **[docs/measurements.md](docs/measurements.md)** — what measuring the agents actually showed

## Related public repos

- **[kloo](https://github.com/lokalhub/kloo)** — the agent coding driver (Go) this dispatches to
- **[Helm](https://github.com/SilverJROM/Helm)** — an agent-orchestration application
