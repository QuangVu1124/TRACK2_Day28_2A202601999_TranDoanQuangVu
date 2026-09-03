# Failure and recovery evidence

Capture date: **2026-09-03**  
Stack: Docker Compose full profile, Windows PowerShell  
Verification command:

```text
uv run --python 3.11 pytest integration-tests -m "not gpu and not langsmith" -q --tb=short
```

Result: **56 passed, 16 deselected**. The deselected cases are the real-vLLM
and LangSmith gates. The scenarios below are the recovery behaviors exercised
by `integration-tests/test_j4_degraded_recovery.py`; replay behavior is also
covered by `integration-tests/test_j2_idempotent_replay.py`.

## Incident record

| Scenario | Failure injection | Expected signal | Recovery / proof |
|---|---|---|---|
| Optional Feast outage | Stop the `feast` service using the test dependency fixture | The Feast component becomes unready, but the overall rotation verdict does not become `not_ready`; the owner and reason are present | Start/restore Feast; the probe returns healthy again. The non-GPU J4 readiness test passed. |
| Mandatory Qdrant outage | Stop the `qdrant` service | `/ready` returns HTTP 503 with status `not_ready`; the Qdrant detail is actionable | Restore Qdrant; its probe returns healthy and the baseline status code returns. |
| Poison Kafka message | Publish malformed bytes to `data.raw` beside one valid feedback event | Airflow run succeeds; malformed bytes are parked in `data.raw.dlq` with topic, partition, offset, key and raw payload | The valid event still reaches Delta. The poison message is not lost and does not block the batch. |
| DLQ replay | Put a valid envelope and an unparseable envelope in the DLQ, then replay | Valid envelope is republished to its original topic; unparseable envelope is skipped | Airflow processes the replay; Delta contains exactly one row for the idempotency key. |
| Kafka/Delta redelivery | Re-submit/replay the same event batch | Kafka may contain multiple deliveries, but Delta MERGE does not create a second logical row | J2 passed 9/9 and asserts the replayed event produces one Delta row and one deterministic Qdrant point. |

## No-data-loss invariant

The implementation order is:

1. poll and validate the raw batch;
2. publish poison records to the DLQ;
3. merge valid events into Delta and run downstream updates;
4. commit Kafka offsets only after the handler returns successfully.

If the durable handler fails, the good event is not committed and Kafka can
redeliver it. If Kafka delivers it twice, the Delta idempotency key and the
stable Qdrant point ID make the result converge to one logical record. This is
asserted by J2 and by the J4 tests
`test_the_good_record_in_the_same_batch_still_reached_the_lakehouse` and
`test_the_replayed_event_does_not_duplicate_the_row`.

## Readiness evidence

The distinction is intentional:

- `ready`: all mandatory and optional probes pass;
- `degraded`: mandatory probes pass but an optional probe, such as Feast or
  vLLM under the local policy, is unavailable;
- `not_ready`: a mandatory probe, such as Qdrant for grounded retrieval, is
  unavailable, so the gateway must remove the instance from rotation.

The J4 tests verify both the HTTP status and the component-level breakdown.
The recovery runbook is
[`runbooks/failure-injection.md`](../runbooks/failure-injection.md).

## Limitations of this record

This file records the reproducible automated incident checks. It does not
claim an unexecuted manual Kubernetes incident or a real vLLM outage. For a
classroom recording, add the wall-clock timestamps, terminal screenshots,
Kafka offsets, Delta versions, Qdrant counts and the operator name while
running one scenario live.

