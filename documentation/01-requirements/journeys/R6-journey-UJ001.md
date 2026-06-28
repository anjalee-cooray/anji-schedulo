---
title: Journey — Tenant Onboarding
journeyId: UJ001
persona: P1
priority: critical
layer: 01-requirements
status: current
lastUpdated: 2026-06-27
---

# R6 · Journey UJ001 — Tenant Onboarding

## Metadata

| Field | Value |
|---|---|
| Journey ID | UJ001 |
| Name | Tenant Onboarding |
| Primary Persona | [P1 — Tenant Admin](../personas/R4a-persona-P1.md) |
| Priority | Critical |
| Related Functional Requirements | FR001, FR002 |
| Related Business Rules | BR005, BR013, BR014 |
| Architecture Pattern | Pub/Sub Fan-out, Transactional Outbox |
| Services Involved | tenant-service, notification-service, billing-service |

---

## Goal

A new business owner registers their organisation on AnjiSchedulo and completes all configuration steps necessary to accept live customer bookings — without needing any technical assistance or operator intervention.

---

## Preconditions

- The business owner has a valid email address.
- The business owner has selected a pricing plan (Starter, Pro, or Enterprise).
- No existing AnjiSchedulo account exists for this email address.
- The platform is available and accepting new registrations.

---

## Step-by-Step Journey

### Step 1 — Submit Registration

The business owner navigates to the AnjiSchedulo sign-up page and completes the registration form with:
- Business name
- Owner email address
- Password
- Selected pricing plan

The owner submits the form.

**Outcome:** The platform acknowledges the submission and displays a "We're setting up your account" screen.

---

### Step 2 — Receive Welcome Email

Within 30 seconds of registration, the owner receives a welcome email at the address provided. The email contains:
- A confirmation that the account has been created.
- A direct link to begin account setup.
- The name of the selected plan and what it includes.

**Outcome:** Owner has a verified entry point into the platform setup flow.

---

### Step 3 — Configure Business Hours

The owner logs in via the setup link and is guided to the business hours configuration screen. They set the days and operating hours for their business — for example, Monday to Friday, 9 am to 6 pm.

Business hours can be set at the business level (applied to all staff) or per individual staff member.

**Outcome:** The platform records the working hours. These hours will be used to determine which appointment slots are shown to customers.

---

### Step 4 — Build the Service Catalogue

The owner navigates to the Services section and adds the services their business offers. For each service they provide:
- Service name (e.g. "Haircut", "Consultation")
- Duration in minutes
- Price (can be zero for no-charge services)
- Whether the service requires payment at booking or in person

The owner can add multiple services and deactivate any that are not currently offered.

**Outcome:** The service catalogue is live and will appear on the public booking page.

---

### Step 5 — Add Staff Members

The owner navigates to the Staff section and adds the staff members who will be visible for bookings. For each staff member they provide:
- Full name
- Email address (used for booking notifications)
- The services they are qualified to deliver
- Their individual working hours (if different from business hours)

**Outcome:** Staff are added to the roster. Customers will be able to select from available staff when booking.

---

### Step 6 — Go Live

Once business hours, at least one service, and at least one staff member are configured, the platform marks the tenant as active and the public booking page becomes visible and bookable.

The owner is shown the public URL for their booking page, which they can share with customers or embed in their website.

**Outcome:** The tenant's booking page is live. Customers can browse slots and book appointments.

---

## Success Outcome

The tenant's booking page is live and a first customer appointment can be booked within 30 minutes of the owner submitting the registration form. All provisioning steps have completed without any operator intervention. The owner has received their welcome email and the platform has recorded their business hours, services, and staff.

---

## Failure Paths

### Registration Email Already in Use

If the submitted email address is already associated with an AnjiSchedulo account, the registration form returns a validation error before submission. The owner is prompted to log in or use a different email address. No provisioning occurs.

### Welcome Email Not Delivered

If the welcome email fails to deliver (e.g. due to a provider outage), the failure is routed to the notification Dead Letter Queue after 4 retry attempts. This does not block the account from being active or the owner from logging in. The platform operator is alerted and can trigger a manual resend from the ops console.

### Billing Activation Failure

If the billing subscription cannot be activated at registration time (e.g. Stripe is unavailable), the tenant status remains in a `provisioning` state. The platform operator receives an alert. The owner cannot access the booking page until billing confirms. This is an exceptional failure path; normal registration completes synchronously with Stripe.

### Incomplete Configuration — Page Not Yet Live

If the owner saves their registration but does not yet complete business hours, services, or staff setup, the booking page remains unpublished. The dashboard displays a setup checklist showing which steps remain. The owner can return to complete setup at any time.

---

## Related Business Rules

| Rule ID | Rule Summary | Application in This Journey |
|---|---|---|
| BR005 | A tenant's data must never be readable by another tenant. | All tenant configuration data written during onboarding is scoped to the tenant's `tenant_id`. PostgreSQL RLS policies prevent any cross-tenant reads from the moment the record is created. |
| BR013 | Every event must carry `tenant_id`, `event_id`, `correlation_id`, and `occurred_at`. | The `tenant.provisioned` event emitted at the end of Step 1 must include all four envelope fields. Any event missing these fields is routed to the DLQ rather than processed. |
| BR014 | All write operations that modify tenant-scoped data must produce an immutable audit log entry. | Every configuration change in Steps 3–5 (business hours, services, staff) produces an `audit_log` record. This satisfies GDPR forensic requirements. |

---

## Implementation Note

When the owner submits the registration form, `tenant-service` writes a new `Tenant` record and an `OutboxRecord` for `tenant.provisioned` in the same database transaction (Transactional Outbox pattern). The `outbox-relay` polls every 500 milliseconds and publishes `tenant.provisioned` to the SNS topic, which fans out to independent SQS queue subscribers: `notification-service` sends the welcome email, and `billing-service` activates the Stripe subscription. These steps are eventually consistent and independent — a billing delay does not prevent the welcome email from sending, and vice versa. Tenant status transitions to `active` only after `billing.subscription_activated` is received by `tenant-service`, completing the provisioning lifecycle.
