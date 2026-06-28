# DX3 · Git Workflow

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Branch Strategy

AnjiSchedulo uses a trunk-based development model with short-lived feature branches.

```
main (protected)
  └── release/v1.x.x          (release branch — staging + production deploy)
        └── feature/{ticket}-{description}   (feature branches)
        └── fix/{ticket}-{description}       (bug fix branches)
        └── hotfix/{ticket}-{description}    (production hotfixes)
        └── chore/{description}              (tooling, deps, docs)
```

**`main`** is the source of truth. It is always deployable. Direct pushes are blocked — all changes go through PRs.

---

## 2. Branch Naming

| Type | Pattern | Example |
|---|---|---|
| Feature | `feature/{ticket}-{short-description}` | `feature/AS-142-reschedule-flow` |
| Bug fix | `fix/{ticket}-{short-description}` | `fix/AS-201-slot-lock-timeout` |
| Hotfix | `hotfix/{ticket}-{short-description}` | `hotfix/AS-210-stripe-webhook-sig` |
| Release | `release/v{major}.{minor}.{patch}` | `release/v1.1.0` |
| Chore | `chore/{short-description}` | `chore/upgrade-spring-boot-3-3` |

Ticket numbers refer to Linear issue IDs.

---

## 3. Commit Message Format

Commits follow the Conventional Commits specification:

```
<type>(<scope>): <short summary>

[optional body]

[optional footer]
```

**Types:**

| Type | When to use |
|---|---|
| `feat` | New feature or capability |
| `fix` | Bug fix |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `test` | Adding or updating tests |
| `docs` | Documentation changes only |
| `chore` | Tooling, dependencies, build config |
| `perf` | Performance improvement |

**Scope** is the service or layer affected: `booking`, `availability`, `payment`, `notification`, `tenant`, `dashboard`, `ops`, `api-gateway`, `db`, `infra`, `shared`.

**Examples:**

```
feat(booking): add idempotency key deduplication to saga orchestrator

Prevents duplicate booking charges when the client retries a request.
BookingIdempotencyKey checked before slot lock acquisition (BR006).

Closes AS-142
```

```
fix(availability): correct slot boundary calculation for DST transitions

Slots spanning a DST change were returning incorrect UTC end times.
Now converts to tenant timezone before boundary check.

Closes AS-201
```

```
chore: upgrade spring-boot to 3.3.2
```

---

## 4. Day-to-Day Workflow

```bash
# 1. Start from an up-to-date main
git checkout main
git pull origin main

# 2. Create your branch
git checkout -b feature/AS-142-reschedule-flow

# 3. Make changes, commit frequently with meaningful messages
git add apps/booking-command/src/booking/booking.service.ts
git commit -m "feat(booking): add reschedule saga step"

# 4. Keep branch up to date with main (rebase, not merge)
git fetch origin
git rebase origin/main

# 5. Push and open PR
git push origin feature/AS-142-reschedule-flow
# Open PR in GitHub — use the PR template
```

---

## 5. Rebase, Not Merge

When incorporating changes from `main` into your feature branch, use rebase — not merge:

```bash
# ✅ Rebase keeps history linear
git fetch origin
git rebase origin/main

# ❌ Merge creates a merge commit — noisy history
git merge origin/main
```

If a rebase creates conflicts, resolve them and continue:

```bash
# After resolving conflicts in editor
git add {resolved-files}
git rebase --continue
```

---

## 6. Squashing

Before opening a PR, squash work-in-progress commits into logical units. A good rule: one commit per logical change (not one commit per file, not one commit for the entire PR).

```bash
# Interactive rebase to squash last 5 commits
git rebase -i HEAD~5

# In the editor, change 'pick' to 'squash' (or 's') for commits to squash
```

The PR itself is squash-merged — GitHub squashes all PR commits into one commit on `main`. The PR title becomes the commit message, so write a meaningful PR title.

---

## 7. Release Branch Workflow

```bash
# Cut a release branch from main
git checkout main && git pull
git checkout -b release/v1.1.0

# Push — triggers staging deploy automatically (CD-002)
git push origin release/v1.1.0

# After staging validation, open PR: release/v1.1.0 → main
# PR review + manual approval gate → triggers production deploy (CD-003)
```

Release branches are short-lived — they exist only for the duration of the staging + production deployment. Once merged, they are deleted.

---

## 8. Hotfix Workflow

For production incidents requiring an immediate fix that cannot wait for the normal release process:

```bash
# Branch from main (not from release/*)
git checkout main && git pull
git checkout -b hotfix/AS-210-stripe-webhook-sig

# Make the minimal fix
git commit -m "fix(api-gateway): validate Stripe-Signature header before processing"

# Push — triggers PR → staging → production (CD-003 with manual approval)
git push origin hotfix/AS-210-stripe-webhook-sig

# Fast-track approval: notify Engineering Lead in Slack #incidents
# After production deploy: tag the hotfix
git tag v1.0.1
git push origin v1.0.1
```

Hotfixes go through staging before production — no exceptions (NFR001 requires zero-downtime validated deploys).

---

## 9. Protected Branch Rules

`main` and `release/*` branches have these GitHub branch protection rules:

| Rule | Setting |
|---|---|
| Require PR before merging | ✓ |
| Required reviewers | 1 (feature), 2 (release/main) |
| Require status checks | PR pipeline (CD-001) must pass |
| Dismiss stale reviews on push | ✓ |
| Require linear history | ✓ (squash merge only) |
| Block direct push | ✓ |

---

## 10. Tags and Versioning

Versions follow Semantic Versioning (semver):

- `MAJOR` — breaking change to the API or tenant data model
- `MINOR` — new feature, backward-compatible
- `PATCH` — bug fix or hotfix

Tags are created after every production deploy:

```bash
git tag v1.1.0 -m "Release v1.1.0 — reschedule flow"
git push origin v1.1.0
```
