# Operational Observability Dashboards

## Purpose

This guide defines a platform-neutral approach to operational dashboards used to classify problems, choose the next investigation, and support bounded operational decisions. It applies to virtual machines, services, databases, queues, network components, and other observable systems.

A dashboard is a publication and exploration surface. It is not the observed system, a system of record, evidence by itself, an approval, or an automated decision-maker.

The primary reader is an SRE or platform engineer. A separate owner-facing publication may use the same evidence but should expose only the information and actions appropriate to that reader.

## Scope and non-goals

The guide covers dashboard reasoning: how to select an entity, express a decision question, construct readings, compare time windows, preserve evidence boundaries, and route a user from a collection-level view to a specific investigation.

It does not prescribe one observability product, one query language, universal thresholds, a complete service-management process, alert lifecycle design, an automatic rightsizing policy, or a universal definition of healthy. Grafana, PromQL, MetricsQL, recording rules, inventory integration, and dashboards-as-code are implementation choices.

## Decision first

Start with the operational decision or problem class, not with the available metric list. Examples include:

- Is a service breaching its user-facing objective?
- Which entities require investigation first?
- Is a virtual machine a candidate for a capacity review?
- Is the observed condition likely a resource constraint, an application fault, or an observation-path failure?

The decision question declares what a reading may support and what it cannot establish. A low CPU reading can support a compute-rightsizing review; it cannot by itself prove that a service is unimportant, safe to decommission, or safe to resize.

## Bounded context

Every dashboard operates in a named semantic frame. The context must define:

- The intended reader and their first action
- The entity type and recognition rule
- The decision or investigation use
- The local meaning of terms such as healthy, saturation, candidate, pressure, breach, and available
- The relevant time window, data source, and freshness expectations
- The non-use boundary and escalation route

A useful context name is specific, for example `OperationalObservability.ServiceInvestigation` or `VirtualMachine.ComputeRightsizing`.

## Ontology

### Entity and scale

Choose the Entity of Concern before aggregation:

| Scale | Typical concern | Example question |
|---|---|---|
| Fleet or collection | Prioritisation and outlier detection | Which instances deserve review? |
| Entity | Diagnosis or bounded decision | Can this VM be resized? |
| Resource part | Local mechanism | Which vCPU, filesystem, process, or interface is constrained? |
| Service outcome | Consumer-visible behaviour | Are users receiving the promised result? |

A collection is not automatically an acting system. A fleet map is a view over a selected collection; it does not make the collection the owner of a decision.

### Carriers, selectors, readings, and views

| Object | Meaning | Example |
|---|---|---|
| Carrier | A source metric, event, trace, log, inventory record, or label-bearing series | `node_cpu_seconds_total` |
| Selector | A scope control applied to a query or view | project, job, hostname, instance |
| Reading | A derived, unit-bearing interpretation of one or more carriers | CPU p95 %, error rate, available RAM % |
| Claim | A bounded statement supported by readings | `resize-review candidate` |
| View | A panel, table, report, or dashboard that publishes readings | Fleet table, VM detail panel |
| Decision or work | A human or automated action performed under a separate process | approve and execute resize |

A technical label such as `instance` is a retrieval identity, not the observed system itself. A dashboard view is not the claim, and the claim is not the performed work.

## Outcome and mechanism signals

Use the decision context to decide whether outcome or mechanism signals lead the view.

For service-reliability decisions, lead with user- or consumer-relevant outcomes: availability, latency, error rate, correctness, freshness, or another declared service objective. CPU, memory, I/O, retries, and queue depth are mechanism signals that help explain likely causes.

For resource-capacity decisions, allocated resource and consumption can lead the view. CPU and memory readings are primary evidence for compute rightsizing, while service outcomes remain a guardrail against harmful change.

SLO-first therefore means outcome-or-decision first, not that every dashboard must display the same availability graph.

## Reading construction

A reading definition includes:

- Carrier identity and label scope
- Aggregation and grouping rule
- Unit and normalization
- Range or rate window
- Missing-data behaviour
- Time window and comparison baseline
- Interpretation and non-interpretation boundary

Preserve units visibly. A percentage, a byte count, an event rate, an absolute resource equivalent, and a percentile are different readings even when they share a label such as `utilization`.

For monotonic counters, use a short-window `rate` for a readable diagnostic trend and an `increase` over the declared decision window for a cumulative decision reading. `irate` is a local high-resolution diagnostic view based on the newest two samples; do not use it as the base for long-window percentiles, fleet ranking, or a rightsizing claim.

## Time and comparison

Time is part of a reading's meaning. Use a declared observation period and, where useful, compare:

- Current value for immediate orientation
- Trend for shape, drift, and burst detection
- p50 for typical behaviour
- p95 for high-but-normal behaviour
- Time above a threshold for duration of pressure
- Business-hours and full-time views
- Current period and prior baseline

A single average can hide short saturation periods; a single maximum can overstate a rare event. Percentiles and time-above-threshold should be interpreted only with their range, sampling, and aggregation semantics stated.

A range selector such as `[30d]` defines the history used by a query function. It does not by itself set the x-axis range of a time-series panel. Use an Instant/Stat or Table presentation for a single rolling 30-day result; use a time-series view when the intended result is the evolution of a reading.

## Fleet-to-entity navigation

A collection-level view followed by a per-entity view is not an anti-pattern. It is appropriate when the collection is used for screening and prioritisation, while a specific entity is the subject of an investigation or capacity claim.

Use a deliberate hierarchy:

1. **Fleet overview:** counts by attention state, data-quality gaps, distribution and Top-N outliers.
2. **Fleet table:** one row per recognised entity, with comparable decision readings and a clearly bounded status.
3. **Entity detail:** trends, percentiles, source return, and rival-explanation checks for one selected entity.
4. **Review or change view:** the separate work path for a proposed action, owner constraints, approval, rollback, and recorded outcome.

Do not draw every entity as a line in one shared chart when the fleet is large. Use a ranked table, Top-N bar chart, histogram, heatmap, or a repeated panel only after filtering to a small set. A repeated panel is an implementation convenience, not the primary fleet reasoning surface.

A fleet table should carry a human-recognisable entity name and retain technical selectors only for reproducible retrieval and drill-down. A row link may pass `project`, `job`, and `instance` into an entity-detail view, but that link does not establish that the selected technical target is the business or operational owner of the entity.

## Platform/SRE and owner views

One carrier set can support several views without making the views interchangeable:

| Viewpoint | Primary concern | Suitable content | Not a substitute for |
|---|---|---|---|
| Platform/SRE | Evidence quality, saturation, data gaps, investigation | Technical identity, trends, counter semantics, diagnostic signals, drill-down | Application ownership or approval |
| Service/application owner | Capacity implication and planned action | Plain entity name, resource profile, suggested review, change impact | Root-cause analysis or automatic permission |
| Change/review authority | Bounded approval of proposed work | Evidence package, constraints, risk, rollback and outcome | The observability dashboard alone |

A simplified owner-facing view is a controlled coarsening of an SRE view: it should retain the entity, declared period, source return, admissible use, and the condition under which the user must reopen the technical detail.

## Example: Linux VM capacity review

This section is illustrative, not a universal capacity policy. It assumes Linux virtual machines export node-level metrics, a `project` selector bounds the collection, and `job` and `instance` identify observation series. Local labels, inventory joins, scheduling patterns, application constraints, and resource policies may differ.

### Purpose and entity

The bounded question is: **which Linux VMs deserve a human capacity review, and what evidence is required before proposing a CPU or RAM reduction?** The fleet view identifies candidates; an individual VM is the Entity of Concern for a concrete recommendation.

A 30-day full-time period is a useful default control window because it can include non-working-hour and periodic workloads. A business-hours view may explain interactive demand but must not replace the full-time safety baseline. If known workload cycles exceed 30 days, the evidence window must be extended or the claim restricted.

### Fleet overview

The default `node` selector should be All, while `project` scopes the fleet. The overview should prioritise action rather than calculate a misleading single fleet average:

- Number of recognised VMs and number with adequate data coverage
- Number with OOM events in the control window
- Number requiring manual review because of incomplete evidence or pressure signals
- Ranked Top-N CPU p95, lowest RAM available p05, and highest major-fault totals
- A fleet table, sorted by a declared reading or status, with a data link to VM detail

Counts and rankings are screening readings. They do not establish a rightsizing action.

### Fleet table

A platform/SRE table can contain the following columns. Values should retain units and the observation period in the panel title or field name.

| Column | Reading use | Interpretation boundary |
|---|---|---|
| VM name and technical identity | Recognition and drill-down | A scrape label alone is not inventory identity |
| Allocated vCPU / allocated RAM | Capacity context | Allocation is not consumption |
| CPU p50 and p95 % | Typical and high-normal CPU utilisation | Does not show application outcomes |
| CPU time above local threshold | Duration of elevated CPU demand | Threshold is local policy, not universal truth |
| RAM used p95 % | High-normal memory consumption | Must use a declared `used` definition |
| RAM available p05, % and GiB | Low-normal headroom | Stronger than `MemFree`, but not a guarantee |
| Major page faults, 30d | Investigation signal | May reflect storage or workload behaviour, not RAM shortage alone |
| OOM kills, 30d | Strong stop/review signal | Requires investigation; does not prove one cause |
| Data coverage | Confidence boundary | Missing series is observation-path evidence, not healthy state |
| Status | Prioritisation claim | Never an automatic resize approval |

Suggested provisional states are `observe`, `manual review`, `resize review`, and `resize recommended`. Their entry rules must be local, versioned, and reviewed against actual changes and outcomes. Avoid collapsing these readings into one opaque utilisation score.

### VM detail

The first visual panel should make the resource envelope legible. For memory, show `Total RAM`, `Used RAM`, and `Available RAM` on one non-stacked time-series chart, in consistent fixed colours and bytes/GiB. `Total = Used + Available`; stacking all three would double-count the total and create a false visual capacity boundary.

A useful initial detail layout is:

- CPU utilisation trend and consumed vCPU-equivalent
- Memory capacity and use: Total, Used, Available
- Fixed 30-day decision readings: CPU p50/p95, RAM used p95, RAM available p05, OOM total, major-fault total
- Diagnostic row: major-fault rate and relevant source or inventory links

Use human-readable legend names such as `Total RAM`, `Used RAM`, `Available RAM`, and `Major faults/s`. Fixed colour assignments preserve meaning when labels, query order, or returned series change.

### CPU readings

`node_cpu_seconds_total` is a counter. Convert idle time to a CPU-utilisation gauge with a short-window `rate` before computing p50 or p95 over the decision period. `irate` is useful for an immediate troubleshooting lens but is too sensitive to the newest sample pair for a 30-day candidate claim.

CPU utilisation percentage and aggregate vCPU-equivalent are different readings. The first is normalised by observed vCPU capacity; the second expresses consumed work across vCPUs. Do not label aggregate vCPU-equivalent as a VM percentage.

### Memory readings

For Linux, use `MemAvailable` rather than `MemFree` as the primary headroom reading. `MemAvailable` estimates memory available for new allocations without swapping and accounts for reclaimable memory; it supports a more useful rightsizing reading than raw free pages.

A minimum set is:

- Trend: `MemTotal`, `MemAvailable`, and `MemTotal - MemAvailable`
- Decision readings: RAM used p50/p95, RAM available p05, and observed minimum available
- Safety readings: OOM-kill total over the control window and major-page-fault total over the control window
- Investigation trend: short-window major-page-fault rate

A page fault is not automatically a fault condition. Minor page faults normally occur when a page is already memory-resident and only a mapping is required. Major faults require obtaining an absent page, often from storage, and are therefore a stronger diagnostic signal; even major faults are not proof that RAM is undersized.

### Data coverage and duplicate series

Before using any 30-day claim, verify carrier coverage and uniqueness. Query a metric separately to inspect the full label set, then identify whether multiple returned series represent multiple VMs, multiple scrape paths, or genuinely different dimensions.

Do not remove unexpected multiple lines with `sum` or `avg` before establishing their meaning. Choose a canonical observation path or an explicit aggregation rule. An empty result, zero value, stale sample, and changed label set are distinct observation outcomes.

### Non-use boundary

The example can identify a review candidate. It cannot, without additional evidence and work, prove that a VM is low importance, safe to decommission, free of scheduled workload risk, safe to shrink storage, or safe to resize without owner, topology, licensing, application, rollback, and change controls.

## Drill-down design

A useful dashboard supports an explicit investigation route:

1. Start with a fleet or service summary to locate outliers.
2. Select one entity using stable identity and context selectors.
3. Inspect trend, percentile, and time-window readings.
4. Open resource or subsystem detail only when the decision question requires it.
5. Return to source carriers, inventory, runbooks, traces, or owners when a stronger claim is needed.

The panel layout should expose this route rather than merely display all available metrics.

## Evidence boundaries

A metric reading is an evidence carrier for a claim only within a declared scope, method, and relevance window. A screenshot or a dashboard tile can support orientation but cannot on its own prove safety, approval, root cause, outage, or completed work.

For a reliance-bearing claim, retain enough provenance to answer: which carriers, source systems, query method, time window, entity scope, and rival explanations were considered? Refresh the claim when the data source, labels, instrumentation, workload, or operating context changes.

## Design rationale

The design follows several complementary, externally documented practices:

- Grafana recommends a logical dashboard narrative, hierarchical dashboards and drill-downs, purposeful grouping, and a reduction in cognitive load rather than one unstructured surface.
- Google SRE treats monitoring as a means to understand long-term resource trends, diagnose systems, and compare behaviour before and after change; it distinguishes observation from action.
- Fleet-health designs use collection status, per-entity tables, prioritised Top-N views, and drill-down rather than a single average or a chart containing every entity.

These are design patterns, not a substitute for local semantics, capacity policy, or an assurance case. Reassess the dashboard after real review and change outcomes.

## General failure modes

- **Metric-first construction:** panels are chosen because data exists, not because they support a decision.
- **Average-only reasoning:** typical load masks regular peaks, contention, or burst behaviour.
- **Label-as-identity:** a scrape target is treated as the system without an inventory or recognition rule.
- **Dashboard-as-proof:** a visualisation is treated as assurance, approval, or completed work.
- **Outcome-mechanism collapse:** low-level resource metrics are mistaken for user experience, or vice versa.
- **Scope-free threshold:** one value is applied across workloads without a local context, window, or rationale.
- **Missing-data overread:** absence of a series is treated as proof of an object or service failure.
- **Fleet-average comfort:** an aggregate hides an entity that needs immediate investigation.
- **Unbounded repetition:** a panel per entity creates visual overload and backend cost instead of a fleet screening view.
- **Counter misuse:** an instantaneous counter rate is treated as a 30-day decision reading, or a rolling cumulative result is displayed as a misleading trend.
- **Double-counted capacity:** Total, Used, and Available are stacked despite Total already containing the other two.

## Related DPFs

- [VM Compute Rightsizing](dpf-vm-compute-rightsizing.md) applies this guide to CPU and RAM capacity decisions for virtual machines.
- [Alerting and Observability Assurance](dpf-alerting-and-observability-assurance.md) applies the evidence and outcome principles to alert conditions, missing data, and validation.

## Implementation note

Grafana variables, panel options, annotations, query expressions, dashboard JSON, data links, and colour overrides are implementation details. Keep implementation examples in a separate appendix or repository directory so the decision model remains portable across tools.
