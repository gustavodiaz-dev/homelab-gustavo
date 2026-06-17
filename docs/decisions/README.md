# Architecture Decision Records

Short records of the non-obvious technical decisions made while building this homelab, in [Michael Nygard's ADR format](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions). Written after the fact, close to when each decision was made, so the reasoning doesn't get lost.

| # | Decision | Status |
|---|----------|--------|
| [0001](0001-remote-state-in-s3-without-dynamodb.md) | Remote Terraform state in S3 without DynamoDB locking | Accepted |
| [0002](0002-manual-dispatch-as-the-approval-gate.md) | Manual workflow dispatch as the production-apply gate | Accepted |
| [0003](0003-exclude-self-managing-playbook-from-ci.md) | Exclude the CI runner's own Ansible playbook from CI | Accepted |
| [0004](0004-commit-the-ansible-inventory.md) | Commit the Ansible inventory instead of gitignoring it | Accepted |
| [0005](0005-secrets-as-transient-files-in-ci.md) | Materialize host secrets as transient files in CI, not as repo files | Accepted |
| [0006](0006-grafana-native-alerting-over-alertmanager.md) | Grafana native alerting instead of a separate Alertmanager | Accepted |
