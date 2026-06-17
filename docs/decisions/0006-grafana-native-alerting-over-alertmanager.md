# 0006. Grafana native alerting instead of a separate Alertmanager

Date: 2026-06-17

## Status

Accepted

## Context

The observability stack (Prometheus, Grafana, Loki) collected metrics and rendered dashboards, but nothing proactively notified about problems based on those metrics — the homelab's existing notifications (uptime checks, backup job results) were all black-box, endpoint-reachability checks, not whitebox, metrics-threshold alerts.

The traditional Prometheus-ecosystem answer is Alertmanager: a separate component that receives alerts fired by Prometheus rules and routes them to notification channels. Every other notification in this homelab, however, already flows through the same pattern: an automation tool receives a webhook and forwards a formatted message to a Telegram bot. Deploying Alertmanager would mean a second, parallel notification pipeline with its own routing and templating config, duplicating a problem already solved.

Since Grafana 8, Grafana ships its own unified alerting engine that can evaluate Prometheus (or any datasource's) queries directly and fire to arbitrary contact points — including a plain webhook — without needing Alertmanager as a separate hop.

## Decision

Use Grafana's built-in alerting, with a webhook contact point pointing at a small automation workflow that formats and forwards the alert to Telegram — the same notification path already used elsewhere in the homelab.

Three rules cover the highest-value, easy-to-miss conditions to start: a host or service being unreachable for a sustained period, low free disk space, and high memory usage.

## Consequences

- One fewer long-running component to operate (no separate Alertmanager deployment, config reload, or routing tree to maintain).
- All homelab notifications — uptime, backups, and now metric-based alerts — converge on a single delivery mechanism, which makes the overall notification surface easier to reason about and to extend.
- Some Alertmanager-specific features (e.g. its more advanced silencing/inhibition rules across many alert sources) aren't available; not currently needed at this scale, and Alertmanager remains an option to add later if Grafana's alerting becomes a limiting factor.
- Alert rules live in Helm values alongside the rest of the Grafana configuration, version-controlled and reviewable the same way as dashboards and datasources.
