---
title: Hi-fi Design Spec — Web
layer: 07-frontend-design
status: current
lastUpdated: 2026-06-28
---

# F3 · Hi-fi Design Specification — Web

This document defines the intended visual design for the AnjiSchedulo web application. It is a written specification for developers and designers — not a visual mockup. All design decisions are derived from the design tokens defined in `F2-design-tokens.md` and the personas and journeys in the 01-requirements layer.

---

## 1. Design Language

**Aesthetic:** Clean, professional SaaS product. Neutral backgrounds with purposeful use of the brand colour as an action signal. Generous whitespace. No decorative illustration — functionality-first.

**Typography:**
- Headings: Inter, weights 600 (h2–h4) and 700 (h1, page titles)
- Body text: system-ui / -apple-system stack, weight 400
- Monospace (refs, IDs, code): `ui-monospace`, weight 400
- Base size: 16px. Line height 1.5 for body, 1.2 for headings
- Scale: 12 / 14 / 16 / 18 / 20 / 24 / 32 / 40px

**Colour palette (from design tokens):**
- Brand primary: `--color-brand-500` (#2563EB) for interactive elements, CTAs, selected states
- Brand hover: `--color-brand-600` (#1d4ed8)
- Surface: `--color-surface-default` (#ffffff), `--color-surface-subtle` (#f8fafc)
- Border: `--color-border-default` (#e2e8f0)
- Text primary: `--color-text-primary` (#0f172a)
- Text secondary: `--color-text-secondary` (#475569)
- Text disabled: `--color-text-disabled` (#94a3b8)
- Success: `--color-success-500` (#16a34a)
- Warning: `--color-warning-500` (#d97706)
- Danger: `--color-danger-500` (#dc2626)

**Elevation (box-shadow):**
- Level 1 (cards at rest): `0 1px 3px rgba(0,0,0,0.08)`
- Level 2 (cards on hover, dropdowns): `0 4px 12px rgba(0,0,0,0.10)`
- Level 3 (modals): `0 16px 40px rgba(0,0,0,0.16)`

---

## 2. Customer-Facing Booking Flow

The public booking page is fully white-label: it shows only the tenant's name and logo — no AnjiSchedulo branding is visible to end customers.

### Page Header

- Background: `--color-surface-default` (#ffffff)
- Border bottom: 1px solid `--color-border-default`
- Left: tenant logo (48×48px, rounded 8px, object-fit: contain). If no logo is set, show tenant initials in a `--color-brand-100` circle with `--color-brand-700` text.
- Left of logo: tenant name in Inter 600 20px `--color-text-primary`
- Right: "Powered by AnjiSchedulo" in 12px `--color-text-disabled` (visible only if tenant has not suppressed it — Enterprise setting)
- Header height: 64px. Sticky on scroll.

### Step Indicator (all booking steps)

- 4 steps shown as numbered circles connected by a horizontal line
- Incomplete step: circle outline `--color-border-default`, number in `--color-text-secondary`
- Current step: filled `--color-brand-500` circle, white number, label in `--color-brand-600` 600 weight
- Completed step: filled `--color-success-500` circle, white checkmark
- Positioned below the header, 24px vertical padding, max-width 640px centred

### Service Cards (Step 1)

- Layout: 2-column CSS grid, gap 16px, max-width 640px centred
- Card: background white, border 2px solid `--color-border-default`, border-radius 12px, padding 20px
- On hover: border-color changes to `--color-brand-300`, box-shadow Level 2, transition 150ms ease
- On selected: border-color `--color-brand-500` (2px), background `--color-brand-50` (#eff6ff)
- Service icon: 32×32px placeholder circle (`--color-surface-subtle`), top-left of card
- Service name: Inter 600 16px `--color-text-primary`
- Duration badge: `--color-surface-subtle` background, rounded-full, 12px `--color-text-secondary`, displayed inline-block below name
- Price: Inter 700 20px `--color-text-primary`, displayed prominently
- Description: 14px `--color-text-secondary`, max 2 lines, text-overflow ellipsis
- Radio input visually hidden; entire card is the click target (aria-pressed on the card div)

### Staff Cards (Step 2)

- Layout: same 2-column grid as service cards
- Avatar: 56×56px circle. If photo: object-fit cover. If no photo: initials (first letter of first and last name) in Inter 600 20px white on `--color-brand-500` background
- "Any available" option: rendered as a card with a generic group icon avatar; always appears first
- "Available today" badge: inline chip, `--color-success-50` background, `--color-success-700` text, 12px Inter 500
- Unavailable staff (not qualified for selected service): card opacity 0.45, cursor not-allowed, no hover state, badge reads "Not available for [service name]" in `--color-text-disabled`

### Calendar (Step 3)

- Layout: two-column on desktop — calendar on left (50%), time slots on right (50%), 24px gap
- Calendar header: month/year in Inter 600 18px, prev/next chevrons (`--color-text-secondary`, 24px)
- Day grid: 7 columns (Mo–Su). Day labels 12px `--color-text-secondary`. Day numbers 14px.
- Date states:
  - Available: `--color-text-primary`, hover `--color-brand-100` background circle, cursor pointer
  - Selected: `--color-brand-500` filled circle, white text
  - Today (not selected): text underline, `--color-brand-600`
  - Closed / outside business hours: `--color-text-disabled`, no hover, cursor default
  - Outside booking window: same as closed
- Time slot panel (right):
  - Header: selected date in Inter 600 16px
  - Slots: 2-column pill grid. Pill: 80px wide, 40px tall, border 1.5px solid `--color-border-default`, border-radius 9999px, 14px Inter 500
  - Available pill hover: border `--color-brand-400`, background `--color-brand-50`
  - Selected pill: background `--color-brand-500`, white text, border `--color-brand-500`
  - Unavailable (already booked): text with strikethrough, `--color-text-disabled`, cursor not-allowed

### Customer Details & Payment Form (Step 4)

- Max-width 480px, centred
- Booking summary bar at top: light grey background, 14px, shows service / staff / date / time / price
- Fields: label above input (Inter 500 14px `--color-text-secondary`). Input: 48px height, full-width, border 1.5px solid `--color-border-default`, border-radius 8px, 16px padding horizontal
- Focus ring: 2px offset outline, `--color-brand-500`, 2px width
- Error state: border `--color-danger-500`, error message 13px `--color-danger-600` below field with ⚠ icon
- Payment section: white card with `--color-border-default` border, Stripe Elements embedded. Lock icon with "Secured by Stripe" 12px `--color-text-secondary`
- CTA button: full-width, 52px height, `--color-brand-500` background, white Inter 600 16px. Hover: `--color-brand-600`. Loading state: spinner replaces text.

### Booking Confirmation Page (Step 5 — Success)

- Centred, max-width 480px, padding-top 64px
- Large checkmark icon: 64×64px circle, `--color-success-50` background, `--color-success-500` check icon (Heroicons check-circle, 40px)
- Heading: Inter 700 28px "Booking confirmed!" `--color-text-primary`
- Subtext: 16px `--color-text-secondary` with customer email
- Appointment card: border-left 4px solid `--color-brand-500`, white background, shadow Level 1, padding 24px. Rows: label (14px `--color-text-secondary`) + value (16px `--color-text-primary` 500 weight)
- Add to calendar: three small outlined buttons side-by-side (Google / Apple / Outlook), 14px, icons 16px
- Secondary CTAs: text links 14px `--color-brand-500`

---

## 3. Tenant Admin Dashboard

### Global Layout

- **App shell:** 100vh, flex row
- **Sidebar:** 240px wide, fixed height, background `#0f172a` (dark navy), no border. Never collapses on desktop.
- **Main content:** flex-grow 1, background `--color-surface-subtle` (#f8fafc), overflow-y scroll

### Sidebar

- Logo area: 64px tall top section. AnjiSchedulo wordmark in white, 18px Inter 700. Plan badge below: pill in `--color-brand-700` background, white 11px text.
- Nav items: 48px tall tap targets, padding horizontal 16px. Icon (20px, white 60% opacity) + label (14px Inter 500, white 80% opacity). Active item: `--color-brand-600` left border 3px, label white 100%, icon white 100%, background `rgba(255,255,255,0.08)`. Hover: background `rgba(255,255,255,0.06)`.
- Nav groups: "Manage" (Dashboard, Appointments, Staff, Services) and "Account" (Settings, Billing, Log out) separated by 1px `rgba(255,255,255,0.12)` line
- Bottom: user avatar (32px initials circle) + user name 13px white 80% + role badge

### Dashboard Page Header

- Page title: Inter 700 28px `--color-text-primary`
- Subtitle: 14px `--color-text-secondary`, e.g. "Monday, 28 June 2026"
- Anomaly alert banner (if active): full-width amber bar below page title. Background `--color-warning-50`, border `--color-warning-200`, left border 4px `--color-warning-500`. Icon ⚠ `--color-warning-600`. Text 14px `--color-text-primary`. Dismiss button ×.

### Metric Cards

- Layout: 4-column CSS grid, gap 16px
- Card: white background, shadow Level 1, border-radius 12px, padding 20px
- Label: 13px Inter 500 `--color-text-secondary`, uppercase letter-spacing 0.5px
- Value: Inter 700 36px `--color-text-primary`, line-height 1
- Trend indicator: small pill below value. Arrow icon (↑ or ↓) + percent. Green (↑ revenue, ↑ bookings = good). Red (↑ cancellation rate = bad, ↓ revenue = bad). Logic controlled by `trendPositive` prop.
- Cards shown: Confirmed Bookings, Net Revenue, Cancellation Rate, Peak Hour

### Today's Appointments (Table)

- White card, shadow Level 1, border-radius 12px
- Table header: 12px Inter 600 `--color-text-secondary` uppercase. Columns: Time, Service, Staff, Status, Actions
- Row height: 56px. Border-bottom 1px `--color-border-default` between rows
- Time column: 14px Inter 600 `--color-text-primary`
- Service column: coloured badge (service-type chip), 13px
- Staff column: 28px avatar circle + name 14px
- Status chip: rounded pill. Confirmed: `--color-success-50` bg / `--color-success-700` text. Cancelled: `--color-surface-subtle` / `--color-text-disabled`.
- Actions: ghost icon buttons (cancel, reschedule) — appear on row hover only

### Forms (Create/Edit Services, Staff, Hours)

- Slide-in drawer from right: 480px wide, full-height overlay, background white, shadow Level 3
- Drawer header: title 20px Inter 600, × close button top right
- Fields follow the same spec as the customer-facing form
- Primary action button at bottom of drawer, full-width

---

## 4. Responsive Breakpoints

| Breakpoint | Width | Layout change |
|---|---|---|
| Desktop | ≥ 1280px | Full sidebar (240px) + content |
| Tablet landscape | 1024px–1279px | Sidebar collapses to 64px icon-only rail |
| Tablet portrait | 768px–1023px | Sidebar hidden, top navigation bar shown |

The customer-facing booking flow has its own breakpoints: ≥ 640px shows two-column service/staff grids; < 640px single column. Calendar + slots switch to stacked layout at < 640px.

---

## 5. Accessibility (WCAG 2.1 AA)

**Colour contrast:**
- `--color-text-primary` (#0f172a) on white (#ffffff): 17.8:1 (exceeds AAA)
- `--color-text-secondary` (#475569) on white: 5.9:1 (passes AA)
- White on `--color-brand-500` (#2563EB): 5.9:1 (passes AA — all CTAs)
- White on `#0f172a` (sidebar): 17.8:1
- `--color-danger-600` (#b91c1c) on `--color-danger-50` (#fef2f2): 5.4:1 (passes AA)

**Focus management:**
- All interactive elements have a visible 2px `--color-brand-500` focus ring on keyboard focus (not only mouse focus)
- Focus must not disappear on custom components (service cards, time slots)
- Modals and drawers: focus is trapped inside while open; returns to trigger element on close

**Screen reader:**
- Service cards: `role="radio"`, wrapped in `role="radiogroup"` with accessible group label
- Time slot pills: `role="button"`, unavailable slots have `aria-disabled="true"` and `aria-label="[time] — unavailable"`
- Status chips in appointment table: `aria-label="Status: Confirmed"` (not just colour)
- Metric card trend: `aria-label="Net Revenue £1,240. Up 12% from yesterday"`
- Alert banners: `role="alert"` with `aria-live="polite"` so screen reader announces them on appearance
- Booking confirmation: `aria-live="polite"` region wrapping the success state so it is announced when it appears
