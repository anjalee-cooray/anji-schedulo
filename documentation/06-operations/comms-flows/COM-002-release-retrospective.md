# COM-002 · Release Retrospective

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

The release retrospective is a structured 30-minute team meeting held within 5 business days of a successful stabilisation period (REL-005). Its goal is to improve the release process, not to assess feature quality (which belongs in product reviews).

---

## 2. Meeting Details

| Field | Value |
|---|---|
| When | Within 5 business days of stabilisation complete |
| Duration | 30 minutes (time-boxed) |
| Facilitator | Engineering Lead (rotates each release) |
| Attendees | Engineering team (all who contributed to the release) |
| Output | Retrospective notes posted to Slack #releases |

---

## 3. Retrospective Agenda

**5 min — Metrics recap**

Present the following for the release:
- Time from RC cut to production deploy
- Validation period (actual vs. target)
- Number of issues found during RC validation
- Post-deploy incidents during stabilisation period (P1/P2 count)
- DLQ events in first 48 hours
- Rollbacks or kill switches applied: yes/no

**10 min — What went well**

Each attendee nominates one thing that worked well. Examples:
- "RC validation checklist caught the migration issue before production"
- "On-call response time was fast when ALT004 fired"
- "Smoke tests ran in under 3 minutes"

**10 min — What to improve**

Each attendee nominates one thing to improve. Examples:
- "Staging DB didn't match production size — migration took longer than expected"
- "PR review took 2 days — blocked RC cut"
- "No clear owner for manual notification test"

**5 min — Action items**

Agree on 1–3 concrete actions with owners and due dates. No more than 3 — focus on the most impactful.

---

## 4. Retrospective Record Template

```markdown
## Release Retrospective — v{N}

Date: YYYY-MM-DD
Facilitator: {name}
Attendees: {names}

### Metrics

| Metric | Target | Actual |
|---|---|---|
| RC cut to production | 48 hours | {X} hours |
| RC validation issues found | 0 | {N} |
| Post-deploy P1/P2 incidents | 0 | {N} |
| DLQ events in first 48 hours | 0 | {N} |
| Kill switch or rollback applied | No | Yes/No |

### What went well

- {item}
- {item}

### What to improve

- {item}
- {item}

### Action items

| Action | Owner | Due |
|---|---|---|
| {specific, measurable action} | {name} | YYYY-MM-DD |
```

Post the completed record to Slack #releases and link it from the release tracking issue.

---

## 5. Action Item Tracking

Action items from retrospectives are tracked as GitHub issues with the label `process-improvement`. The Engineering Lead reviews outstanding process-improvement issues at the start of each sprint.

If an action item is not completed before the next release, it is raised at that retrospective.

---

## 6. Retrospective for Post-Incident Releases

If the release triggered a PIR (O5), the retrospective and PIR are held as separate meetings. The retrospective focuses on the release process; the PIR focuses on the incident root cause.

---

## 7. Traceability

| Concern | Doc |
|---|---|
| Stabilisation period | REL-005-post-release-stabilisation.md |
| Internal comms | COM-001-internal-release-comms.md |
| Incident PIR template | O5-incident-response.md |
| Release plan | O1-release-plan.md |
