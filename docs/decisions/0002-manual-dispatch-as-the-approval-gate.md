# 0002. Manual workflow dispatch as the production-apply gate

Date: 2026-06-17

## Status

Accepted

## Context

The CI/CD design called for `plan`/`check` to run automatically on every pull request, but `apply`/`deploy` to require a deliberate approval step before touching real infrastructure — the standard "automatic plan, gated apply" pattern.

GitHub Actions Environments support a native "required reviewers" protection rule for exactly this. Attempting to configure it on this repository's environment returned an HTTP 422 from the GitHub API. The cause: **required reviewers on Environments is a paid-plan feature for private repositories** — it's unavailable on the Free plan unless the repository is public.

Three ways forward existed: upgrade to GitHub Pro (a few dollars a month), make the repository public, or replace the native approval gate with a different mechanism.

## Decision

Use `workflow_dispatch` (a manually-triggered workflow with no automatic trigger) as the gate for `apply`/`deploy`, instead of an Environment-protected job.

The effective gate becomes: only a repository collaborator with write access can click "Run workflow" — which, on a private repo, is already restricted to the owner. The `plan`/`check` jobs stay fully automatic on pull requests so review feedback is still immediate; only the mutating step requires a manual click.

## Consequences

- No recurring cost, no change to repository visibility.
- The gate is weaker than native required-reviewers in one respect: it doesn't enforce a second pair of eyes, since the same person who could approve is also the one merging. For a single-operator homelab this is an acceptable trade; it would not be for a team.
- The plan generated during the automatic PR run and the plan executed during the manual apply are *not* guaranteed to be identical, since they're separate workflow runs — unlike the artifact-passing pattern used when plan and apply share a single run. Mitigated by always reviewing the plan output printed during the manual run itself before trusting it.
