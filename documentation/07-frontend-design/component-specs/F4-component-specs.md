---
title: Component Specifications
layer: 07-frontend-design
status: current
lastUpdated: 2026-06-28
---

# F4 · Component Specifications

Eight key shared components used across the AnjiSchedulo web and mobile interfaces. Each spec defines props, states, visual behaviour, and accessibility requirements. Design token references align with `F2-design-tokens.md`.

---

## 1. ServiceCard

Used in the appointment booking flow (UJ002 Step 1) and the tenant services management screen.

### Props

| Prop | Type | Required | Description |
|---|---|---|---|
| `service_id` | string (UUID) | yes | Unique identifier for the service |
| `name` | string | yes | Display name of the service |
| `duration_minutes` | number | yes | Duration of the appointment in minutes |
| `price_amount` | number | yes | Price in smallest currency unit (pence/cents) |
| `currency` | string | yes | ISO 4217 currency code (e.g. "GBP") |
| `selected` | boolean | yes | Whether this card is the currently selected option |
| `onClick` | function | yes | Called when the card is tapped/clicked |
| `disabled` | boolean | no | Default `false`. Set `true` if the service is inactive or unavailable. |
| `description` | string | no | Optional short description (max 80 chars, truncated with ellipsis) |

### States

| State | Visual behaviour |
|---|---|
| **Default** | White background, 2px border `--color-border-default`, shadow Level 1 |
| **Hover** | Border `--color-brand-300`, shadow Level 2, transition 150ms ease |
| **Selected** | Border `--color-brand-500` (2px), background `--color-brand-50` (#eff6ff), no additional shadow |
| **Disabled** | Opacity 0.4, cursor `not-allowed`, no hover state, no `onClick` fired |

### Visual Structure

```
┌──────────────────────────────────────┐
│  [●]  Service name            [○/●]  │
│  45 min                     £40.00   │
│  Optional short description...       │
└──────────────────────────────────────┘
```

- Service icon: 32×32px placeholder circle (`--color-surface-subtle`) in top-left
- Name: Inter 600 16px `--color-text-primary`
- Duration badge: inline pill `--color-surface-subtle` background, 12px `--color-text-secondary`
- Price: Inter 700 20px `--color-text-primary`, right-aligned
- Description: 14px `--color-text-secondary`, max 2 lines

### Accessibility

- Rendered as a `div` with `role="radio"` and `aria-checked={selected}`
- Group of service cards wrapped in a `div` with `role="radiogroup"` and `aria-label="Select a service"`
- `aria-disabled={disabled}`
- `aria-label="{name}, {duration_minutes} minutes, {price formatted}. {selected ? 'Selected.' : ''}"`
- Keyboard: focusable, Enter/Space fires `onClick`. Arrow keys move focus within the radiogroup.
- Disabled cards: `tabindex="-1"`, not reachable by keyboard

---

## 2. StaffSelector

Used in UJ002 Step 2 — staff member selection within the booking flow.

### Props

| Prop | Type | Required | Description |
|---|---|---|---|
| `staff` | StaffOption[] | yes | Array of staff available for the selected service |
| `selectedStaffId` | string \| null | yes | Currently selected staff_id, or `null` for "any available" |
| `onSelect` | function(staff_id: string \| null) | yes | Called with staff_id or null when selection changes |
| `showAnyOption` | boolean | yes | Whether to show the "Any available" option first |
| `loading` | boolean | no | Default `false`. Shows skeleton when `true`. |

**StaffOption shape:** `{ staff_id, name, email, services: string[], available: boolean, photo_url?: string }`

### States

| State | Visual behaviour |
|---|---|
| **Loading** | Three skeleton rows, 72px tall each, shimmer animation |
| **Default** | List of staff cards, no selection |
| **Selected** | Selected card has `--color-brand-50` background, `--color-brand-500` border, radio filled |
| **No staff available** | Empty state: person icon + "No staff available for this service and date" 14px |

### Visual Structure

Each option is a full-width row card:
- Avatar: 48×48px circle. Photo if available; otherwise initials (first letter of first + last name) in Inter 600 18px white on `--color-brand-500` background.
- "Any available" option: shows generic group icon avatar. Label "Anyone available" Inter 500 16px. Sub-label "We'll assign the first free team member" 13px `--color-text-secondary`.
- Unavailable staff: opacity 0.4, shows "Unavailable for [service]" sub-label, `cursor: not-allowed`, no `onSelect` fired.

### Accessibility

- Container: `role="listbox"` with `aria-label="Choose a staff member"`
- Each option: `role="option"`, `aria-selected={selected}`, `aria-disabled={!available}`
- `aria-label="{name}. {available ? 'Available' : 'Unavailable for ' + serviceName}"`
- Keyboard: arrow keys move focus between options, Enter/Space selects

---

## 3. TimeSlotPicker

Used in UJ002 Step 3 — choosing a specific appointment time for a selected date.

### Props

| Prop | Type | Required | Description |
|---|---|---|---|
| `date` | string (ISO 8601 date) | yes | The date for which slots are shown |
| `availableSlots` | string[] | yes | Array of available times, e.g. `["09:00", "09:45", "10:30"]` |
| `unavailableSlots` | string[] | no | Times that exist in the schedule but are already booked |
| `selectedSlot` | string \| null | yes | Currently selected time or `null` |
| `onSelect` | function(time: string) | yes | Called when an available slot is tapped |
| `loading` | boolean | no | Default `false`. Shows skeleton when `true`. |

### States

| State | Visual behaviour |
|---|---|
| **Loading** | Grid of 8 skeleton pills, shimmer animation |
| **Loaded** | Pills rendered per slot |
| **No slots available** | Empty state: clock icon + "No available slots on this date. Try another day." 14px `--color-text-secondary` centred |

### Visual Structure

- 2-column pill grid, gap 8px
- Pill: 100% column width, 44px height, border 1.5px solid `--color-border-default`, border-radius 9999px, 14px Inter 500 centred
- Available pill hover: border `--color-brand-400`, background `--color-brand-50`, transition 100ms
- Selected pill: background `--color-brand-500`, white text, no border
- Unavailable pill (already booked): text with CSS `text-decoration: line-through`, `--color-text-disabled`, `cursor: not-allowed`, `aria-disabled`

### Accessibility

- Container: `role="grid"` with `aria-label="Available times for {date formatted}"`
- Each pill: `role="gridcell"` containing a `button`. `aria-pressed={selected}`. `aria-disabled={unavailable}`.
- `aria-label="{time}. {unavailable ? 'Unavailable.' : selected ? 'Selected.' : 'Available.'}"`
- Keyboard: arrow keys navigate grid cells, Enter/Space selects available slot, Tab moves focus out of grid

---

## 4. BookingConfirmationCard

Displayed on the booking confirmation screen (UJ002 Step 5) and in the "My Appointments" detail view.

### Props

| Prop | Type | Required | Description |
|---|---|---|---|
| `appointment_id` | string | yes | Used to generate the formatted reference number |
| `service_name` | string | yes | Name of the booked service |
| `staff_name` | string | yes | Name of the assigned staff member |
| `slot_date` | string (ISO 8601) | yes | Date of the appointment |
| `slot_start` | string (HH:MM) | yes | Start time |
| `slot_end` | string (HH:MM) | yes | End time |
| `amount_charged` | number \| null | no | Amount charged in smallest currency unit. `null` if no payment. |
| `currency` | string \| null | no | ISO 4217 code. `null` if no payment. |
| `addToCalendarLinks` | object | no | `{ google: string, apple: string, outlook: string }`. If omitted, add-to-calendar section is hidden. |

### Visual Structure

- White card, border-left 4px `--color-brand-500`, shadow Level 1, border-radius 12px, padding 24px
- Rows: label (`--color-text-secondary`, 13px, Inter 500) above value (`--color-text-primary`, 16px, Inter 500)
- Rows shown: Service / Staff / Date / Time / Paid (if amount_charged) / Reference
- Date formatted as "Thursday, 4 June 2026"; time as "10:30 – 11:15"
- Reference: monospace font, light grey background chip
- Add to calendar: 3 icon-text buttons in a row below the card rows, 14px, icons 16px, border 1px `--color-border-default`

### Accessibility

- Card is a `section` with `aria-label="Your appointment details"`
- Each row: `dt` + `dd` inside a `dl` (definition list)
- Add to calendar links: `<a>` with `aria-label="Add to Google Calendar"` etc., opens in new tab with `rel="noopener noreferrer"`
- No interactive states — display only (except add-to-calendar links)

---

## 5. DashboardMetricCard

Displayed on the Tenant Admin dashboard (UJ005) in a 4-card grid.

### Props

| Prop | Type | Required | Description |
|---|---|---|---|
| `label` | string | yes | Metric label, e.g. "Confirmed Bookings" |
| `value` | string \| number | yes | Display value, e.g. "12" or "£1,240" |
| `unit` | string | no | Optional unit appended to value, e.g. "%" or "bookings" |
| `trend` | `'up' \| 'down' \| 'neutral'` | yes | Direction of change since previous period |
| `trend_percent` | number | yes | Magnitude of change as a percentage |
| `trendPositive` | `'up' \| 'down'` | yes | Which direction is good for this metric. `'up'` = higher is better. `'down'` = lower is better (for cancellation_rate). |
| `loading` | boolean | no | Default `false`. Shows skeleton when `true`. |
| `error` | boolean | no | Default `false`. Shows error state with retry. |

### States

| State | Visual behaviour |
|---|---|
| **Loading** | Full card skeleton with shimmer, same card dimensions |
| **Loaded** | Label + value + trend indicator |
| **Error** | Replaces value with "—" and shows "Unable to load" 12px below |

### Visual Structure

- White card, shadow Level 1, border-radius 12px, padding 20px
- Label: 13px Inter 500 `--color-text-secondary`, uppercase, letter-spacing 0.5px
- Value: Inter 700 36px `--color-text-primary`, 4px margin-top
- Trend indicator (below value): small row with arrow icon (16px) + percent text (14px Inter 500). Positive: `--color-success-600`. Negative: `--color-danger-600`. Neutral: `--color-text-secondary`.
- Positive/negative is determined by comparing `trend` against `trendPositive`:
  - `trend === 'up'` and `trendPositive === 'up'` → green ↑
  - `trend === 'down'` and `trendPositive === 'down'` → green ↓
  - `trend === 'up'` and `trendPositive === 'down'` → red ↑
  - `trend === 'down'` and `trendPositive === 'up'` → red ↓
  - `trend === 'neutral'` → grey →

### Accessibility

- Card: `role="region"` with `aria-label="{label}"`
- Value: `aria-live="polite"` so screen readers announce value changes when dashboard refreshes
- `aria-label` for full card: `"{label}: {value}{unit}. {trend_percent}% {trend} since yesterday. {positive/negative description}."`

---

## 6. AppointmentListItem

Used in "My Appointments" (customer view, UJ003 Step 1) and the admin appointment list.

### Props

| Prop | Type | Required | Description |
|---|---|---|---|
| `appointment_id` | string | yes | Appointment ID |
| `service_name` | string | yes | Name of the service |
| `staff_name` | string | yes | Name of the staff member |
| `slot_date` | string (ISO 8601) | yes | Appointment date |
| `slot_start` | string (HH:MM) | yes | Start time |
| `status` | `'confirmed' \| 'cancelled' \| 'rescheduled' \| 'completed'` | yes | Current status |
| `within_cancellation_window` | boolean | yes | Whether the cancellation is still allowed (BR002) |
| `onCancel` | function | no | Called when Cancel is clicked. Only rendered if `status === 'confirmed'` and `within_cancellation_window === true`. |
| `onReschedule` | function | no | Called when Reschedule is clicked. Only rendered if `status === 'confirmed'`. |

### States

| State | Visual behaviour |
|---|---|
| **Confirmed** | Full opacity, status chip green, action buttons visible |
| **Cancelled** | Opacity 0.6, status chip grey, no action buttons, strikethrough on time |
| **Completed** | Opacity 0.7, status chip muted, no action buttons |
| **Rescheduled** | Full opacity, status chip blue, Reschedule action visible (allows rescheduling again) |

### Visual Structure

- Full-width row, 80px min-height, padding 16px, border-bottom 1px `--color-border-default`
- Left: date/time column — date 14px Inter 600 `--color-text-primary`, time 13px `--color-text-secondary`
- Centre: service badge (13px pill, `--color-brand-50`/`--color-brand-700`) + staff name 14px
- Right: status chip + action icon buttons (cancel: ×, reschedule: ↺). Action buttons only appear on row hover (desktop) or always visible (mobile).
- Status chip: rounded-full pill, 12px Inter 500. Colours by status:
  - Confirmed: `--color-success-50` / `--color-success-700`
  - Cancelled: `--color-surface-subtle` / `--color-text-disabled`
  - Completed: `--color-surface-subtle` / `--color-text-secondary`
  - Rescheduled: `--color-brand-50` / `--color-brand-700`

### Accessibility

- Row: `role="listitem"` within a `role="list"`
- Status chip: `aria-label="Status: Confirmed"` (not colour-only)
- Cancel button: `aria-label="Cancel appointment: {service_name} on {date formatted} at {time}"`
- Reschedule button: `aria-label="Reschedule appointment: {service_name} on {date formatted}"`
- Cancelled items: `aria-label` on row includes "Cancelled" in the description

---

## 7. AlertBanner

Displayed on the Tenant Admin dashboard when an anomaly is detected (FR009 — velocity spike or high cancellation rate).

### Props

| Prop | Type | Required | Description |
|---|---|---|---|
| `type` | `'velocity_spike' \| 'high_cancellation_rate'` | yes | Determines icon, title, and default message |
| `message` | string | yes | Human-readable alert body |
| `timestamp` | string (ISO 8601) | yes | When the alert was triggered |
| `onDismiss` | function | yes | Called when the × button is clicked or auto-dismiss fires |

### States

- Only one state: visible. The banner is conditionally rendered by the parent; it does not animate visibility on render in reduced-motion contexts.

### Visual Structure

- Full-width bar below the page title (above metric cards)
- Background `--color-warning-50` (#fffbeb)
- Border: 1px solid `--color-warning-200`, border-left 4px solid `--color-warning-500`
- Border-radius 8px, padding 12px 16px
- Left: ⚠ icon (`--color-warning-600`, 20px) + alert title (Inter 600 14px `--color-text-primary`) + message body (14px `--color-text-secondary`)
- Right: timestamp (12px `--color-text-secondary`) + × dismiss button (44×44px tap target, `--color-text-secondary`)
- Auto-dismiss: 10 minutes after `timestamp` if `onDismiss` has not been called. Timer is reset if user interacts with the banner (clicks, hovers).

### Accessibility

- `role="alert"` with `aria-live="polite"` — announced by screen readers on mount
- Alert title and message combined in a single announcement: `aria-label="{type title}: {message}. Detected at {timestamp formatted}."`
- Dismiss button: `aria-label="Dismiss {type title} alert"`
- Auto-dismiss does not remove focus from the dismiss button if it has focus at dismiss time — focus returns to the previously focused element

---

## 8. CancellationModal

Triggered when the user clicks "Cancel" on an `AppointmentListItem` (UJ003 Step 2).

### Props

| Prop | Type | Required | Description |
|---|---|---|---|
| `appointment` | AppointmentSummary | yes | Object with `service_name`, `staff_name`, `slot_date`, `slot_start`, `slot_end`, `amount_charged`, `currency` |
| `policy_hours` | number | yes | The tenant's cancellation policy window in hours (from BR002 / TenantConfig) |
| `onConfirm` | function | yes | Called when the user confirms cancellation |
| `onDismiss` | function | yes | Called when the user dismisses (× button, "Keep appointment", or Escape key) |

**AppointmentSummary shape:** `{ appointment_id, service_name, staff_name, slot_date, slot_start, slot_end, amount_charged?: number, currency?: string }`

### States

| State | Visual behaviour |
|---|---|
| **Open** | Modal visible, content rendered, confirm/dismiss buttons active |
| **Confirming** | Confirm button shows spinner, disabled. Dismiss button remains active. |
| **Error** | Error message shown below confirm button: "Cancellation failed. Please try again." `--color-danger-600` 13px. Confirm button re-enabled. |

### Visual Structure

**Desktop:** centred modal overlay
- Backdrop: `rgba(0, 0, 0, 0.48)` fixed overlay
- Modal: white, border-radius 16px, shadow Level 3, max-width 480px, padding 24px

**Mobile:** bottom sheet (slides up from bottom)
- Same backdrop
- Sheet: white, border-radius 16px 16px 0 0, padding 24px, safe-area bottom

**Content:**
- Header: "Cancel appointment?" Inter 700 20px + × close button (44×44px) top-right
- Divider: 1px `--color-border-default`
- Appointment summary: service name + staff + date + time. 16px `--color-text-primary`.
- Policy notice: amber card (`--color-warning-50`, border `--color-warning-200`, border-left 4px `--color-warning-500`)
  - If within window: "Cancellation policy: [X] hours notice required. You are within the allowed window."
  - If refund applicable: "A refund of [amount] will be issued to your original payment method." `--color-success-600`
- CTA row:
  - "Yes, cancel it" button: full-width (mobile) or right-aligned (desktop). Background `--color-danger-500`, white Inter 600 16px, border-radius 8px, 52px height.
  - "Keep my appointment" button: full-width (mobile) or left-aligned (desktop). Outlined `--color-border-default` border, `--color-text-primary` Inter 500 16px.

### Accessibility

- `role="dialog"` with `aria-modal="true"` and `aria-labelledby` pointing to the heading
- Focus trap: on open, focus moves to the first focusable element in the modal (× close or "Yes, cancel" button). Tab cycles within the modal only. Escape fires `onDismiss`.
- On dismiss: focus returns to the "Cancel" button that triggered the modal
- Confirming state: `aria-busy="true"` on the form/dialog element; confirm button `aria-disabled="true"` during loading
- Error state: error message in a `role="alert"` container so it is announced immediately
