# Security Policy

## Supported versions

This project is pre-alpha and has not made a release. No version is supported for production use.

## Reporting a vulnerability

Please do not open a public issue for a security problem.

Report it through GitHub's private vulnerability reporting on this repository. Include what you
found, how to reproduce it, and what you think the impact is. You will get an acknowledgement, and an
assessment once the report has been reviewed.

## Scope

When implementation exists, the areas most likely to matter are:

- The information store, which holds whatever corpus is loaded into it
- The ledger, which records prompts, answers and resource use, and is therefore a data retention
  question as much as a security one
- Executor adapters that carry credentials for remote models
