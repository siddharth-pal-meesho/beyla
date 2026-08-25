<!-- m-wiki: type=concept slug=survey-info-metrics topic=telemetry base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: concept. 0 sources.

# Survey info metrics

An info-style metric reporting which processes Beyla *discovered and matched*, independent of whether it instrumented them. It converts the hardest support question — "is Beyla broken, or did it never match my service?" — into something answerable with a query.

## Where it applies in this repo

`pkg/export/otel/metrics_survey.go:SurveyMetricsReporter` holds the state; `pkg/export/otel/metrics_survey.go:SurveyInfoMetrics` is the pipeline-facing constructor, with `pkg/export/otel/metrics_survey.go:newSurveyMetricsReporter` doing the actual build.

The reporter is event-driven rather than poll-driven. `pkg/export/otel/metrics_survey.go:SurveyMetricsReporter.watchForProcessEvents` consumes the decorated process-event stream, and `pkg/export/otel/metrics_survey.go:SurveyMetricsReporter.onProcessEvent` dispatches each event to either `createSurveyInfo` or `deleteSurveyInfo`.

Because a single service can own many PIDs, the reporter maintains a PID↔service association explicitly: `pkg/export/otel/metrics_survey.go:SurveyMetricsReporter.setupPIDToServiceRelationship` on creation and `pkg/export/otel/metrics_survey.go:SurveyMetricsReporter.disassociatePIDFromService` on termination, the latter reporting whether that was the service's *last* PID so the info metric is only withdrawn when the service genuinely disappears.

Attributes come from `pkg/export/otel/metrics_survey.go:SurveyMetricsReporter.attrsFromService`. A Prometheus counterpart lives at `pkg/export/prom/prom_survey.go`.

The discovery side that feeds this is described in [Process discovery and survey mode](../pipeline/process-discovery-and-survey.md).

## Why this design

An info metric — value 1 with descriptive labels, present while the thing exists — is the right shape here because the question is existential, not quantitative. You are asking whether a service is known, and the label set tells you *how* Beyla identified it, which is exactly what you need to debug a selection-criteria mistake.

The PID-to-service bookkeeping is what makes the lifecycle correct. A naive implementation keyed on PID would withdraw the metric when any worker process exited; keying on service UID and refcounting PIDs means the signal tracks the service, not an arbitrary process in it.

Being event-driven rather than periodic keeps cost proportional to process churn instead of to fleet size.

## Related

- [Process discovery and survey mode](../pipeline/process-discovery-and-survey.md)
- [Exporters and telemetry surface](../05-EXPORTERS.md)
- [Operating and deploying Beyla](../08-OPERATIONS.md)

## Sources

No `raw/` sources yet — this page is synthesized from code at HEAD.

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, decisions, dates, incident references the synthesis missed. -->

---

[← Wiki index](../../index.md)

<!-- atomic: keep this page ≤600 words. New scope → new concept page that builds on this one. Do not append paragraphs here. -->
