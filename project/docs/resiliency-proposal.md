# n8n Resiliency Proposal: Queue Mode, Stalled Approvals and Dead Letter Queue

**To:** CTO
**From:** DevOps
**Re:** n8n production crash of 2023-11-18

---

## 1. Summary

A surge of bulk booking-modification webhooks exhausted the memory of our single n8n container. n8n ran in `regular` mode, which keeps its queue in memory. When the container crashed, it lost 542 queued jobs and every refund approval paused on a Wait node. Webhooks were also dropped during the surge.

This proposal covers three changes:

1. Move n8n to Queue Mode, using Redis as the queue and separate worker containers (`docker-compose-queue-mode.yaml`, `docs/architecture-diagram.md`).
2. Escalate any refund approval that gets no response within 24 hours.
3. Store any operation that still fails after 3 retries in a PostgreSQL Dead Letter Queue (DLQ) table so it can be replayed.

---

## 2. Failure Analysis (`logs/n8n-error-logs.json`)

Heap usage climbed from 75% (19:35) to 88% (19:38) and 96% (19:41). At 19:42:01 the process crashed with `JavaScript heap out of memory`.

### Failure Point 1: Dropped webhook events

- **Evidence:** 19:40:01 `In-memory execution queue backing up: 540+ items pending processing.` Then three `Dropped webhook event: Queue capacity overflow, execution rejected for incoming payload.` messages at 19:41:25 to 19:41:26 (2 × `wf_flight_booking_mod`, 1 × `wf_hotel_sync`).
- **Cause:** Webhook intake and execution shared one process and one in-memory queue. Once that queue was full, new events were rejected.
- **Risk:** Booking changes and inventory updates are lost before any record of them exists.

### Failure Point 2: Queued and running executions lost on restart

- **Evidence:** 19:45:10 `Startup recovery failed: In-memory queue cleared during restart. 542 pending jobs permanently lost.` Executions 852938 to 852941: `Active execution lost during restart. In-memory state unrecoverable.`
- **Cause:** Pending jobs and running execution state existed only in process memory.
- **Risk:** Any crash or restart permanently loses all in-flight work.

### Failure Point 3: Stalled Wait nodes ("Wait and resume" approvals)

- **Evidence:** From 19:46 to 19:58, manager resume webhooks for executions 852931, 852935, 841002, 841099, 830591, 830605 and 840111 all failed with `Failed to resume execution: Wait node state not found in memory. Execution context was dropped during container restart.`
- **Cause:** The wait context of paused executions was held in memory and wiped by the restart.
- **Risk:** Approvals that managers believe they acted on are never processed. More than 200 refunds were affected. Approvals also have no timeout, so an unanswered one waits forever.

---

## 3. Target Architecture and Data Flow

| Component | Role |
|---|---|
| `n8n` (main web process) | Receives webhooks and approval resume calls, saves the execution in PostgreSQL, pushes the job ID to Redis |
| `redis` | Persistent queue (AOF) between intake and processing |
| `n8n-worker` (2+, scalable) | Pulls jobs from Redis and runs the executions |
| `postgres` | Executions, Wait-node state, DLQ table |

The Compose file switches every n8n process to queue mode with `EXECUTIONS_MODE=queue`, `QUEUE_BULL_REDIS_HOST`, `QUEUE_BULL_REDIS_PORT` and `QUEUE_BULL_REDIS_PASSWORD`. All secrets come from environment variables, and the file contains no IP addresses.

**Data flow:**

1. A webhook arrives at `n8n` main.
2. Main saves the execution in PostgreSQL and pushes its job ID to Redis.
3. Main acknowledges the caller right away and does no execution work itself.
4. A worker pulls the job, runs it, and writes the results to PostgreSQL.

**No single point of failure between intake and workers:**

| Hop | Failure | Why work is not lost |
|---|---|---|
| Main to Redis | Main node crashes | Jobs already in Redis keep running on workers. Wait state is in PostgreSQL. Main restarts automatically (`restart: unless-stopped`). In production, run 2+ main/webhook instances behind the load balancer. |
| Redis | Redis restarts | AOF persistence restores the queue. In production, use managed HA Redis (primary and replica with automatic failover). |
| Redis to worker | A worker crashes | The other workers keep pulling jobs. The stalled job is picked up again by a healthy worker. |
| PostgreSQL | Primary fails | In production, use managed HA PostgreSQL (primary and standby). Jobs wait in Redis until the database is back. |

Intake and processing communicate only through Redis and PostgreSQL. Either side can crash, restart or scale without losing the other's work.

---

## 4. Stalled Approval Strategy: Wait and Resume with 24-Hour Escalation

The `wf_refund_approval` workflow works as follows:

1. **Webhook** receives the refund request.
2. **Notify Reviewer** sends approve and reject links built from `{{$execution.resumeUrl}}?decision=approved|rejected`.
3. **Wait for Human Approval** (Wait node):
   - Resume: `On Webhook Call`
   - Limit Wait Time: `true`, `After Time Interval`, `24` `Hours`
4. **IF: Decision Received?** checks whether `{{ $json.query.decision }}` is not empty.
   - **True** (the reviewer responded): process the approval or rejection.
   - **False** (24 hours passed with no response): go to escalation.
5. **Escalation:** post to the `#refund-escalations` Slack channel and email the finance lead. The message includes the transaction ID, the amount, the original reviewer, and new approve and reject links. The workflow then waits again on a second Wait node until finance decides.

In Queue Mode the wait is stored in PostgreSQL (`status=waiting`, `waitTill`), so neither the resume link nor the 24-hour timeout depends on any container staying alive.

---

## 5. Dead Letter Queue (DLQ) Routing to PostgreSQL

**DLQ table:**

```sql
CREATE TABLE dlq_failed_executions (
    id             BIGSERIAL PRIMARY KEY,
    workflow_id    TEXT        NOT NULL,
    execution_id   TEXT,
    failed_node    TEXT,
    error_message  TEXT,
    payload        JSONB       NOT NULL,
    attempts       INT         NOT NULL,
    status         TEXT        NOT NULL DEFAULT 'pending_replay', -- pending_replay | replayed
    failed_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    replayed_at    TIMESTAMPTZ
);
```

**Routing mechanism:**

1. **Retry.** Every external call (Amadeus sync, hotel inventory sync, database writes, refund processor) uses these node settings: `Retry On Fail` on, `Max Tries` `4` (1 attempt plus 3 retries), `Wait Between Tries` `5000` ms.
2. **Capture.** `On Error` is set to `Continue (using error output)`. The error output fires only after the 3rd retry fails. It connects to a **Postgres: Insert** node that writes the workflow ID, execution ID, failed node, error message, original payload and `attempts = 4` to `dlq_failed_executions`. A Slack alert then goes to `#n8n-alerts`.
3. **Catch-all.** Every workflow sets **Settings → Error Workflow** to `wf_dlq_handler`, so errors outside those nodes are captured too. Its **Error Trigger** retries the failed execution up to 3 times through the n8n API. If the 3rd retry also fails, it inserts the payload into the same table.
4. **Replay.** `wf_dlq_replay` runs every 15 minutes (or manually). It re-runs rows with `status='pending_replay'` and marks them `replayed` once they succeed.

---

## 6. Recovery Plan and Time-to-Recovery (TTR)

**Recovery steps for stalled workflows:**

1. **Detect:** a Slack alert, or `SELECT count(*) FROM dlq_failed_executions WHERE status='pending_replay';`
2. **Triage:** group the rows by `workflow_id` and `error_message`, and confirm the downstream system is healthy again.
3. **Replay:** run `wf_dlq_replay`, or wait for its 15-minute schedule.
4. **Verify:** recovery is complete when no `pending_replay` rows remain.

| Scenario | Before | After | Estimated TTR |
|---|---|---|---|
| Downstream API outage; payload fails 3 retries | Payload lost | Payload in DLQ, alert, replay | **≤ 25 minutes** after the API recovers (≤ 10 min triage + ≤ 15 min to the next replay run) |
| Worker crashes mid-execution | All work lost | Another worker takes over the stalled job | **~1 minute** |
| Main node crashes | 542 jobs and all approvals lost | Workers keep draining Redis; main restarts | **< 1 minute**; 0 jobs lost |
| Reviewer does not respond | Waits forever, or lost on restart | Escalated to finance | **24 hours** to escalation |
| Repeat of the ~540-job surge | Out-of-memory crash | Jobs wait in Redis; 2 workers × 10 concurrent jobs drain them | **~1 minute** (28 rounds × ~2 s typical run) |
