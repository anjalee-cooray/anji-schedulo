---
title: Notification Design
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D9 · Notification Design

## 1. Overview

The notification-service is a fully decoupled asynchronous subscriber of all appointment and tenant lifecycle events. It never blocks, delays, or rolls back any booking operation — a delivery failure is fully isolated to the notification pipeline (BR007). All notification content is derived exclusively from the event payload snapshot (Event-Carried State Transfer); the notification-service never calls back to booking-command-service, tenant-service, or any other domain service at send time.

Notification delivery supports three channels across pricing tiers:

| Channel | Provider | Plans | Auth |
|---|---|---|---|
| Email | SendGrid | Starter, Pro, Enterprise | API key (`anji-schedulo/{env}/sendgrid/api-key`) |
| SMS | Twilio | Pro, Enterprise | Account SID + Auth Token (`anji-schedulo/{env}/twilio/auth-token`) |
| WhatsApp | Twilio WhatsApp API | Enterprise only | Same credentials as SMS |

Channel selection at runtime: notification-service reads the `plan` field from the event payload to determine which channels are available for the triggering tenant. No back-call to tenant-service is needed.

---

## 2. Notification Events

### 2.1 `appointment.confirmed` — UJ002

| Recipient | Channel | Plans | Template |
|---|---|---|---|
| Customer | Email | All | `appointment-confirmed-customer-email` |
| Customer | SMS | Pro, Enterprise | `appointment-confirmed-customer-sms` |
| Staff | Email | All | `appointment-confirmed-staff-email` |

**Customer email content** (derived from `customer_snapshot`, `staff_snapshot`, `service_snapshot`, `slot_date`, `slot_start`, `slot_end`, `amount_charged`):
- Subject: `Your appointment is confirmed — [service_name] with [staff_name] on [slot_date] at [slot_start]`
- Body: service name, staff name, date, start time, end time, duration in minutes, amount charged (if any), cancellation policy window (from event payload), booking reference (`appointment_id`)
- CTA: "Cancel or reschedule" deep-link to the tenant booking page

**Staff email content** (derived from `staff_snapshot`, `customer_snapshot`, `service_snapshot`, `slot_date`, `slot_start`):
- Subject: `New booking: [customer_name] — [service_name] on [slot_date] at [slot_start]`
- Body: customer name, customer email, service, date, time

**Customer SMS content**: `[TenantName] Confirmed: [service_name] with [staff_name] on [slot_date] at [slot_start]. Ref: [appointment_id_short]`

---

### 2.2 `appointment.cancelled` — UJ003

| Recipient | Channel | Plans | Template |
|---|---|---|---|
| Customer | Email | All | `appointment-cancelled-customer-email` |
| Customer | SMS | Pro, Enterprise | `appointment-cancelled-customer-sms` |
| Staff | Email | All | `appointment-cancelled-staff-email` |

**Customer email content** (derived from `customer_snapshot`, `staff_snapshot`, `service_snapshot`, `slot_date`, `slot_start`, `amount_eligible_for_refund`):
- Subject: `Your appointment has been cancelled — [service_name] on [slot_date]`
- Body: appointment details, cancellation confirmation, refund notice (only if `amount_eligible_for_refund > 0`): "A refund of [amount] will be returned to your original payment method within 5–10 business days."

**Staff email**: `Appointment cancelled: [customer_name] — [service_name] on [slot_date] at [slot_start]`

---

### 2.3 `appointment.rescheduled` — UJ004

| Recipient | Channel | Plans | Template |
|---|---|---|---|
| Customer | Email | All | `appointment-rescheduled-customer-email` |
| Customer | SMS | Pro, Enterprise | `appointment-rescheduled-customer-sms` |

**Customer email content** (derived from `old_slot_date`, `old_slot_start`, `new_slot_date`, `new_slot_start`, `new_slot_end`, `staff_snapshot`, `service_snapshot`):
- Subject: `Your appointment has been rescheduled — [service_name] now on [new_slot_date]`
- Body: old date/time (struck through), new date/time, staff name, service, updated booking reference

---

### 2.4 `tenant.provisioned` — UJ001

| Recipient | Channel | Plans | Template |
|---|---|---|---|
| Tenant Admin | Email | All | `welcome-tenant-email` |

**Content** (derived from `owner_name`, `owner_email`, `name`, `plan`, setup link):
- Subject: `Welcome to AnjiSchedulo — let's set up your booking page`
- Body: personalised greeting with `owner_name`, plan name and included features, direct setup link, support contact

---

### 2.5 `tenant.data_exported` — UJ007

| Recipient | Channel | Plans | Template |
|---|---|---|---|
| Tenant Admin | Email | All | `data-export-ready-email` |

**Content** (derived from `owner_email`, `signed_url`, `export_includes`, `occurred_at`):
- Subject: `Your AnjiSchedulo data export is ready`
- Body: list of included data categories, signed download link (expires 24 hours from `occurred_at`), warning about link expiry

---

### 2.6 `analytics.velocity_spike_detected` — v1.0

| Recipient | Channel | Plans | Template |
|---|---|---|---|
| Tenant Admin | Email | Pro, Enterprise | `analytics-velocity-spike-email` |

**Content**: booking velocity spike detected, window start/end, booking_count_in_window, historical_average, velocity_ratio.

---

### 2.7 `analytics.cancellation_rate_alert` — v1.0

| Recipient | Channel | Plans | Template |
|---|---|---|---|
| Tenant Admin | Email | Pro, Enterprise | `analytics-cancellation-rate-email` |

**Content**: cancellation_rate as percentage, sample_size (50 bookings), window start/end.

---

## 3. Template Naming Convention

| Template Name | Triggering Event | Recipient | Channel |
|---|---|---|---|
| `appointment-confirmed-customer-email` | `appointment.confirmed` | Customer | Email |
| `appointment-confirmed-customer-sms` | `appointment.confirmed` | Customer | SMS |
| `appointment-confirmed-staff-email` | `appointment.confirmed` | Staff | Email |
| `appointment-cancelled-customer-email` | `appointment.cancelled` | Customer | Email |
| `appointment-cancelled-customer-sms` | `appointment.cancelled` | Customer | SMS |
| `appointment-cancelled-staff-email` | `appointment.cancelled` | Staff | Email |
| `appointment-rescheduled-customer-email` | `appointment.rescheduled` | Customer | Email |
| `appointment-rescheduled-customer-sms` | `appointment.rescheduled` | Customer | SMS |
| `welcome-tenant-email` | `tenant.provisioned` | Tenant Admin | Email |
| `data-export-ready-email` | `tenant.data_exported` | Tenant Admin | Email |
| `analytics-velocity-spike-email` | `analytics.velocity_spike_detected` | Tenant Admin | Email |
| `analytics-cancellation-rate-email` | `analytics.cancellation_rate_alert` | Tenant Admin | Email |

---

## 4. Retry Policy

All notification delivery attempts follow exponential backoff:

| Attempt | Delay | Action |
|---|---|---|
| 1 | Immediate | Call provider API (SendGrid / Twilio) |
| 2 | +1 second | Retry on transient error |
| 3 | +2 seconds | Retry |
| 4 | +4 seconds | Final retry |
| Exhausted | — | Set `NotificationRecord.status = dlq`; publish `notification.failed` event |

Retry is not triggered for permanent provider rejections (invalid recipient address, unsubscribed contact — classified as `sent_failed`).

---

## 5. Idempotency

notification-service uses the **InboxRecord pattern** for all consumed events:

1. Before processing any event, insert `(event_id, consumer_name='notification-service')` into `inbox_records`.
2. If the insert fails (unique constraint violation), the event has already been processed — the SQS message is deleted without any side effect.
3. This guarantees exactly-once notification delivery even under SQS at-least-once delivery or during event replay (FR011).

---

## 6. NotificationRecord Schema

Derived from `domain.json` — `NotificationRecord` entity.

| Field | Type | Description |
|---|---|---|
| `notification_id` | UUID (PK) | Unique identifier for this notification attempt |
| `tenant_id` | UUID | Tenant scope — enforced by RLS (BR005) |
| `appointment_id` | UUID (FK, nullable) | Related appointment, if applicable |
| `recipient_type` | enum: `customer \| staff \| admin` | Who receives this notification |
| `recipient_contact` | string (PII) | Email address or phone number — redacted in logs |
| `channel` | enum: `email \| sms \| whatsapp` | Delivery channel used |
| `template_name` | string | Template name from the naming convention table above |
| `status` | enum: `pending \| sent \| failed \| dlq` | Current delivery status |
| `attempt_count` | integer | Number of delivery attempts made (1–4) |
| `last_attempted_at` | timestamp | Timestamp of the most recent attempt |
| `sent_at` | timestamp (nullable) | Set when provider confirms delivery |
| `failure_reason` | string (nullable) | Provider error message on failure |
| `trigger_event_id` | UUID | The `event_id` of the event that triggered this notification — used for InboxRecord deduplication |

---

## 7. Plan-Based Channel Selection

Channel selection logic executed per notification at runtime:

```
plan = event.payload.plan

if plan == 'starter':
    channels = ['email']
elif plan == 'pro':
    channels = ['email', 'sms']
elif plan == 'enterprise':
    channels = ['email', 'sms', 'whatsapp']

for channel in channels:
    if template exists for (event_type, recipient_type, channel):
        dispatch(template, channel, recipient_contact)
```

For each channel, a separate `NotificationRecord` row is created and tracked independently. A WhatsApp failure does not affect the email delivery attempt.

---

## 8. Failure Path

### Single Attempt Failure

On a provider API error (5xx, timeout, or transient network issue):
1. notification-service catches the exception.
2. Increments `NotificationRecord.attempt_count`.
3. Updates `NotificationRecord.last_attempted_at`.
4. Waits the backoff delay (1s / 2s / 4s for attempts 2–4).
5. Retries the provider call.

### Retry Exhaustion (4 Failures)

After 4 consecutive failures:
1. `NotificationRecord.status` → `dlq`.
2. notification-service publishes `notification.failed` event via Outbox.
3. `notification.failed` routes to `ops-service-notification-failed-queue`.
4. ops-service surfaces it in the DLQ console.
5. Grafana `ALT003` fires within 5 minutes.

### Operator Resolution

The Platform Operator (P4) opens the ops console and views the `notification.failed` entry:
- **Re-trigger**: ops-service re-queues the triggering event. The InboxRecord for the original `event_id` is reset (or a new `notification_id` is issued).
- **Discard**: operator discards with a documented reason. AuditLog entry written (BR014).

**Critical invariant (BR007)**: notification failures are never visible to the booking operation. A booking that fails to send a confirmation email is still a fully confirmed booking.
