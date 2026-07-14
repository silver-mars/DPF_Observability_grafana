# DPF: Alerting and Observability Assurance

## Purpose

This DPF defines principles for alert conditions and their evidence boundaries. It helps teams distinguish a user-impacting service condition, an internal object condition, and a failure of the observation path.

It applies independently of alerting product or notification channel. Alert rules, routing, escalation, runbooks, and incident processes are implementations that must serve the declared condition and response.

## Scope and non-goals

This DPF covers outcome-oriented alert design, alert-expression validation, missing-data interpretation, lifecycle semantics, and evidence required for an actionable alert.

It does not prescribe one alert manager, one SLO formula, on-call organisation, severity taxonomy, incident process, or universal threshold.

## Alerting context

Use a named context that states the intended responder, protected outcome, affected entity, expected action, observation sources, and condition duration. A useful alert is an action invitation for a responsible system or person; it is not merely a visible graph crossing a value.

## Three observation layers

Keep these layers distinct:

| Layer | Question | Example evidence |
|---|---|---|
| Object state | What does the component report about itself? | process state, queue depth, resource use |
| Observation-path state | Can monitoring obtain that report? | scrape success, collector health, discovery state |
| Service outcome | Can a consumer receive the intended result? | synthetic check, error rate, availability, latency |

A missing internal metric can justify an observability alert. It does not by itself prove a service outage. A service-outcome alert can indicate user impact even when internal telemetry is healthy or incomplete.

## Outcome and SLO orientation

For user-impacting services, begin with the promised outcome and its acceptance measure: availability, error rate, latency, freshness, correctness, or another service-level objective. Mechanism alerts remain useful when they reliably predict or prevent outcome failure and have a clear response path.

Do not turn SLO into a decorative dashboard metric. Its measurement population, scope, exclusions, time window, error-budget treatment, and owner must be explicit before it is used for alerting or escalation.

## Alert condition semantics

An alert rule has at least two distinct semantics:

- **Expression (`expr`):** whether the condition evaluates as true at one evaluation instant.
- **Sustained duration (`for` or equivalent):** whether the true condition persists long enough to fire.

An expression is not an alert lifecycle. A rule can also involve evaluation interval, pending/firing/resolved state, grouping, routing, inhibition, mute windows, deduplication, and notification delivery. Keep these concerns separate in design and validation.

## Expression validation

Validate an alert expression before relying on it:

1. Run the exact expression over a historical interval containing both normal and suspected failure conditions.
2. Use a resolution no coarser than the rule evaluation interval.
3. Confirm the returned series has the intended labels, units, cardinality, and truth semantics.
4. Check whether the condition remains true for at least the configured sustained duration.
5. Test absent, zero, stale, reset, and label-change cases separately.
6. Confirm the alert carries enough context for the next action or drill-down.

Historical query validation tests the expression against observed data. It does not fully test routing, notification, lifecycle state, ownership, or responder action.

## Missing data

An empty result, a zero-valued result, a stale sample, and a changed label set are not equivalent. Missing data may arise from a target outage, exporter failure, scrape failure, network partition, credentials, relabeling, service discovery, query scope, or retention behaviour.

Design missing-data alerts as observation-path claims unless independent evidence establishes the service outcome. Where possible, combine independent sources or provide an explicit drill-down from the observation-path alert to service checks and collector diagnostics.

## Evidence and actionability

An alert should make the following recoverable:

- The condition and affected entity
- The outcome or risk it protects
- The evidence carriers and query semantics
- The relevance window and freshness assumptions
- The responsible responder or routing policy
- The intended first action and runbook link
- The principal rival explanation
- The reopen or suppression conditions

A dashboard badge, alert state display, or generated summary is a publication of alert-related information. It is not approval, proof of impact, or completed incident work by itself.

## Failure modes

- **Technical-only alerting:** alerts on every measurable mechanism without a user impact or action path.
- **Outcome-only blindness:** no mechanism signals to provide early warning or diagnosis.
- **Missing equals outage:** an observation gap is treated as conclusive service failure.
- **Expression equals lifecycle:** a true query is assumed to mean a fired, delivered, and owned alert.
- **Zero equals absent:** numerical zero and no series are conflated.
- **Noisy threshold:** a condition repeatedly fires without meaningful action or duration filtering.
- **Unowned alert:** no responder, runbook, or escalation is attached.
- **Dashboard-as-assurance:** a green or red tile is used as proof without source, time, and condition semantics.

## Relationship to the general guide

This DPF applies the evidence, outcome/mechanism, and time-window principles of [Operational Observability Dashboards](guide-operational-observability-dashboards.md). Implementation-specific rules, query language examples, routing configuration, and runbooks should live in an implementation appendix or operational repository.
