# Testing and eval boundary

Status: agreed ownership. Source behavior belongs to the language chapter;
its eval execution profile remains proposed.

## Agreed boundary

- **TEST-1.** This specification owns test syntax, visibility, target selection,
  argument checking, comparison meaning, and error categories.
- **TEST-2.** Core owns the mandatory facilities used by eval, runtime
  conformance fixtures, and expected runtime observations.
- **TEST-3.** A compiler implements admission, lowering, and the test runner.
  A runtime adapter implements core's required capabilities for its target.
- **TEST-4.** The specifications contain illustrative examples; executable test
  source belongs outside this repository.

## Contract ownership

[Tests and eval](language/tests-and-eval.md) owns syntax, target selection,
argument/trace typing, comparison, and expected-error behavior. The
[core eval contract](https://github.com/hhenson/hgl_core/blob/main/spec/testing-and-eval.md)
owns runtime facilities, recording, and shared observations. Keep each rule in
its owning document rather than repeating the contract here.

## Scope of the proposed exercise

Timed traces, structural deltas, and richer test syntax are outside the proposed
first exercise. Their exclusion makes no claim about their eventual design.
