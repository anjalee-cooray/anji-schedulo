# O4 · Runbook

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

This runbook covers the most common operational tasks for AnjiSchedulo: DLQ triage, event replay, tenant management, and routine monitoring checks. For incident response procedures, see O5. For rollback, see O3 and CD-004.

---

## 2. DLQ Triage

### 2.1 Detecting DLQ Issues

ALT003 fires when any DLQ has depth > 0. To identify which queues are affected:

```bash
# List all DLQ depths
aws sqs list-queues --queue-name-prefix anji-schedulo-production --output json \
  | jq -r '.QueueUrls[]' \
  | grep dlq \
  | while read url; do
      depth=$(aws sqs get-queue-attributes --queue-url "$url" \
        --attribute-names ApproximateNumberOfMessages \
        | jq -r '.Attributes.ApproximateNumberOfMessages')
      echo "$depth $url"
    done \
  | sort -rn \
  | head -20
```

### 2.2 Inspecting a DLQ Message

```bash
# Receive one message from a DLQ (does NOT delete it)
aws sqs receive-message \
  --queue-url https://sqs.eu-west-1.amazonaws.com/{account}/anji-schedulo-production-{consumer}-{event}-dlq.fifo \
  --max-number-of-messages 1 \
  --attribute-names All \
  --message-attribute-names All
```

Check the message body for:
- `event_type` — which event failed
- `tenant_id` — which tenant is affected
- `correlation_id` — trace the original request in Grafana Loki/Tempo
- `failure_reason` — in the `MessageAttributes` (added by consumer on failure)

### 2.3 Re-queuing a DLQ Message

If the failure was transient (provider timeout, temporary DB connection issue):

```bash
# Via ops-service API (preferred — validates message before re-queuing)
curl -X POST https://api.anji-schedulo.com/ops/dlq/requeue \
  -H "Authorization: Bearer {operator_jwt}" \
  -H "Content-Type: application/json" \
  -d '{
    "queue_url": "https://sqs.eu-west-1.amazonaws.com/{account}/anji-schedulo-production-{consumer}-{event}-dlq.fifo",
    "message_receipt_handle": "{receipt_handle}",
    "reason": "Transient SendGrid 503 — provider recovered"
  }'
```

**Idempotency:** Consumer services use `inbox_records` to deduplicate — re-queuing a message that was already successfully processed produces no additional side effects.

### 2.4 Quarantining a Poison-Pill Message

If a message cannot be re-queued (invalid payload, schema mismatch):

```bash
# Delete from DLQ after archiving the body for investigation
aws sqs delete-message \
  --queue-url {dlq-url} \
  --receipt-handle {receipt_handle}
# Log the deletion in ops-service audit trail via POST /ops/dlq/quarantine
```

### 2.5 DLQ Names Reference

| Consumer | Event | DLQ Name |
|---|---|---|
| notification-service | appointment.confirmed | `anji-schedulo-production-notification-appointment-confirmed-dlq.fifo` |
| notification-service | appointment.cancelled | `anji-schedulo-production-notification-appointment-cancelled-dlq.fifo` |
| notification-service | appointment.rescheduled | `anji-schedulo-production-notification-appointment-rescheduled-dlq.fifo` |
| notification-service | appointment.reminder_due | `anji-schedulo-production-notification-appointment-reminder-due-dlq.fifo` |
| dashboard-service | appointment.confirmed | `anji-schedulo-production-dashboard-appointment-confirmed-dlq.fifo` |
| dashboard-service | appointment.cancelled | `anji-schedulo-production-dashboard-appointment-cancelled-dlq.fifo` |
| analytics-service | appointment.confirmed | `anji-schedulo-production-analytics-appointment-confirmed-dlq.fifo` |
| payment-service | appointment.cancelled | `anji-schedulo-production-payment-appointment-cancelled-dlq.fifo` |

---

## 3. Event Replay

Use event replay when derived read models (dashboards, analytics) show incorrect or stale data, but `appointment_events` (the source of truth) is intact.

### 3.1 Trigger a Replay

```bash
# Via ops-service — replay all events for a specific tenant
curl -X POST https://api.anji-schedulo.com/ops/replay \
  -H "Authorization: Bearer {operator_jwt}" \
  -H "Content-Type: application/json" \
  -d '{
    "scope": "tenant",
    "tenant_id": "tid-abc-123",
    "target_consumers": ["dashboard-service", "analytics-service"]
  }'
```

Supported scopes:

| Scope | Parameters |
|---|---|
| `tenant` | `tenant_id` |
| `date_range` | `from_date`, `to_date` (ISO-8601) |
| `event_type` | `event_type` (e.g. `appointment.confirmed`) |
| `specific_ids` | `appointment_ids[]` |
| `full` | No parameters — replays everything (use with caution) |

### 3.2 Monitor Replay Progress

```bash
# Check replay job status
curl https://api.anji-schedulo.com/ops/replay/{job_id} \
  -H "Authorization: Bearer {operator_jwt}"
```

Job states: `pending` → `running` → `completed` | `failed`

### 3.3 Verify After Replay

```sql
-- Verify dashboard view matches event count
SELECT
  tdv.confirmed_bookings_today,
  COUNT(*) AS actual_confirmed_today
FROM tenant_dashboard_views tdv
JOIN appointments a
  ON a.tenant_id = tdv.tenant_id
  AND a.status = 'confirmed'
  AND a.created_at::date = tdv.view_date
WHERE tdv.tenant_id = 'tid-abc-123'
GROUP BY tdv.confirmed_bookings_today;
```

---

## 4. Tenant Management

### 4.1 Provision a New Tenant

Handled automatically via the product UI (tenant self-registration, FR001). For manual operator provisioning:

```bash
curl -X POST https://api.anji-schedulo.com/ops/tenants \
  -H "Authorization: Bearer {operator_jwt}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Harmony Wellness Clinic",
    "slug": "harmony-wellness",
    "owner_email": "admin@harmonywellness.com",
    "plan": "pro"
  }'
```

### 4.2 Suspend a Tenant

```bash
curl -X PATCH https://api.anji-schedulo.com/ops/tenants/{tenant_id}/suspend \
  -H "Authorization: Bearer {operator_jwt}" \
  -d '{"reason": "Payment failure — outstanding invoice"}'
```

Suspended tenants: booking page returns 503, admin UI shows suspension notice. Existing appointments are not cancelled.

### 4.3 Trigger GDPR Erasure

```bash
curl -X POST https://api.anji-schedulo.com/ops/tenants/{tenant_id}/customers/{customer_id}/erase \
  -H "Authorization: Bearer {operator_jwt}" \
  -d '{"reason": "Customer erasure request submitted via support ticket ST-1234"}'
```

Pseudonymisation completes in the same request. S3 archive PII redaction completes within 24 hours (FR013).

---

## 5. Routine Monitoring Checks

Perform these checks at the start of each business day and after every production deployment:

| Check | Where | Expected |
|---|---|---|
| Booking saga success rate | Grafana → Booking Operations dashboard | ≥ 99.9% |
| Booking saga p95 latency | Grafana → Booking Operations dashboard | < 3 seconds |
| DLQ depth | Grafana → DLQ Triage dashboard | 0 across all queues |
| Outbox backlog | Grafana → Booking Operations → outbox_backlog_total | < 10 records |
| RDS CPU | Grafana → Infrastructure Health | < 70% |
| Redis hit rate | Grafana → Infrastructure Health | > 80% |
| Active P1/P2 alerts | Grafana → Alerting | 0 |
| Isolation monitor | Grafana → Security dashboard → isolation_monitor_violations_total | 0 (always) |

---

## 6. Useful Loki Queries for On-Call

```logql
# Errors in the last 30 minutes (all services)
{env="production", level="error"} | json | __error__=""

# Trace a specific booking by correlation_id
{env="production"} |= "\"correlation_id\":\"corr-xyz-789\""

# Saga failures in the last hour
{service="booking-command-service", env="production", level="error"} | json

# Notification failures
{service="notification-service", env="production", level="error"} | json

# Recent DLQ events
{env="production"} |= "dlq" | json
```

---

## 7. Traceability

| Operation | NFR | Alert | Doc |
|---|---|---|---|
| DLQ triage | NFR016 | ALT003 | O5-incident-response.md |
| Event replay | NFR009 | — | RES-003-disaster-recovery.md |
| GDPR erasure | — | — | FR013, A6-data-privacy-arch.md |
| Daily monitoring | NFR001, NFR004 | ALT001–ALT008 | A11-observability.md |
