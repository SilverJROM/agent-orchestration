# The protocol

An agent does not report completion in prose. Prose is unparseable, and the one time it
matters it will contain the word "blocked" inside a sentence explaining that nothing is
blocked. (That exact bug cost a run: a status line was read by substring-scanning, a `DONE`
note merely *containing* "BLOCKED" was parked, and the rollback destroyed finished work. The
fix was to read status from a regex capture group through one shared accessor, so the poll and
the read cannot drift apart.)

## Completion callback

```
[<engine> callback] <role> <slice> STATUS: DONE|BLOCKED
```

Three facts, fixed positions, one status token taken from a capture group — never from a
substring search over the line.

## Validation verdict

```
[VERDICT-V1] run= batch= attempt= marker= verdict=
             req= defect_class= failed_gate= commit= evidence=<path:sha256,…>
```

Properties that matter more than the shape:

- **`failed_gate=` is authoritative.** An earlier `gate=` field is accepted for old records and
  ignored. One authoritative field beats two plausible ones.
- **Evidence is hashed by the wrapper, not by the agent.** An agent that computes its own
  evidence hashes can compute convenient ones.
- **Defect class is a closed vocabulary.** Free-text defect descriptions cannot be aggregated,
  and un-aggregatable failure data is failure data you will never act on.

## Refusals are part of the contract

The submission wrapper rejects contradictory verdicts rather than storing them:

| Rejected | Why |
|---|---|
| `PASS` with a defect class | A pass with a defect is two different claims |
| `PASS` with a failed gate | Same |
| `FAIL` with no failed gate | An unattributed failure is not actionable |
| `FAIL` with a *list* of gates | Which one? Vagueness here propagates forever |
| Missing or empty prescription | A failure with no prescribed next step is a complaint |
| Missing evidence | Unevidenced verdicts are opinions |

This is the highest-leverage code in the system. Every refusal is a class of ambiguity that
cannot enter the record, and therefore a class of wrong report that cannot be generated later.

## Role cards

Three model-facing roles — **implementer**, **validator**, **prepare** — each with a role card
that is a versioned artifact, not an ad-hoc prompt.

Two spend profiles:

- **`local`** — tokens are effectively free (self-hosted), so the card carries full guidance:
  worked examples, defect-class glosses, failure-mode warnings.
- **`lean`** — tokens are metered, so the card carries the protocol only.

**The invariant that makes this safe:** the protocol core must be *identical* between profiles,
and that is enforced by reconstruction — strip the guidance supplement from the verbose render
and it must equal the terse render byte for byte. The lint refuses to ship a card that fails.

That invariant is the whole point. Two prompt variants maintained by hand drift, and the drift
is silent until a validator in one profile is enforcing a subtly different contract than the
same validator in the other. Names are protocol; glosses are guidance; they live on opposite
sides of the line.

One role inverts a rule: **`prepare` must never emit BLOCKED.** A preparation step that can
block gives every downstream agent a legitimate-looking way to decline work.

## Gates

Work advances only through an explicit gate. A gate that can be satisfied by assertion is not
a gate.

- **G0 — transport conformance.** Before any model work: can the engine reach the seat, and
  does the handshake conform? Most "the model is bad" reports are transport failures.
- **G1–G4** — per-slice correctness, evidence completeness, and requirement coverage.

Two practices keep gates honest:

**Preflight, fail closed.** Missing wrappers, unresolvable seats, and absent binaries are
refused *before* dispatch with a named edge, not discovered halfway through a run.

**Tests must be proven able to fail.** For each control, the mechanism is deliberately weakened
and the suite must go red. A green suite only means something if you have shown it can still
see the bug. This is also how you catch a test that string-matches help output instead of
checking behaviour — a real defect in my own suite, found exactly this way.
