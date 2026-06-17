# 0004. Commit the Ansible inventory instead of gitignoring it

Date: 2026-06-17

## Status

Accepted

## Context

The Ansible inventory file (`hosts.yml`) had historically been gitignored, on the general instinct that inventory files often carry sensitive details. Only an `.example` template was committed.

Once the CI pipeline started running Ansible from an ephemeral, self-hosted runner that checks out a fresh copy of the repository on every job, this became a real blocker: there was no mechanism to deliver a gitignored file onto the runner's workspace before each run, short of treating the entire inventory as a secret.

Inspecting the actual file's contents settled the question: it contains private-LAN IP addresses and hostnames, and a *reference* to the path of an SSH private key (not the key material itself). None of that is sensitive on its own — the IPs aren't reachable from outside the local network, and the key path is meaningless without the key.

## Decision

Remove the inventory file from `.gitignore` and commit it directly.

## Consequences

- CI jobs work against a fresh checkout with no extra delivery step for the inventory.
- Anyone with read access to the repository can see the internal network's host layout. Judged acceptable, since the repository is already private and the addresses are not internet-routable.
- If a host is ever added with a genuinely sensitive inventory variable, that variable needs to be excluded on its own (e.g. moved to a CI secret or an Ansible Vault file) — committing the inventory is not a blanket policy that all future inventory content is automatically safe to commit.
