# ADR 0004: Every record carries a hardware profile, unmeasurable values are null, and profiles are never compared

- Status: accepted
- Date: 2026-09-22

## Context

The primary machine is Apple silicon with unified memory. Other hardware is used when it is available
at no cost, which means irregularly and without guarantee.

Unified memory has no separate accelerator memory, host memory and transfer path. Several quantities
central to the research agenda, including KV cache size and bytes moved between tiers, either do not
exist there or are not exposed by the available engines.

The tempting failure is to estimate those quantities so the tables look complete. The result would be
numbers that look measured and are not, which is precisely the failure this project claims to be
correcting in others.

## Decision

Three rules, enforced in code.

1. Every record carries a `HardwareProfile`, including a `measurable` list stating which `Outcome`
   fields that profile can actually fill.
2. A field that is not measurable on the current profile is recorded as null. Null never means zero
   and is never replaced with an estimate. If an estimate is genuinely wanted, it is a separate,
   differently named field that is clearly marked as derived.
3. The ledger refuses to compare records from different hardware profiles, and states the two
   profiles when it refuses.

## Consequences

Positive:

- Every published number is true on the machine that produced it.
- The limits of the primary hardware are visible in the data rather than hidden in a footnote.
- When a CUDA machine becomes available, its results form a separate profile and do not silently
  contaminate the existing ones.

Negative:

- Result tables will have visible holes, and phase 2 will have a section stating what could not be
  demonstrated on the primary hardware. That is the correct outcome and the README says so up front.
- Cross hardware conclusions need explicitly designed paired experiments rather than casual
  comparison. That work is real and is not free.
