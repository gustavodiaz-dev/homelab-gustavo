# 0005. Materialize host secrets as transient files in CI, not as repo files

Date: 2026-06-17

## Status

Accepted

## Context

Several Ansible playbooks need files that contain real secrets at apply time: an SSH private key for connecting to managed hosts, and a per-service `.env` file containing application credentials (database passwords, encryption keys, third-party API credentials). These files are intentionally never committed to the repository — some never even exist outside the host they belong to.

Running these playbooks from a fresh CI checkout means the files Ansible expects to find on disk (e.g. `files/<host>/.env`) simply aren't there, since they were never part of the repository in the first place.

## Decision

Store each secret's full content as a GitHub Actions repository secret. At the start of any CI job that needs it, write the secret's value to the exact path the playbook expects, with restrictive file permissions; at the end of the job (including on failure), delete it.

This mirrors how the SSH private key is already handled, applied to other per-host secret files as the same need arose.

## Consequences

- Playbooks don't need to know or care whether they're running from a workstation (where the real file already exists) or from CI (where it's materialized just-in-time) — no playbook logic changes.
- The secret only exists on the runner's disk for the duration of a single job step, not persisted between runs (the runner is also ephemeral and wipes its workspace regardless).
- Anyone who can read repository secrets in the GitHub UI (effectively, the repository owner) can read the full plaintext of these files — acceptable for a single-operator setup, but this pattern doesn't scale cleanly to a team without a proper secrets manager.
- Each new secret file follows the same two-step pattern (write before, clean up after with `if: always()`), keeping the workflow files consistent as more hosts are added.
