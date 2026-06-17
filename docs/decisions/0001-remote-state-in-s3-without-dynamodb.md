# 0001. Remote Terraform state in S3 without DynamoDB locking

Date: 2026-06-17

## Status

Accepted

## Context

The CI/CD pipeline runs Terraform from a self-hosted, ephemeral GitHub Actions runner that lives inside the homelab. The runner wipes its workspace after every job, so a `terraform.tfstate` file living only on that runner's disk (or on a personal laptop) would not survive between runs — and worse, a CI `apply` starting from an empty state could try to recreate infrastructure that already exists.

State needed to live somewhere both the laptop and the CI runner could reach: an S3 backend.

The classic Terraform pattern for concurrent-write protection pairs an S3 backend with a DynamoDB table for locking. Terraform 1.10 added native S3 locking (`use_lockfile = true`), which uses a lock file object in the same bucket instead of a separate table.

## Decision

Use an S3 backend with native locking (`use_lockfile = true`) and no DynamoDB table.

A small separate Terraform configuration ("bootstrap"), with its own local state, provisions the bucket and a narrowly-scoped IAM user dedicated to CI. This avoids a chicken-and-egg problem: the backend that creates the bucket can't itself live in a bucket that doesn't exist yet.

## Consequences

- One fewer AWS resource to provision, pay for (trivially) and reason about.
- Locking is good enough for a single-operator homelab with low apply concurrency; a team environment with frequent concurrent applies might still prefer DynamoDB's more mature locking semantics.
- Native S3 locking requires Terraform ≥ 1.10 — pinned in the CI runner's toolchain.
- The bootstrap config's local state is itself a single point of manual care: it's small, changes rarely, and is treated as a one-time setup step rather than part of the regular apply cycle.
