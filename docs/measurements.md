# What measuring the agents showed

Every number here came off my own harness, on my own task distribution. They are not
general claims about these models; they are the numbers that changed how I build. Where a
finding is uncomfortable, it is reported anyway — those were the useful ones.

---

## Headline

| | |
|---|---|
| Slices landing with no human touch | **~70%** |
| Pass rate by attempt | 17% → 17% → 25% → 27% |
| Best local seat's protocol compliance | **82%** |
| AI validator true-negative rate on real broken diffs | **36%** |
| Share of wall clock in decode | **~97%** |
| Quality vs context | −11% @ 32k · −40% @ 131k · −56% @ 254k |
| Prompt:completion ratio on a prompt-heavy workload | **162:1** at 5.8% cache hit |

---

## The retry curve is flat

Pass rate by attempt number: **17%, 17%, 25%, 27%.**

The naive reading of an agent failure is "try again." The curve says a second attempt is worth
essentially nothing, and the modest lift at attempts three and four comes from *changing
something* — a different model, a smaller slice — not from repetition.

**What changed:** a hard attempt cap, and escalation on exhaustion rather than looping. An
agent in a retry loop is not converging, it is spending.

This is also the cleanest answer I have to *"when is a deterministic workflow better than an
autonomous agent?"* An autonomous agent's core move on failure is to try again. If retrying
doesn't work on your task distribution, autonomy is the wrong shape, and you need something
outside the agent making the escalation decision.

---

## My AI reviewer was rubber-stamping

Scored validator models against a frozen set of diffs with known defects.

**True-negative rate — the share of genuinely broken diffs correctly rejected — came in at
36%** for a configuration whose aggregate accuracy looked fine. It was passing nearly
two-thirds of real defects. Aggregate accuracy hid it completely, because the corpus had more
clean diffs than broken ones and agreeing with "clean" is cheap.

**What changed:**

- Validators are scored on **Matthews correlation coefficient**, not accuracy. MCC collapses
  on exactly the degenerate behaviour accuracy rewards.
- Every validator is pre-scored against known-bad diffs before it is trusted with anything.
- Two defective packets in the scoring set were themselves wrong; removing them moved scores
  materially — **so the answer key gets audited too.** A benchmark is an artifact that can be
  broken, and a wrong answer key silently punishes the models that got it right.

The result worth knowing: a **27B self-hosted open model matched a frontier model** on
validation once scored properly. The expensive model was not buying accuracy on this task. You
only find that out by measuring, and you only believe it if your metric can't be gamed by
agreeing with the majority class.

---

## Context length

Output quality against context length, same task, same model:

| Context | Degradation |
|---|---|
| 32k | −11% |
| 131k | −40% |
| 254k | −56% |

Long context is advertised as capacity. Measured, it behaves like a budget you spend quality
out of.

**What changed:** this is the real justification for atomic slices. "Keep tasks small" is
usually an aesthetic argument; here it is a measured one. A big slice does not just risk a
bigger diff — it is executed by a measurably worse worker.

---

## Where the time goes

**Decode is ~97% of wall clock.**

Prefill, scheduling, orchestration overhead and network are, together, noise. Any optimisation
that is not reducing output tokens or raising decode throughput is theatre.

Corollary that paid for itself: one model produced equivalent accuracy with **~5× fewer output
tokens**. At 97% decode, that is a ~5× wall-clock win on the dominant term — a far bigger
effect than any prompt tuning I did.

---

## A cost defect

A prompt-heavy workload was running at a **162:1 prompt-to-completion ratio with a 5.8% cache
hit rate.**

Which is to say: the same large context was being re-sent almost every call and almost never
being reused. The fix is structural (stable prefixes, routing that respects cache affinity),
not a prompt edit. Replica affinity and prefix caching pull against each other, and if you
route for availability without thinking about it you will destroy your own cache hit rate — I
did.

---

## Protocol compliance

The best self-hosted seat emitted a conformant callback **82%** of the time.

Which is the whole argument for the wrapper. An 18% malformed-output rate is unusable as a
control-flow signal — but it is completely fine if the agent calls a **wrapper** that
constructs the record, instead of being asked to type the protocol correctly. Move the
formatting burden off the model and an 82% compliant model becomes a 100% compliant subsystem.

**General form of the lesson:** don't ask a model to be reliable at something a deterministic
program can do. Ask it for the judgment and let code handle the form.

---

## What I'd tell someone starting

1. **Measure true-negative rate separately.** Aggregate accuracy hides a reviewer that
   approves everything, and that is the single most expensive failure in the stack.
2. **Audit your answer key.** Mine was wrong in two places and it was penalising the correct
   models.
3. **Plot pass-rate by attempt before you build retry logic.** If it's flat, your retry logic
   is a billing mechanism.
4. **Measure quality against context length on your own tasks.** It will change your slice size
   and it will be a bigger lever than your prompts.
5. **Separate the roles and benchmark them separately.** Planning, implementing and validating
   are different jobs and models are not uniformly good at them.
6. **Put the protocol in a wrapper.** Then model compliance stops being load-bearing.
