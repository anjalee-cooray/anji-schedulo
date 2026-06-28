# INF-004 · Serverless Provisioning Flow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Overview

AnjiSchedulo uses serverless AWS services for specific bounded operations that do not require long-running containers: S3 Object Lambda (PII redaction on event archives), CloudWatch scheduled rules (appointment reminder dispatch), and AWS Lambda for lightweight event-driven tasks. This document covers how these serverless resources are provisioned and connected to the rest of the architecture.

---

## 2. Serverless Components

| Component | AWS Service | Trigger | Purpose |
|---|---|---|---|
| Reminder dispatcher | EventBridge Scheduled Rule | Every 15 minutes | Find appointments starting in 24h; publish `appointment.reminder_due` events |
| PII archive redactor | S3 Object Lambda | `customer.erased` event via SQS | Redact PII fields from event archive NDJSON files in S3 |
| Outbox archive exporter | EventBridge Scheduled Rule | 1st of each month, 02:00 UTC | Export `appointment_events` to `event-archives` S3 bucket (NDJSON) |
| Tenant data export | Lambda (invoked by `ops-service`) | On-demand (FR012) | Generate NDJSON archive of all tenant data; upload to `tenant-exports` bucket; return presigned URL |

---

## 3. Reminder Dispatcher

### Provisioning

```
Terraform creates:
  aws_lambda_function: reminder-dispatcher-{env}
    Runtime: nodejs20.x
    Handler: index.handler
    Image: {ecr-uri}/anji-schedulo/{env}/reminder-dispatcher:{tag}
    Timeout: 60s
    Memory: 256 MB
    VPC: private subnets (needs RDS access)
    Environment:
      DB_HOST: {rds-endpoint}
    Secrets (injected via Lambda env from Secrets Manager):
      DB_PASSWORD

  aws_cloudwatch_event_rule: reminder-dispatch-{env}
    ScheduleExpression: rate(15 minutes)

  aws_cloudwatch_event_target:
    Target: reminder-dispatcher Lambda
    Input: '{"env": "{env}"}'
```

### Flow

```
EventBridge fires every 15 minutes
          │
          ▼
reminder-dispatcher Lambda
    SELECT appointment_id, tenant_id, staff_id, customer_id, slot_start
    FROM appointments
    WHERE status = 'confirmed'
      AND slot_start BETWEEN NOW() + INTERVAL '23h 45m'
                        AND NOW() + INTERVAL '24h 15m'
      AND reminder_sent = false
    (RLS enforced — Lambda sets app.tenant_id per row via service account)
          │
          ▼
    For each appointment:
      Publish appointment.reminder_due to SNS
      UPDATE appointments SET reminder_sent = true WHERE appointment_id = ?
          │
          ▼
notification-service (SQS consumer) → SendGrid / Twilio
```

---

## 4. PII Archive Redactor

### Provisioning

```
Terraform creates:
  aws_s3_object_lambda_access_point: pii-redactor-{env}
    Supporting access point: event-archives-ap-{env}
    TransformationConfiguration:
      Actions: GetObject
      ContentTransformation: Lambda: pii-redactor-{env}

  aws_lambda_function: pii-redactor-{env}
    Runtime: nodejs20.x
    Timeout: 300s
    Memory: 1024 MB
    VPC: private subnets
    Trigger: SQS (customer.erased events via anji-schedulo-{env}-pii-redactor-customer-erased.fifo)
```

### Flow

```
customer.erased event → SNS → SQS (pii-redactor queue)
          │
          ▼
pii-redactor Lambda
    1. Parse customer_id and tenant_id from event
    2. List all objects in event-archives-{env} for this tenant prefix
    3. For each NDJSON file:
       - Read via S3 Object Lambda access point
       - Scan for records where customer_id matches
       - Replace: name → "REDACTED-{sha256(customer_id)}"
                  email → "redacted-{sha256}@erased.anji-schedulo.com"
                  phone → null
       - Write back (PUT object with new content)
    4. Write audit_log entry: 'event_archive_pii_redacted'
          │
          ▼
Completes within 24h of erasure request (FR013)
```

---

## 5. Outbox Archive Exporter

### Provisioning

```
Terraform creates:
  aws_lambda_function: outbox-exporter-{env}
    Runtime: nodejs20.x
    Timeout: 900s (15 min — large exports)
    Memory: 2048 MB
    VPC: private subnets

  aws_cloudwatch_event_rule: monthly-archive-{env}
    ScheduleExpression: cron(0 2 1 * ? *)  (1st of month, 02:00 UTC)
```

### Flow

```
EventBridge fires on 1st of month, 02:00 UTC
          │
          ▼
outbox-exporter Lambda
    SELECT * FROM appointment_events
    WHERE occurred_at >= date_trunc('month', NOW() - INTERVAL '1 month')
      AND occurred_at <  date_trunc('month', NOW())
    (No RLS on this export — service account has cross-tenant read for archival)
    Stream results as NDJSON → gzip
          │
          ▼
    PUT to s3://anji-schedulo-event-archives-{env}/
        {year}/{month}/appointment_events.ndjson.gz
          │
          ▼
    CloudWatch Logs: log archive job completion + row count
```

---

## 6. Tenant Data Export Lambda

### Provisioning

```
Terraform creates:
  aws_lambda_function: tenant-exporter-{env}
    Runtime: nodejs20.x
    Timeout: 300s
    Memory: 1024 MB
    VPC: private subnets
    Invoked by: ops-service (aws:lambda:InvokeFunction permission)
```

### Flow

```
Tenant Admin requests export → ops-service
    POST /ops/tenants/{id}/export
          │
          ▼
ops-service invokes tenant-exporter Lambda:
    { tenant_id: "{tid}", requested_by: "{user_id}" }
          │
          ▼
tenant-exporter Lambda:
    For each tenant-scoped table:
      SELECT * WHERE tenant_id = {tid}
    Stream as NDJSON → gzip
    PUT to s3://anji-schedulo-tenant-exports-{env}/{tid}/{iso-date}.ndjson.gz
    Generate presigned URL (24h expiry)
    Return presigned URL to ops-service
          │
          ▼
ops-service sends presigned URL to tenants.owner_email via SendGrid
(URL is NOT returned in the API response — email delivery only)
```

---

## 7. IAM Roles for Lambda Functions

Each Lambda function has its own task-specific IAM role (least privilege):

| Lambda | Permissions |
|---|---|
| reminder-dispatcher | `rds-data:ExecuteStatement`, `sns:Publish` (reminder topic only) |
| pii-redactor | `s3:GetObject`, `s3:PutObject` (event-archives bucket only), `sqs:ReceiveMessage`, `sqs:DeleteMessage` |
| outbox-exporter | `rds-data:ExecuteStatement` (read-only), `s3:PutObject` (event-archives bucket only) |
| tenant-exporter | `rds-data:ExecuteStatement` (tenant-scoped), `s3:PutObject` (tenant-exports bucket only), `s3:GeneratePresignedUrl` |

---

## 8. Traceability

| Component | FR | NFR | Doc |
|---|---|---|---|
| Reminder dispatcher | FR007 | NFR002 | A10-integrations.md |
| PII archive redactor | FR013 | — | A6-data-privacy-arch.md |
| Outbox archive exporter | FR011 | — | A12-disaster-recovery.md |
| Tenant data export | FR012 | — | A6-data-privacy-arch.md |
