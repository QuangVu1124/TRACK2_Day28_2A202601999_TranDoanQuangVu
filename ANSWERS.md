# Day 28 Track 2 — Individual reflection and Q&A

## Submission status

This is an individual submission by Trần Đoàn Quang Vũ. The four starter seams
were implemented in `src/lab28_platform/integration_tasks.py` and the live
stack was exercised on 2026-09-03.

Verified locally:

- `87 passed` from `starter-tests tests`;
- `56 passed, 16 deselected` from the non-GPU, non-LangSmith integration suite;
- J1 golden path: `12 passed, 3 skipped` because the GPU gate had no real vLLM;
- J2 replay/idempotency: `9 passed`;
- Ruff, matrix, portability, manifest and Compose validation all passed.

The 16 deselected tests are environment gates, not hidden failures: they need a
real vLLM endpoint or a LangSmith credential. The honest local vLLM evidence is
in [`evidence/ip07-vllm-identity.json`](evidence/ip07-vllm-identity.json).

## Architecture and ownership

| Point | Boundary | Individual ownership / proof |
|---|---|---|
| IP01 | Gateway/API → Kafka | Kafka event key and trace/idempotency headers; `ip01-kafka-consume.json` |
| IP02 | Kafka → Airflow | Manual-commit batch consumer, DAG run and asset events; `ip02-airflow-run.json` |
| IP03 | Airflow/Spark → Delta | Replay-safe MERGE and Delta history; `ip03-delta-history.json` |
| IP04 | Delta → Feast | Shared feature references and online materialization; `ip04-feast-online.json` |
| IP05 | Delta → Qdrant | Deterministic document point IDs; `ip05-qdrant-search.json` |
| IP06 | Data/evaluation → MLflow | Champion alias, provenance and rollback; `ip06-mlflow-release.json` |
| IP07 | Champion → vLLM | Contract is implemented, but a real GPU-backed vLLM was not available locally; `ip07-vllm-identity.json` is intentionally unverified |
| IP08 | Envoy → API | Request ID, local rate limit and health routing; `ip08-gateway.json` |
| IP09 | Components → Prometheus/Grafana | Scrape targets, dashboard and alert rules; `ip09-prometheus-targets.json` and `ip09-grafana-dashboards.json` |
| IP10 | Components → OpenTelemetry/Jaeger | Same trace ID across synchronous and asynchronous legs; `ip10-trace.json` |

The architecture diagram is
[`docs/images/lab28-architecture-overview.svg`](docs/images/lab28-architecture-overview.svg)
and the role split is documented in
[`docs/team-role-cards.md`](docs/team-role-cards.md).

## Main technical choices

1. Kafka is at-least-once. The consumer commits offsets only after the Delta
   handler succeeds; malformed messages go to the DLQ so they cannot block a
   partition forever.
2. Delta deduplicates by `idempotency_key` and keeps the greatest
   `(occurred_at, event_id)`. This makes a replay safe even when Kafka delivers
   the same event more than once.
3. Trace context is carried in the Kafka `traceparent` header and in the
   Airflow run configuration. An empty trace header is omitted rather than
   emitted as an invalid value.
4. Feast requests use the canonical `FEATURE_REFS` list from `contracts.py`.
   The serving contract therefore has one source of truth for feature names.
5. Readiness is severity-aware: a failed mandatory dependency is
   `not_ready`, an optional dependency failure is `degraded`, and a healthy
   dependency set is `ready`.
6. Envoy owns edge policy. Its token bucket returns 429 before a burst reaches
   the API, while `/healthz` is answered directly by the gateway.

## Recovery and rollback explanation

The recovery record is in
[`evidence/failure-recovery.md`](evidence/failure-recovery.md). The J4 journey
covers Feast degradation, mandatory Qdrant failure, poison messages, DLQ replay
and the no-duplicate Delta assertion. J3 covers MLflow promotion and moving
the champion alias back to its previous version. The rollback mechanism changes
the alias, not application code or a container image.

## Observability explanation

The platform exposes request counters and latency histograms to Prometheus,
consumer lag through the Kafka exporter, component readiness gauges, and two
actionable alerts in [`monitoring/alerts.yml`](monitoring/alerts.yml). Grafana
is provisioned from the repository. Jaeger evidence demonstrates the gateway,
API, Kafka producer/consumer, Airflow and Spark Delta merge spans on one trace.

The serving-leg spans and the vLLM identity cannot be claimed on this laptop:
there is no real GPU vLLM endpoint. LangSmith is also an optional credentialed
gate. No mock service or fabricated trace was used.

## Production gaps and next steps

- Run IP07 and the serving-leg IP10 tests against the lecturer-provided real
  vLLM endpoint, then replace the honest unavailable identity evidence with
  `/version`, `/v1/models` and `vllm:` metrics from that endpoint.
- Run the LangSmith gate only when `LANGSMITH_API_KEY` is supplied.
- Repeat the load profile with 16 workers and with representative `/ask`
  traffic; the current 200-request baseline is documented in
  [`evidence/load-profile.json`](evidence/load-profile.json).
- In a Kubernetes environment, record an actual Argo CD drift/self-heal and
  desired-revision rollback, not only manifest validation.
- Use external secret management, immutable image digests, multi-zone Kafka/
  storage and a real incident notification route for production.

## Q&A preparation

**Why not exactly-once?** Kafka delivery and downstream processing are kept
at-least-once because the durable boundary is Delta. Idempotent MERGE gives the
desired user-visible result without depending on a distributed transaction
across Kafka, Spark, Feast and Qdrant.

**What happens when Delta is unavailable?** The handler raises before the
consumer commit. Kafka redelivers the batch later; valid messages are not
silently dropped.

**What is the difference between the three readiness states?** `ready` means
mandatory and optional probes pass; `degraded` means only optional probes
failed; `not_ready` means at least one mandatory probe failed and the gateway
must stop routing traffic to that instance.

**What proves trace continuity?** The caller chooses a W3C trace ID, the API
puts its context into Kafka headers, the consumer extracts it, and Airflow
receives the same trace ID in its run configuration. Jaeger evidence checks the
span names rather than trusting a log line.

**What did I implement personally?** I implemented the four integration task
functions, ran the unit/live verification, started the full local stack,
collected the IP evidence, and prepared this reflection plus the recovery and
performance records.

