# COM-001 · Internal Release Comms

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

This document defines the communication plan for each production release. It covers internal team notifications and outbound tenant communications. All release communications are logged in Slack #deployments.

---

## 2. Communication Timeline

| When | Audience | Channel | Content |
|---|---|---|---|
| RC cut (REL-001) | Engineering team | Slack #deployments | RC SHA, validation period, Go/No-Go date |
| Go decision (REL-003) | Engineering team | Slack #deployments | GO confirmed, deploy time |
| Production deploy starts (REL-004) | Engineering team | Slack #deployments | Deploy in progress |
| Deploy stable (REL-004) | Engineering team | Slack #deployments | v{N} live and stable |
| PATCH/MINOR features | Tenant admins | Email | What's new (optional for patches) |
| MAJOR release or breaking change | Tenant admins | Email | 30-day advance notice + migration guide |
| API deprecation | Tenant admins | Email | Per VER-003 — 30+ day notice |

---

## 3. Slack Message Templates

### RC Cut

```
🔖 RC cut — release/v{N}

SHA: {git-sha}
Scope: {brief list of FRs / bug fixes}
Validation period: {start date} to {Go/No-Go date}
Go/No-Go: {date} at 09:00 UTC

RC deployed to staging. Validation in progress.
```

### Go Decision — GO

```
✅ GO — v{N} is cleared for production

Decision: GO
Decision maker: @{Engineering Lead}
Deploy time: {date} at 10:00 UTC
On-call: @{engineer}
Runbook: O4-runbook.md
```

### Production Deploy Complete

```
🟢 v{N} LIVE in production

SHA: {git-sha}
Deploy duration: {X} minutes
Smoke tests: passed
Monitoring window: 10 minutes (until {HH:MM} UTC)
Grafana: http://grafana.internal/d/platform-overview
```

### Stabilisation Complete

```
🏁 v{N} STABLE

48-hour stabilisation period complete.
No P1/P2 incidents since deploy.
Release branch archived.
Retrospective: COM-002 — {retro date}
```

---

## 4. Tenant Email Templates

### New Feature Announcement (MINOR release)

```
Subject: AnjiSchedulo v{N} — What's new

Hi {tenant_name},

We've released AnjiSchedulo v{N} with the following improvements:

🆕 {Feature name}: {one sentence benefit}
🐛 Fixed: {issue description} — {what improved}

The update was applied automatically to your account.
No action is required.

If you have any questions, reply to this email.

— The AnjiSchedulo Team
```

### Breaking Change Notice (30-day advance)

```
Subject: Action required — AnjiSchedulo v{N} deprecates {endpoint/feature} on {sunset_date}

Hi {tenant_name},

On {sunset_date}, we will remove the following:

{deprecated endpoint or feature}

What you need to do:
{migration steps in plain language}
Migration guide: {link}

This change will not affect your account until {sunset_date}.
After that date, calls to the deprecated endpoint will return HTTP 410.

If you need more time or have questions, contact us before {sunset_date - 7 days}.

— The AnjiSchedulo Team
```

---

## 5. Status Page

For MAJOR releases or planned maintenance affecting booking availability:
- Update the status page at `status.anji-schedulo.com` with a maintenance notice.
- Post a "maintenance complete" update after the release.
- The status page is updated manually via the status page provider.

---

## 6. Traceability

| Concern | Doc |
|---|---|
| Release process | REL-004-production-release.md |
| Retrospective | COM-002-release-retrospective.md |
| API deprecation comms | VER-003-api-deprecation.md |
| Tenant data / GDPR | A6-data-privacy-arch.md |
