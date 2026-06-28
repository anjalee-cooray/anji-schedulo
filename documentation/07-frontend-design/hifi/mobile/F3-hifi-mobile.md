---
title: Hi-fi Design Spec — Mobile
layer: 07-frontend-design
status: current
lastUpdated: 2026-06-28
---

# F3 · Hi-fi Design Specification — Mobile

This document defines the intended visual design for the AnjiSchedulo mobile experience (375px–430px viewport). Mobile uses the same design token system as the web spec (`F3-hifi-web.md`) but applies mobile-specific layout patterns. All design tokens, colour values, and typography scales are shared; only layout, interaction patterns, and navigation structure differ.

---

## 1. Viewport & Layout

- **Base width:** 375px (iPhone SE / iPhone 14 minimum). Scales fluidly to 430px.
- **Safe area insets:** respect `env(safe-area-inset-*)` for notch and home indicator
- **Single column:** all layouts are full-width single column. No sidebar. No multi-column grids except where explicitly noted.
- **Touch targets:** minimum 44×44px for all interactive elements (WCAG 2.5.5 Target Size)
- **Spacing unit:** 4px base. Common spacings: 8, 12, 16, 20, 24px

---

## 2. Navigation

### Customer-Facing (Unauthenticated Booking Flow)

- No persistent navigation bar during the booking flow — the flow is a dedicated funnel
- Back arrow in top-left (44×44px tap target) navigates to previous step
- Booking step indicator shown in a horizontal strip below a minimal header (tenant logo + name, 56px tall)

### Tenant Admin (Authenticated)

- **Bottom navigation bar:** fixed at bottom. Height 56px + safe-area-inset-bottom. Background white. Border-top 1px `--color-border-default`.
- Tabs (5 for admin): Home (dashboard icon), Appointments (calendar icon), Staff (person-group icon), Services (grid icon), Settings (cog icon)
- Tab label: 11px Inter 500. Active: `--color-brand-500` icon + label. Inactive: `--color-text-disabled`.
- Tab active indicator: 2px `--color-brand-500` bar at the top of the active tab

### Customer (Authenticated — "My Bookings")

- Bottom navigation bar with 3 tabs: Home (booking page link), My Bookings (calendar), Notifications (bell)

---

## 3. Customer-Facing Booking Flow

### Page Header (all steps)

- Height 56px. White background. Border-bottom 1px `--color-border-default`.
- Tenant logo (36px) + name (Inter 600 16px) centred
- Step progress: 4-dot indicator below header (same token colours as web, dots 8px each, selected 20px pill transition with CSS)
- No "Powered by AnjiSchedulo" branding visible in the booking flow header

### Service Selection (Step 1)

- Full-width stacked cards. Margin horizontal 16px. Vertical gap 12px.
- Card: white background, border 2px `--color-border-default`, border-radius 12px, padding 16px, min-height 80px
- Service name: Inter 600 16px left-aligned, `--color-text-primary`
- Duration + price: same row, 14px `--color-text-secondary`
- Right side: radio circle (24×24px). Selected: filled `--color-brand-500`.
- Entire card is the tap target
- Sticky CTA bar at screen bottom: white background, 16px padding, "Continue →" button full-width 52px, `--color-brand-500`, disabled until selection made

### Staff Selection (Step 2)

- Same full-width stacked card pattern
- Avatar: 48×48px circle (initials or photo), left side of card
- Name + service qualification: 16px Inter 500 + 13px `--color-text-secondary` below
- "Any available" card: appears first, icon is a person-with-plus outline 48×48px
- Unavailable staff: opacity 0.4, no tap response, label "Unavailable for [service]" 12px `--color-text-disabled`
- Sticky CTA bar: same as step 1

### Date & Time Picker (Step 3)

**Date strip (horizontal scroll):**
- Horizontally scrollable row of date chips. Chip width 48px, height 64px. Border-radius 12px.
- Chip contents: day label 12px `--color-text-secondary` (e.g. "Thu") + date number 18px Inter 700
- Selected date chip: `--color-brand-500` background, white text
- Closed date chip: `--color-surface-subtle` background, `--color-text-disabled` text, no tap response
- Today (not selected): blue underline on date number

**Time slot list (vertical, below date strip):**
- Section header: selected date in full (e.g. "Thursday, 4 June") — 14px Inter 600 `--color-text-secondary`, 16px padding
- Time slots as full-width tappable rows: 56px height, border-bottom 1px `--color-border-default`
- Available slot: time (Inter 600 16px `--color-text-primary`) left-aligned, radio circle right-aligned
- Unavailable slot: time with strikethrough, `--color-text-disabled`, "Booked" badge right-aligned
- Selected slot: row background `--color-brand-50`, time in `--color-brand-600`, filled radio

### Customer Details & Payment (Step 4)

- Booking summary: full-width grey bar, 14px, 12px vertical padding. Shows: "[service] · [staff] · [date] · [time]"
- Form fields: same token values as web spec. Input height 52px for comfortable mobile tap.
- Keyboard: `inputmode="email"` on email, `inputmode="tel"` on phone, `inputmode="numeric"` on card fields
- Stripe Elements: rendered at their native mobile-optimised height
- Payment section in its own white card with 16px margin horizontal, shadow Level 1
- Sticky bottom bar: "Confirm — £X.XX →" button. Disabled until all required fields valid.

### Booking Confirmation (Step 5 — Full-screen)

- Full-screen success state (no header, no bottom nav during celebration moment)
- Large check icon: 80×80px `--color-success-500` circle, white check, centred, margin-top 64px
- Heading: Inter 700 24px "Booking confirmed!" centred
- Subtext: email confirmation line 15px `--color-text-secondary` centred
- Appointment card: margin horizontal 16px, border-radius 16px, shadow Level 1, padding 20px
- Add to calendar: three equal-width buttons in a row (Google / Apple / Outlook), 14px, icon 16px, border 1.5px `--color-border-default`, border-radius 8px
- Auto-redirect: after 4 seconds a "View my bookings" link appears; no forced redirect
- Secondary action: "Book another appointment" text link 14px `--color-brand-500` centred

---

## 4. Tenant Admin — Mobile Patterns

### Dashboard Screen

- Page title: Inter 700 22px, 16px horizontal padding, 20px top padding
- Anomaly alert: full-width amber banner below title (same spec as web), 14px, amber left border 4px, dismiss × button
- Metric cards: **2-column grid** (the only multi-column layout on mobile). Gap 12px, 16px horizontal padding.
- Card: padding 16px, border-radius 12px, shadow Level 1. Label 12px uppercase. Value Inter 700 28px.
- Today's appointments: vertical list of `AppointmentListItem` components (see F4). Tapping a row opens a bottom sheet with full detail and action buttons.

### Forms — Bottom Sheet Pattern

- All create/edit forms (Add Service, Add Staff, Edit Hours) slide up as a bottom sheet, not a full-screen page
- Bottom sheet: white background, border-radius 16px 16px 0 0 at top, shadow Level 3
- Drag handle: 4×32px grey pill centred at top, 8px top padding
- Max-height: 90vh, scrollable content
- Header: title Inter 600 18px + × close (44×44px tap target)
- Primary action button: full-width 52px, sticky at bottom of sheet (above safe area)

### Staff Management

- Staff appear as full-width list items (not cards): avatar + name + service tags + Edit chevron
- "Add Staff Member" as a full-width outlined button at top of list

### Services Management

- Same full-width list pattern: service name + duration + price + status badge + Edit chevron
- Plan limit indicator: sticky notice at top if at tier limit ("3 of 5 services. Upgrade for unlimited.")

---

## 5. Accessibility — Mobile Specifics

**Touch:**
- All interactive elements ≥ 44×44px. Cards and list items stretch to full row width to maximise target size.
- Swipe-to-dismiss is a secondary interaction only — never the sole way to perform an action (e.g. cancel button always present alongside swipe on appointment cards)

**Focus:**
- iOS VoiceOver and Android TalkBack tested against all booking flow screens
- Service cards: `accessibilityRole="radio"`, `accessibilityState={{ selected }}`, `accessibilityLabel="Haircut, 45 minutes, £40. Double-tap to select."`
- Time slots: `accessibilityRole="button"`, `accessibilityState={{ disabled: !available }}`
- Bottom sheet: focus trap applied; first focusable element receives focus on open; Escape (keyboard) and swipe-down (touch) dismiss

**Colour contrast:**
- Same WCAG 2.1 AA targets as web. All mobile-specific chip and badge colours verified against background.

**Reduced motion:**
- Bottom sheet slide-up animation respects `prefers-reduced-motion: reduce` — fades instead of translates
- Confirmation success animation (check icon scale-in) skipped when reduced motion is preferred
