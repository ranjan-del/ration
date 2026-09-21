# Contributing

Thank you for looking at this. Ration is a research project, so contributions are judged slightly
differently from a normal library, and the rules below are the difference.

## The rules that are not negotiable

These apply to the maintainer exactly as strictly as to anybody else.

1. **No number without a run.** Any number in code, documentation, a commit message or a pull request
   description must come from a run recorded in the ledger, and the record must be included. A number
   that the ledger cannot reproduce is treated as a defect, not a rounding issue.
2. **Every claim carries its context.** Hardware profile, model id and quantisation, seed, date, and
   the baseline it beat. A result missing any of these is not reviewable.
3. **Never compare across hardware profiles.** The ledger enforces this. Do not work around it.
4. **Unmeasurable is null.** Not zero, not an estimate. If you want an estimate, add a separate field
   with a name that says it is derived.
5. **Name the prior art.** Before describing anything as new, add or update its entry in
   `docs/landscape.md` with the closest existing systems and the specific gap.
6. **Negative results are results.** If your change makes something worse, say so in the pull request
   and keep the measurement. Several phases of this project have a negative result as a legitimate
   outcome.

## Getting set up

Requirements and setup instructions arrive with the phase 1 implementation. There is no code to run
yet.

## Style

- Conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`
- Do not use em dashes in documentation or commit messages
- New behaviour comes with tests, and tests pass with no API key and no network
- `ration-core` imports nothing from the other packages. This is enforced in continuous integration

## Design changes

Anything that changes a contract, a port or the evaluation method needs an ADR in `docs/adr/` before
the code. Anything that changes a task suite invalidates existing comparisons, and the pull request
has to say which results it invalidates.

## Licensing

Contributions are accepted under the Apache License 2.0. See [LICENSE](LICENSE).
