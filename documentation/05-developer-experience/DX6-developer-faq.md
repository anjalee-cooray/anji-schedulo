# DX6 · Developer FAQ

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## Local Development

**Q: `./gradlew bootRunAll` starts all services, but I only want to work on `booking-command-service`. How do I start just that service and its dependencies?**

```bash
./gradlew :services:booking-command-service:bootRun
```

Start the services it depends on separately in other terminals (`availability-service`, `payment-service`, `api-gateway`), or run the full stack via Docker Compose for the backing services — Docker Compose handles the DB and Redis.

---

**Q: The database is in a bad state from testing. How do I reset it?**

```bash
docker compose down -v          # removes the postgres volume
docker compose up -d postgres   # restarts postgres
./gradlew flywayMigrate         # re-runs all Flyway migrations
./gradlew dbSeed                # re-seeds demo data
```

---

**Q: How do I connect to the local database directly?**

```bash
psql -h localhost -p 5432 -U booking_app_user -d anjischedulo
# password: local_dev_password (from .env.local)
```

Or use TablePlus / DBeaver with those same connection details.

---

**Q: How do I test RLS locally? I want to verify that setting `app.tenant_id` to null returns zero rows.**

```sql
-- In psql, connect as booking_app_user
SET app.tenant_id = '';         -- empty string, not a real tenant
SELECT * FROM appointments;    -- should return 0 rows

SET LOCAL app.tenant_id = 'tid-demo-salon';
SELECT * FROM appointments;    -- returns only demo-salon rows
```

---

**Q: LocalStack is running but SNS/SQS events aren't being delivered. How do I debug?**

1. Check that `outbox-relay` is running: it polls `outbox_records` and publishes to SNS.
2. Check `outbox_records` for undelivered events:
   ```sql
   SELECT * FROM outbox_records WHERE published_at IS NULL ORDER BY created_at;
   ```
3. Check outbox-relay logs in the terminal — it logs every SNS publish attempt.
4. Verify LocalStack is healthy:
   ```bash
   curl http://localhost:4566/_localstack/health
   ```
5. Verify SNS topics exist in LocalStack:
   ```bash
   aws --endpoint-url=http://localhost:4566 sns list-topics
   ```

---

## Architecture

**Q: Why doesn't `booking-command-service` directly call `notification-service`? Wouldn't that be simpler?**

Two reasons:
1. **Durability**: if `notification-service` is down when a booking is confirmed, the notification is lost. With the Outbox + SQS pattern, the notification is delivered when `notification-service` recovers — guaranteed.
2. **Decoupling**: `booking-command-service` should not know or care whether notifications are sent by email, SMS, or push. The consumer decides how to handle the event.

---

**Q: Why does `notification-service` use `event.payload` for customer details instead of calling back to `booking-command-service`?**

This is the Event-Carried State Transfer pattern. The event payload contains a full snapshot of the data the consumer needs (`customer_name`, `customer_email`, `appointment_time`, etc.). This means:
- `notification-service` can process the event even if `booking-command-service` is down.
- No network call = no additional latency or failure point.
- The event is a self-contained record of what happened.

The trade-off: event payloads are larger. This is acceptable — SNS/SQS message size limit is 256 KB, and appointment payloads are well under 1 KB.

---

**Q: When should I use saga orchestration vs. choreography?**

| Use orchestration when... | Use choreography when... |
|---|---|
| Steps must happen in strict order | Steps are independent |
| A failure in step N must compensate steps 1–(N-1) | Failure in one step doesn't block others |
| You need a single source of truth for saga state | Each participant owns its own failure handling |
| The flow is complex and needs visibility | The flow is simple |

In AnjiSchedulo: booking creation uses orchestration (slot lock → payment → confirm; payment failure must release the slot). Cancellation uses choreography (slot release, refund, and notification are independent — a failed refund doesn't block slot release, per BR009).

---

**Q: What is the Transactional Outbox pattern and why does AnjiSchedulo use it?**

Without the Outbox, there is a race condition:

```java
// ❌ Without Outbox — can fail between steps
jdbcTemplate.update("INSERT INTO appointments ...");
sns.publish(...);  // if this fails, appointment exists but no event was emitted
```

With the Outbox:

```java
// ✅ With Outbox — atomic (@Transactional wraps both inserts)
@Transactional
public void createBooking(...) {
    jdbcTemplate.update("INSERT INTO appointments ...");
    jdbcTemplate.update("INSERT INTO outbox_records ...");  // same transaction
}
// outbox-relay publishes separately, with its own retry
```

The outbox guarantees that if the appointment is written, the event will eventually be published — even if the service crashes between the commit and the SNS publish call.

---

**Q: Why do we use SQS FIFO with `MessageGroupId = tenant_id`?**

SQS FIFO with `MessageGroupId` groups messages within a FIFO queue. Messages within the same group are processed in order. Using `tenant_id` as the group means:
- Events for tenant_A are processed in order relative to each other.
- A burst of events from tenant_A doesn't block tenant_B's events (different group — processed concurrently).
- One noisy tenant can't starve other tenants (NFR011).

---

## Testing

**Q: Why are integration tests not allowed to mock the database?**

A previous incident: mocked database tests passed because the mock didn't enforce RLS policies. The real PostgreSQL instance rejected a query that the mock accepted. The bug reached production before being caught. Since then, integration tests must use a real PostgreSQL instance via Testcontainers with all Flyway migrations applied, including all RLS policies.

---

**Q: How do I write a cross-tenant isolation test?**

```java
@Test
void returns404WhenTenantAReadsTenantBAppointment() throws Exception {
    // Arrange
    String tokenA = getTestJwt("tenant-a", "tenant_admin");
    UUID appointmentBId = createTestAppointment("tenant-b");

    // Act + Assert
    mockMvc.perform(get("/appointments/{id}", appointmentBId)
            .header("Authorization", "Bearer " + tokenA))
        .andExpect(status().isNotFound())
        .andExpect(jsonPath("$.tenant_id").doesNotExist());
}
```

The test must not just check for 404 — also verify that no tenant_B data appears in the response body.

---

**Q: How do I test the booking saga locally end-to-end with a real Stripe call?**

Use Stripe's test mode. The `.env.local` file uses `sk_test_...` keys. You can use Stripe's test card numbers:

| Card Number | Behaviour |
|---|---|
| `4242 4242 4242 4242` | Payment succeeds |
| `4000 0000 0000 0002` | Payment declined |
| `4000 0025 0000 3155` | 3D Secure required |

Use the Stripe CLI to forward webhooks locally:

```bash
stripe listen --forward-to localhost:4000/webhooks/stripe
```

---

## Deployment

**Q: How do I deploy to staging for a quick test without going through the full release branch process?**

Create a `release/test-{your-name}` branch:

```bash
git checkout -b release/test-anjali
git push origin release/test-anjali
# Triggers staging deploy (CD-002) automatically
```

Delete the branch when done. Do not merge it to `main`.

---

**Q: A migration I wrote passed in dev and staging but failed in production. What do I do?**

1. The pipeline should have halted before deploying services — verify services are still on the old version.
2. Check the Flyway logs in CloudWatch: `/anji-schedulo/production/flyway`
3. The pre-migration RDS snapshot (`prod-pre-{sha}`) is available for PITR if the DB is in a bad state.
4. Write a forward-fix migration that corrects the issue.
5. Do not roll back services unless the old code is incompatible with the current (partially-migrated or pre-migrated) schema.
6. Alert the Engineering Lead immediately — do not attempt to fix alone if data corruption is suspected.

---

**Q: How do I check which version is running in production?**

```bash
# Check ECS task definition revision (which image tag)
aws ecs describe-services \
  --cluster anji-schedulo-production \
  --services api-gateway \
  --query 'services[0].taskDefinition'

# Health endpoint returns the deployed commit SHA
curl https://api.anji-schedulo.com/health
# {"status":"ok","version":"abc123...","service":"api-gateway"}
```

---

## Security

**Q: I need to test something in production but I don't have production secrets. How do I get access?**

Production secrets are never shared with individual engineers. If you need production access for an incident investigation:
1. Use the operator role via `ops-service` — which provides scoped read access.
2. For deeper investigation: request temporary IAM role assumption with the Engineering Lead's approval, time-boxed to the investigation.
3. All production access is logged in CloudTrail.

---

**Q: I accidentally committed a secret to a feature branch. What do I do?**

1. Immediately rotate the secret — assume it is compromised.
2. Remove it from git history: `git rebase -i` to drop the commit, or `git filter-repo` for deep history.
3. Force-push the branch (feature branches only — never `main`).
4. Notify the Engineering Lead so the rotation can be verified.
5. Check CloudTrail to see if the secret was used by any unexpected principal.
