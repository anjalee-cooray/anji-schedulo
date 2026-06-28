# VER-002 · Changelog and Release Notes

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

This document defines how AnjiSchedulo maintains its changelog and generates release notes. The changelog is the authoritative record of what changed in each version.

---

## 2. Changelog Location and Format

The changelog lives at `CHANGELOG.md` in the repository root. It follows the [Keep a Changelog](https://keepachangelog.com) format and uses semantic versioning.

```markdown
# Changelog

## [Unreleased]

### Added
- {new feature}

### Fixed
- {bug fix}

### Changed
- {changed behaviour}

### Removed
- {removed feature}

### Security
- {security fix}

## [1.0.1] - 2026-06-15

### Fixed
- Booking saga: compensate transaction now correctly releases slot on payment failure (BR004)

## [1.0.0] - 2026-06-01

### Added
- Tenant self-registration and provisioning (FR001)
- Real-time slot availability query via Redis cache (FR003)
...
```

---

## 3. Changelog Update Process

```
PR is merged to main
    │
    ▼
Author moves relevant entry from [Unreleased] to [vN]
    (this is done as part of the release PR, not the feature PR)
    │
    ▼
Release PR includes:
    1. CHANGELOG.md update with version and date
    2. package.json version bump
    3. infra/environments/production/terraform.tfvars image_tag update
    │
    ▼
After production deploy (REL-004):
    git tag v{N}
    git push origin v{N}
```

---

## 4. What to Include in the Changelog

**Include:**
- New user-facing features (reference FR IDs)
- Bug fixes that affected production behaviour
- Breaking changes (always under a `### Changed` or `### Removed` header)
- Security fixes (always under `### Security`)
- Database migration notes (if a migration changes observable behaviour)

**Do not include:**
- Internal refactors with no user-visible change
- Test additions
- Dependency bumps (unless a security fix)
- Infrastructure changes with no service-level impact

---

## 5. Release Notes (Tenant-Facing)

For MINOR and MAJOR releases, the Engineering Lead writes brief tenant-facing release notes. These are separate from the developer changelog and are sent via the tenant notification email (COM-001).

Tenant-facing release notes format:
```
Subject: AnjiSchedulo v{N} — What's new

Hi {tenant name},

We've released v{N} with the following updates:

🆕 New: {feature name} — {one sentence benefit to the tenant}
🐛 Fixed: {issue} — {what improved}
⚠️  Important: {any action required from tenant}

The update was applied automatically. No action is needed unless noted above.

— AnjiSchedulo Team
```

---

## 6. Git Tag Conventions

| Tag | When created | Example |
|---|---|---|
| `v{MAJOR}.{MINOR}.{PATCH}` | After production deploy (REL-004) | `v1.0.0` |
| `v{N}-rc.{YYYYMMDD}` | When RC is cut (REL-001) | `v1.0.0-rc.20260628` |

Tags are lightweight (not annotated). Created by the Engineering Lead.

---

## 7. Traceability

| Concern | Doc |
|---|---|
| Release process | REL-004-production-release.md |
| Breaking changes | VER-001-major-version-planning.md |
| API deprecation | VER-003-api-deprecation.md |
| Tenant communication | COM-001-internal-release-comms.md |
