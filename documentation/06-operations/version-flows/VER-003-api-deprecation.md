# VER-003 · API Deprecation

**Project:** AnjiSchedulo  
**Last Updated:** 2026-06-28

---

## 1. Purpose

This document defines how AnjiSchedulo deprecates and removes API endpoints, request fields, and response fields. Deprecation is a formal process with a minimum 30-day notice period before removal.

---

## 2. Deprecation Policy

| Item | Minimum notice before removal |
|---|---|
| REST endpoint | 30 days |
| Request body field | 30 days |
| Response body field | 30 days |
| Event payload field (breaking) | 60 days |
| JWT claim | 60 days |
| Webhook payload field | 60 days |

The notice period starts from the date the deprecation notice is sent to affected tenants, not the date the code is marked deprecated.

---

## 3. Deprecation Flow

```
Engineering Lead identifies a field/endpoint to deprecate
    │
    ▼
Step 1: Mark as deprecated in OpenAPI spec (D5)
    /v1/appointments/{id}/reschedule:
      deprecated: true
      description: "Deprecated in v1.5.0. Use POST /v1/appointments/{id}/slots instead."
    │
    ▼
Step 2: Add deprecation response header to API gateway
    Deprecated: true
    Sunset: {ISO-8601 date of removal}
    Link: <https://docs.anji-schedulo.com/migration/reschedule-v2>; rel="deprecation"
    │
    ▼
Step 3: Add deprecation log warning
    Whenever the deprecated endpoint is called:
    logger.warn({
      event: 'deprecated_endpoint_called',
      endpoint: '/v1/appointments/{id}/reschedule',
      tenant_id: ctx.tenantId,
      sunset_date: '2026-09-01',
    });
    │
    ▼
Step 4: Send deprecation notice (COM-001 tenant communication)
    Subject: Action required — {endpoint} deprecated on {sunset_date}
    Include: migration guide link
    │
    ▼
Step 5: Monitor usage during deprecation period
    LogQL query to track calls to deprecated endpoint:
    {env="production"} |= "deprecated_endpoint_called" | json
    |= "endpoint=\"/v1/appointments/{id}/reschedule\""

    If usage is still high 2 weeks before sunset: send reminder notice
    │
    ▼
Step 6: Sunset date — remove the endpoint
    - Remove from api-gateway routing
    - Remove from OpenAPI spec
    - Remove from all service implementations
    - Update CHANGELOG.md under ### Removed
    - Version bump (MAJOR if breaking, MINOR otherwise)
```

---

## 4. Sunset Response

After the sunset date, any call to the removed endpoint returns:

```json
HTTP 410 Gone

{
  "type": "https://docs.anji-schedulo.com/errors/endpoint-removed",
  "title": "Endpoint removed",
  "status": 410,
  "detail": "POST /v1/appointments/{id}/reschedule was removed on 2026-09-01. Use POST /v1/appointments/{id}/slots instead.",
  "migration_guide": "https://docs.anji-schedulo.com/migration/reschedule-v2"
}
```

---

## 5. Internal API Deprecation (Service-to-Service)

Internal service-to-service APIs (not tenant-facing) follow a shorter deprecation cycle:
- 14-day notice to the engineering team
- Tracked in a GitHub issue
- No tenant communication required
- Code removed after all callers are updated

---

## 6. Event Schema Deprecation

If an event payload field is being removed (60-day notice):
1. Publish the event with both old and new field during the transition:
   ```json
   {
     "appointment_reference": "APT-001",   // deprecated
     "appointment_id": "uuid-...",          // new
   }
   ```
2. After 60 days, remove the deprecated field.
3. Update `specs/ai/events.json` and regenerate event documentation.

---

## 7. Traceability

| Concern | Doc |
|---|---|
| API design | D5-api-design.md |
| Major version planning | VER-001-major-version-planning.md |
| Tenant communication | COM-001-internal-release-comms.md |
| Events schema | specs/ai/events.json |
