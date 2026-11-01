# Testing and eval boundary

Status: agreed ownership; the execution contract below is proposed.

## Agreed boundary

- **TEST-1.** This specification owns test syntax, visibility, target selection,
  argument checking, comparison meaning, and error categories.
- **TEST-2.** Core owns the mandatory facilities used by eval, runtime
  conformance fixtures, and expected runtime observations.
- **TEST-3.** A compiler implements admission, lowering, and the test runner.
  A runtime adapter implements core's required capabilities for its target.
- **TEST-4.** The specifications contain illustrative examples; executable test
  source belongs outside this repository.

## Proposed first contract

| Area | Proposed behavior |
| --- | --- |
| Target | Eval names a module `fn`; wrap an operator in a function |
| Arguments | Ordinary positional/named binding; const parameters receive values, temporal parameters receive traces |
| Dense traces | Cell i occurs at start + i minimum engine steps; `_` means no publication |
| Comparison | Compare presence first, then exact values and equal lengths; ordinary operator candidates cannot alter harness comparison |
| Horizon | Preserve the longest input length and any later output ticks; trailing silence counts |
| Ownership | Each eval has a fresh run and recording; retained captures cannot change on later ticks |
| Empty result | An empty recording exists after successful recorder start; a missing recording or failed run is an error |
| Outputless target | Run it without an output recorder or a comparable output sequence |
| Failures | Separate source rejection, expected execution error, assertion failure, and infrastructure/cleanup failure |
| Dependencies | Eval uses core's designated replay/record implementation; optional libraries and local overloads cannot replace it |

Illustrative proposed case; this document is not executable test source:

```hgl
test repeated_publication {
    assert eval(pass_through_i64, value: [7, _, 7, 9]) == [7, _, 7, 9]
}
```

Core must supply a compute fixture that publishes its own output for this case.
A composition identity alone does not demonstrate compute publication.

## Open

- Accept the table as a bounded scalar contract, or amend individual rows.
- Fix the initial scalar types, run origin, and minimum engine step.
- Define failure codes, cleanup outcomes, and reporting requirements.
- Define timed traces and source-only eval with an explicit end bound.
- Decide test metadata, structured reports, and attributed/nested test scopes.
- Define the host interface for seeded state and observations unavailable in HGL.

Timed traces, structural deltas, and richer test syntax are outside the proposed
first exercise. Their exclusion makes no claim about their eventual design.
