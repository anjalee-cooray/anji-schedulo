---
title: UI/UX Specification
layer: 02-design
status: current
lastUpdated: 2026-06-28
---

# D10 · UI/UX Specification

## 1. Design Principles

All AnjiSchedulo interfaces are guided by four principles derived from the persona research (P1–P3):

| Principle | Definition | Source |
|---|---|---|
| **Scannable at a glance** | Critical information (today's bookings, revenue, cancellation rate) must be visible without scrolling on a 1280px desktop or a 390px mobile screen. | P1 opens the dashboard every morning before the first customer arrives. |
| **Zero training required** | Every action a Tenant Admin (P1) or Staff Member (P2) must take must be self-evident from the UI. Error states explain what to do, not just what went wrong. | P1 is not technical. Setup must complete in under 30 minutes. |
| **Interruption-tolerant** | End Customers (P3) booking on mobile may lose focus mid-flow. The booking form preserves state across tab switches and back-navigation. | P3 books during commutes on mobile. |
| **Trust signals at every transaction** | Payment forms show provider branding (Stripe), lock icons, and explicit confirmation copy. Cancellation flows show policy and refund amounts before the final action. | P3 needs to see what they are agreeing to before paying. |

---

## 2. Accessibility Requirements

Derived from NFR013 (WCAG 2.1 AA compliance, all public and authenticated screens).

| Requirement | Standard | Implementation |
|---|---|---|
| Colour contrast (text) | AA: 4.5:1 minimum | Applied to all body text and interactive labels |
| Colour contrast (large text / UI components) | AA: 3:1 minimum | Applied to headings, button borders, focus rings |
| Keyboard navigation | Full tab order on all interactive elements | No mouse-only interactions in any flow |
| Screen reader support | All images have `alt` text; all form inputs have associated `<label>` | ARIA roles applied to custom components (calendar, slot picker) |
| Focus management | Focus moves to the first relevant element after page navigation or modal open | Applied to all route changes and all modal dialogs |
| Error identification | Errors are identified by text, not colour alone | Error states: icon + colour + descriptive text |
| Motion | Animations ≤ 300ms; `prefers-reduced-motion` media query disables transitions | Applied to all CSS transitions |
| Touch targets | Minimum 44×44px for all interactive elements on mobile | Applied to calendar cells, slot buttons, navigation items |

---

## 3. Design Tokens

Design tokens are the single source of truth for all visual decisions. Full token definitions are in `documentation/07-frontend-design/F2-design-tokens.md`. Key tokens referenced throughout this spec:

| Token | Value | Usage |
|---|---|---|
| `color-primary-600` | `#1E6FD9` | Primary CTA buttons, active state indicators |
| `color-primary-50` | `#EBF3FF` | Hover backgrounds, selected slot backgrounds |
| `color-success-600` | `#16A34A` | Confirmed status badges, success messages |
| `color-warning-600` | `#D97706` | Pending status badges, warning banners |
| `color-error-600` | `#DC2626` | Error states, cancelled status badges |
| `color-neutral-900` | `#111827` | Primary text |
| `color-neutral-500` | `#6B7280` | Secondary/muted text |
| `color-neutral-100` | `#F3F4F6` | Page backgrounds |
| `font-sans` | `Inter, system-ui, sans-serif` | All body and UI text |
| `font-mono` | `JetBrains Mono, monospace` | Booking reference IDs, timestamps |
| `space-4` | `16px` | Base spacing unit |
| `radius-lg` | `12px` | Card border radius |
| `shadow-md` | `0 4px 6px -1px rgba(0,0,0,0.07)` | Card and modal shadows |

---

## 4. Screen Inventory

### 4.1 Unauthenticated / Public Screens

| Screen | Route | Persona | Journey |
|---|---|---|---|
| Landing / Marketing | `/` | All | — |
| Registration | `/register` | P1 | UJ001 |
| Login | `/login` | P1, P2 | — |
| Tenant Booking Page | `/{tenant-slug}/book` | P3 | UJ002 |
| Booking Confirmation | `/{tenant-slug}/book/confirmation` | P3 | UJ002 |
| Booking Cancellation | `/{tenant-slug}/cancel/{token}` | P3 | UJ003 |
| Cancellation Confirmation | `/{tenant-slug}/cancel/confirmed` | P3 | UJ003 |

### 4.2 Tenant Admin Screens (P1)

| Screen | Route | Journey |
|---|---|---|
| Onboarding Checklist | `/dashboard/onboarding` | UJ001 |
| Dashboard — Today's Overview | `/dashboard` | UJ005 |
| Appointments List | `/dashboard/appointments` | UJ005 |
| Appointment Detail | `/dashboard/appointments/{id}` | UJ005 |
| Services Configuration | `/dashboard/services` | UJ001 |
| Staff Configuration | `/dashboard/staff` | UJ001 |
| Business Hours Configuration | `/dashboard/hours` | UJ001 |
| Analytics | `/dashboard/analytics` | UJ005 |
| Data Export | `/dashboard/export` | UJ007 |
| Account Settings | `/dashboard/settings` | — |

### 4.3 Staff Member Screens (P2)

| Screen | Route | Journey |
|---|---|---|
| My Schedule | `/staff/schedule` | — |
| Availability Blocks | `/staff/availability` | — |

### 4.4 Platform Operator Screens (P4)

| Screen | Route | Journey |
|---|---|---|
| Ops Console — DLQ | `/ops/dlq` | UJ006 |
| Ops Console — Tenants | `/ops/tenants` | UJ007 |
| Ops Console — Replay | `/ops/replay` | — |
| Ops Console — Audit Log | `/ops/audit` | — |

---

## 5. Key Screen Specifications

### 5.1 Registration Screen (`/register`)

**Purpose:** P1 signs up for a new tenant account.

**Layout:** Single-column centred card, max-width 480px, vertically centred on page.

**Form fields:**
| Field | Type | Validation |
|---|---|---|
| Business name | Text | Required, 2–100 chars |
| Owner email | Email | Required, valid email, uniqueness checked on blur |
| Password | Password | Required, 12+ chars, show/hide toggle |
| Pricing plan | Radio group | Required; Starter (default), Pro, Enterprise |

**Plan radio group** shows price and 3 key feature bullets per plan. Enterprise shows "Contact us" with a mailto link — no self-serve Enterprise signup in v1.

**Submit button** copy: "Create my account" — disabled until all required fields pass client-side validation.

**Post-submit state:** Button transitions to a spinner. Form is disabled. A progress banner appears: "We're setting up your account…" After the `tenant.provisioned` event is confirmed (poll `GET /api/v1/tenants/{id}/status`), the page transitions to the onboarding checklist.

**Error states:**
- Email already registered: inline field error "This email is already linked to an account. Log in instead."
- Server error: toast: "Something went wrong. Please try again in a moment."

---

### 5.2 Onboarding Checklist (`/dashboard/onboarding`)

**Purpose:** Guides P1 through the four configuration steps required before the booking page goes live.

**Layout:** Left sidebar (step progress indicator) + right content area. On mobile: linear scrolling with sticky step counter.

**Steps:**
1. Business hours (FR002) — select open/closed per day + time range per open day
2. Add first service (FR002) — name, duration, price
3. Add first staff member (FR002) — name, email, assigned services
4. Go live — review and confirm public booking URL

Each step shows a large check icon when complete. The "Go live" step is disabled until steps 1–3 are complete. The onboarding checklist is dismissed automatically when the tenant status transitions to `active` and at least one booking is received.

---

### 5.3 Dashboard — Today's Overview (`/dashboard`)

**Purpose:** P1 sees today's confirmed bookings, net revenue, cancellation rate, and peak hour at a glance (UJ005, FR008).

**Layout:** 4-stat summary bar (full width) + appointments timeline (left, 60% width) + activity sidebar (right, 40%).

**Stat bar — 4 cards:**
| Stat | Value | Source |
|---|---|---|
| Confirmed bookings today | Count | TenantDashboardView |
| Net revenue today | Currency | TenantDashboardView |
| Cancellation rate (7d) | Percentage | TenantDashboardView |
| Peak hour this week | Time range | TenantDashboardView |

All stats load from the `dashboard-service` via `GET /api/v1/dashboard/{tenant_id}/today`. Target: data visible within 200ms (NFR010). Stats refresh automatically every 30 seconds via polling.

**Appointments timeline:** Chronological list of today's confirmed appointments. Each row: staff avatar (initials), customer name, service name, time slot, status badge (confirmed/pending/cancelled), action button ("View detail"). Sorted ascending by `slot_start`. Empty state: "No bookings yet today — share your booking link to get started."

**Activity sidebar:** Last 10 events from the appointment event log for this tenant. Each entry: event type icon, description text, relative timestamp (e.g. "2 minutes ago"). Auto-refreshes every 30 seconds.

**Alert banner (conditional):** If `cancellation_rate_7d > 20%` or `booking_velocity_spike = true`, a dismissible amber banner appears at the top of the content area with a plain-English description and a "View analytics" CTA.

---

### 5.4 Tenant Booking Page (`/{tenant-slug}/book`)

**Purpose:** P3 selects a service, staff member, and time slot, then books and pays (UJ002, FR003–FR007).

**Layout:** Two-column on desktop (service + staff selection left, calendar + slot picker right); single-column stack on mobile.

**Step 1 — Service selection:**
Card grid of available services. Each card: service name, duration badge, price badge. Clicking a service card selects it and advances to step 2. Only active services shown (FR002).

**Step 2 — Staff selection:**
Card grid of staff members assigned to the selected service. Each card: staff avatar (initials if no photo), name, "Any available" option first. Selecting advances to step 3.

**Step 3 — Date and time slot selection:**
Calendar component (month view): days in the past are greyed. Days with no available slots are greyed. Selecting a date loads available time slots for that staff member and service (GET `/api/v1/availability/{tenant_id}?service_id=...&staff_id=...&date=...`). Slots displayed as pill buttons: time + duration. Slots are 30-second soft-locked (SlotLock) from the moment the customer selects them — no other customer can confirm that slot during checkout (BR001).

**Step 4 — Customer details:**
| Field | Type | Validation |
|---|---|---|
| Full name | Text | Required |
| Email address | Email | Required, valid format |
| Phone number | Tel | Optional (required if Pro/Enterprise for SMS) |
| Notes | Textarea | Optional, 500 chars max |

**Step 5 — Payment (if service price > 0):**
Stripe Elements embedded card form. PCI-compliant — card data never touches AnjiSchedulo servers. Form shows: service name, duration, amount to charge, currency, Stripe logo, and lock icon. "Book and pay" submit button.

**Confirmation state:** Replaces the booking form with a confirmation card: booking reference (`appointment_id` short format), service details, date, time, staff name, amount charged, cancellation policy ("You can cancel up to [N] hours before your appointment for a full refund."), "Add to calendar" link (Google, iCal).

---

### 5.5 DLQ Triage Console (`/ops/dlq`)

**Purpose:** P4 (Platform Operator) triages and resolves DLQ events (UJ006, FR010).

**Layout:** Table (full width) with filter sidebar.

**Table columns:** Queue name | Event type | Tenant ID | Failed at | Failure reason | Actions

**Filters:** Queue name dropdown, event type dropdown, tenant ID search, date range.

**Row actions:**
- "View payload" — opens a right-side drawer with the full JSON event payload, formatted with syntax highlighting.
- "Re-queue" — confirms with a modal ("Re-queue this event to [source_queue]?"). After confirmation, ops-service re-publishes to the source SQS FIFO queue.
- "Discard" — opens a modal with a required "Reason" text field. After confirmation, ops-service writes an AuditLog entry (BR014) and marks the DLQ record as discarded.

**Empty state:** "All clear — no DLQ events. The system is healthy."

---

## 6. Component Library

All components are defined in detail in `documentation/07-frontend-design/F4-component-specs.md`. Key components:

| Component | Used In | Notes |
|---|---|---|
| `StatCard` | Dashboard | 4 variants: bookings, revenue, rate, time |
| `AppointmentRow` | Dashboard, appointments list | Status badge, action menu |
| `CalendarPicker` | Booking page | Slot availability colour-coded |
| `SlotButton` | Booking page | Selected, available, locked states |
| `ServiceCard` | Booking page, services admin | Price badge, duration badge |
| `StaffCard` | Booking page, staff admin | Avatar, name, assigned services |
| `StatusBadge` | All table rows | Confirmed (green), Pending (amber), Cancelled (red), DLQ (dark red) |
| `PaymentForm` | Booking page step 5 | Stripe Elements wrapper |
| `DLQRow` | Ops console | View payload, re-queue, discard actions |
| `AuditEntry` | Activity sidebar | Event type icon, description, timestamp |

---

## 7. Responsive Breakpoints

Derived from `specs/ai/infrastructure.json` (Amplify + CloudFront frontend) and NFR013 (mobile-first).

| Breakpoint | Width | Layout Change |
|---|---|---|
| Mobile (default) | < 768px | Single column; stacked navigation; bottom tab bar |
| Tablet | 768px–1279px | Sidebar navigation collapsed to icons; content 100% width |
| Desktop | ≥ 1280px | Full sidebar navigation; two-column layouts on dashboard and booking page |

---

## 8. Navigation Structure

### Tenant Admin (P1)

```
Top bar: [AnjiSchedulo logo] [Tenant name] [Notifications bell] [Profile avatar]

Left sidebar (desktop) / Bottom tab bar (mobile):
  - Dashboard (home icon)
  - Appointments (calendar icon)
  - Services (tag icon)
  - Staff (people icon)
  - Analytics (chart icon)
  - Settings (gear icon)
  - Export Data (download icon)
```

### End Customer (P3) — Booking Flow

No persistent navigation. The booking page is a standalone multi-step form. Progress indicator at the top: 5 steps with filled/unfilled dots. "Back" link appears on steps 2–5.

### Platform Operator (P4)

```
Top bar: [AnjiSchedulo OPS] [Environment badge: prod/staging] [Profile]

Left sidebar:
  - DLQ Console (alert icon)
  - Tenants (building icon)
  - Replay (refresh icon)
  - Audit Log (list icon)
```

---

## 9. Loading States

All data-fetching screens show skeleton loaders, not spinners. Skeleton loaders mirror the shape of the actual content (card skeletons, row skeletons, text line skeletons). The slot picker on the booking page shows a row of grey pill skeletons while `GET /availability` is in-flight. Target: perceived performance — skeleton to content transition ≤ 200ms on cached responses (NFR010).

---

## 10. Error States

| Error Type | Trigger | UI Treatment |
|---|---|---|
| Slot no longer available | Slot confirmed by another customer between selection and checkout | Full-page message with "Choose another time" CTA. Slot lock released immediately. |
| Payment declined | Stripe returns card decline | Inline error below the payment form. Card details cleared. Retry without re-entering customer details. |
| Cancellation window closed | Customer attempts to cancel within `cancellation_hours` of slot start | Full-page message: "The cancellation window for this appointment has passed. Please contact [business name] directly." No refund CTA shown. |
| Session expired | JWT expired during booking flow | Modal: "Your session has expired. Please start again." Booking state cleared. |
| Service unavailable | API returns 503 | Toast with retry button. Booking state preserved in `sessionStorage` for 30 seconds. |
