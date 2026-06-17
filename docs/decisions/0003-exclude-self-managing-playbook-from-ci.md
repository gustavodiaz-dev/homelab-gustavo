# 0003. Exclude the CI runner's own Ansible playbook from CI

Date: 2026-06-17

## Status

Accepted

## Context

The homelab has one Ansible playbook per managed host, including a playbook that installs and configures the self-hosted GitHub Actions runner itself (package install, systemd unit, service restart).

Early in setting up the Ansible CI pipeline, this playbook was included like any other in the automated `--check` matrix and in the manual-deploy options. Two problems surfaced:

1. **Structural risk**: this playbook can restart the very runner process executing it. Run from CI, in the worst case a job restarts its own runner mid-execution, leaving the job (and the runner's registration) in an inconsistent state.
2. It also depends on a host-specific secrets file that, by design, is never checked into the repository — so a fresh CI checkout can't find it, independent of the structural problem above.

The second issue (missing secrets file) is a recurring pattern also seen with another host's playbook, and was solved there by loading the secret from a CI secret store instead. That fix doesn't address the first issue here.

## Decision

Permanently exclude the runner's own playbook from both the automated check job and the manual-deploy options in CI. It continues to be run by hand, directly from a workstation, as it already was before any CI existed.

## Consequences

- One playbook (out of several) lives outside the otherwise-automated pipeline — a documented, deliberate exception rather than an oversight.
- Removes any risk of a CI job interrupting its own execution environment.
- If the runner's configuration ever needs to change, that change still requires a manual step — acceptable, since runner configuration changes are rare compared to changes to the services it manages.
