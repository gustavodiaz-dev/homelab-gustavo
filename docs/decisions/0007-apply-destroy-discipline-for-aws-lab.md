# 0007. Apply/destroy discipline for the AWS lab environment, no always-on infrastructure

Date: 2026-06-17

## Status

Accepted

## Context

Phase 7 introduces the first real AWS infrastructure (VPC + EC2 + Budget) to support AWS CLF-C02/SAA-C03 study, beyond the existing S3 state backend. Unlike Proxmox, where the homelab hardware is already paid for and running 24/7 regardless of what's deployed on it, every hour an EC2 instance runs in AWS draws against a finite, time-boxed resource: the account started today under AWS's $100-200 credit plan, expiring 2026-12-17 or whenever the credit runs out, whichever comes first. There are also other fixed monthly costs already committed (Claude subscription, API usage) that leave little room to absorb an unplanned new recurring expense.

The certification roadmap (CLF → SAA → CKA) needs this AWS environment to last for months, not be exhausted in the first few sessions.

## Decision

The `aws-lab` Terraform environment is never left running. The workflow is: `terraform apply` at the start of a study session, `terraform destroy` at the end of it — every time, no exceptions. It is deliberately kept out of the `terraform.yml` CI pipeline (which does support automatic `apply` via `workflow_dispatch` for the `homelab` environment), since that pipeline assumes the underlying infrastructure costs nothing extra to leave running, which is not true here.

An `aws_budgets_budget` resource (alerting at 80% and 100% of a configurable monthly limit, default $20) acts as a safety net for whatever discipline misses.

## Consequences

- Every study session has a small bit of friction (re-running `apply`, waiting for the instance to come up) instead of an always-available box — acceptable trade-off given the cost asymmetry.
- State and configuration are reused between sessions (same VPC CIDR, same key pair import), so `apply` after a `destroy` reliably recreates the same environment rather than a slightly different one each time.
- If a session is ever interrupted before `destroy` runs, the instance keeps accruing cost until the next session is opened and closed properly, or until the Budget alert fires — there is no automatic shutoff. This is a known gap, acceptable for a single-operator lab, revisited only if it causes an actual surprise.
- As SAA-level topics introduce resources with real per-hour cost regardless of usage (NAT Gateway, ALB, RDS Multi-AZ), the same discipline applies to those — they get added to `aws-lab` only for the specific session that needs them, never as permanent fixtures.
