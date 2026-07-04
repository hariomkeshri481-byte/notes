# Designing a Booking.com-like Application — End to End System Design

A hotel/travel booking platform where users search for hotels, view availability, and book rooms — with millions of listings, high read traffic, and the hardest constraint of all: **never double-book a room**.

---

## 1. Requirements

### Functional Requirements
- Search hotels by location, date range, price, rating, amenities
- View hotel details, room types, and real-time availability
- Book a room for a date range (with payment)
- Cancel / modify a booking
- Hotel partners can manage inventory (add rooms, set prices, block dates)
- Reviews and ratings
- Notifications (booking confirmation, reminders)

### Non-Functional Requirements
- **Consistency where it matters most:** no double-booking (correctness > availability for the booking write path)
- **High availability for search** (search can tolerate slightly stale data; booking cannot tolerate incorrect data)
- **Low search latency** (users expect results in under ~200ms)
- **Scale:** millions of hotels, hundreds of millions of searches/day, spikes during holidays/sales
- **Read-heavy overall** (search reads vastly outnumber booking writes — maybe 1000:1)

**Cross-question:** *"What's the read/write ratio here and why does it matter?"*
> Search is read-heavy and can be served from caches/replicas that are slightly stale (eventual consistency is fine — showing a room as "available" that got booked 2 seconds ago just means the user finds out at checkout). But the actual booking write is a strict consistency operation — this asymmetry is why the system is split into two very differently-designed subsystems (search vs. booking), not one uniform architecture.

---

## 2. High-Level Architecture

```
                        ┌─────────────┐
                        │   Client    │
                        │ (Web/Mobile)│
                        └──────┬──────┘
                               │
                        ┌──────▼──────┐
                        │ API Gateway │  (auth, rate limiting, routing)
                        └──────┬──────┘
              ┌────────────────┼────────────────┐
              │                │                 │
      ┌───────▼──────┐  ┌──────▼───────┐  ┌──────▼───────┐
      │ Search Service│  │Booking Service│  │ Hotel/Inventory│
      │ (read-optimized)│ │(write-critical)│ │   Service     │
      └───────┬──────┘  └──────┬───────┘  └──────┬───────┘
              │                │                 │
      ┌───────▼──────┐  ┌──────▼───────┐  ┌──────▼───────┐
      │Elasticsearch/ │  │  SQL DB      │  │   SQL DB      │
      │Redis (cache)  │  │ (with locking│  │  (hotel data) │
      │               │  │  /versioning)│  │               │
      └──────────────┘  └──────┬───────┘  └──────────────┘
                                │
                        ┌───────▼───────┐
                        │  Payment      │
                        │  Service      │
                        │ (external/PSP)│
                        └───────────────┘

      ┌──────────────────────────────────────┐
      │  Async: Message Queue (Kafka/SQS)     │
      │  → Notification Service                │
      │  → Analytics / Data Warehouse          │
      │  → Search Index Updater                │
      └──────────────────────────────────────┘
```

**Cross-question:** *"Why split Search and Booking into separate services instead of one Hotel service?"*
> They have opposite scaling and consistency needs. Search needs to scale horizontally with cheap reads and can use eventually-consistent replicas/caches. Booking needs strong consistency, is much lower volume, and a bug there costs real money (double-booked rooms, refund disputes). Coupling them means you either over-engineer search for consistency (killing latency) or under-engineer booking for scale (risking correctness). Separate services let each be optimized and scaled independently.

---

## 3. Data Model (Simplified)

```
Hotel(hotel_id, name, location, geo_lat, geo_long, rating, amenities)
RoomType(room_type_id, hotel_id, name, base_price, total_rooms)
Inventory(room_type_id, date, available_count, price)   -- one row per room-type per date
Booking(booking_id, user_id, room_type_id, check_in, check_out, status, version)
Payment(payment_id, booking_id, amount, status)
```

**Key design decision: the `Inventory` table is date-granular, not a raw room count.**
Instead of storing `total_rooms - booked_rooms` and recalculating on the fly, precompute an `available_count` per `(room_type_id, date)`. This makes availability checks a simple range query instead of an aggregation over all bookings — critical for search performance.

**Cross-question:** *"Why not just count bookings that overlap a date range every time someone searches?"*
> That's an expensive query (scanning/aggregating overlapping date ranges across potentially millions of bookings) run on every single search request. Precomputing and maintaining a per-date availability counter turns a search-time aggregation into a simple indexed lookup. The tradeoff: you now have to *keep that counter in sync* every time a booking is made or cancelled — which is exactly the concurrency problem tackled in Section 5.

---

## 4. Search Flow

1. User searches "hotels in Paris, June 10–15, 2 guests"
2. Query hits **Elasticsearch** (or similar) — indexed by location (geo-query), date availability, price, amenities
3. Elasticsearch index is a **denormalized, eventually-consistent copy** of hotel + availability data, updated asynchronously via a queue whenever inventory changes
4. Results are paginated, ranked (by relevance, price, popularity — often ML-ranked), and returned
5. Hot searches (popular cities/dates) are cached in **Redis** with a short TTL (seconds to low minutes)

**Cross-question:** *"If the search index is eventually consistent, how do you prevent showing rooms that are actually sold out?"*
> You don't fully prevent it — you accept it and handle it at the booking step (see Section 5's optimistic locking) with a graceful "sorry, this room was just booked" message. This is the same pattern airlines and e-commerce sites use. Trying to make search perfectly consistent would kill search latency for a rare edge case (the sliver of time between someone booking and the index catching up). It's a deliberate consistency-vs-latency tradeoff, not an oversight.

**Cross-question:** *"How do you keep Elasticsearch in sync with the source of truth (SQL DB)?"*
> Change Data Capture (CDC) — e.g., Debezium reading the DB's write-ahead log — publishes every insert/update to a Kafka topic, and a consumer updates Elasticsearch. This decouples the write path from the search index entirely; the booking service never talks to Elasticsearch directly, so a slow or down search index never blocks a booking.

---

## 5. The Hard Part: Preventing Double-Booking

This is the core interview topic for this system. There are three common strategies — know all three and their tradeoffs.

### Option A: Pessimistic Locking (SELECT FOR UPDATE)

```sql
BEGIN;
SELECT available_count FROM inventory
WHERE room_type_id = ? AND date = ?
FOR UPDATE;  -- locks this row until transaction ends

-- application checks available_count > 0
UPDATE inventory SET available_count = available_count - 1
WHERE room_type_id = ? AND date = ?;

INSERT INTO booking (...) VALUES (...);
COMMIT;
```
`FOR UPDATE` locks the row so no other transaction can read-for-update it until this one commits or rolls back.

**Pros:** Simple to reason about, guarantees correctness.
**Cons:** Under high contention (everyone trying to book the last room on a popular date), transactions queue up and wait — hurts throughput. Risk of deadlock if locks on multiple date-rows are acquired in different orders (e.g., a multi-night booking locking 5 date-rows).

**Cross-question:** *"How do you avoid deadlock when a booking spans multiple nights and locks multiple inventory rows?"*
> Always acquire locks in a consistent order — e.g., always lock date rows in ascending date order across every transaction. This eliminates circular wait, one of the four necessary conditions for deadlock (see multithreading deadlock conditions — this is the same principle applied to DB row locks instead of in-memory locks).

### Option B: Optimistic Locking (Version Number)

```sql
-- Read current state (no lock)
SELECT available_count, version FROM inventory WHERE room_type_id=? AND date=?;

-- Application checks available_count > 0, then tries to update:
UPDATE inventory
SET available_count = available_count - 1, version = version + 1
WHERE room_type_id = ? AND date = ? AND version = <version_just_read>;

-- If 0 rows affected → someone else updated it first → retry from the read
```

**Pros:** No locks held, much higher throughput under normal (low-contention) conditions.
**Cons:** Under very high contention (flash sale on a popular date), many requests retry repeatedly, wasting work — can actually perform worse than pessimistic locking at extreme contention.

**Cross-question:** *"When would you choose optimistic over pessimistic locking here?"*
> Optimistic is better when conflicts are rare relative to total requests — which is true for the vast majority of hotel rooms most of the time (there isn't usually fierce contention for a specific room on a specific date). Pessimistic locking makes more sense for a narrow set of extremely high-demand cases (e.g., last room during a major event) where you'd rather serialize requests than have them all retry and fail repeatedly.

### Option C: Database Constraint as the Final Safety Net

Regardless of A or B, add a hard constraint so even an application bug can't cause a double-book:

```sql
ALTER TABLE inventory ADD CONSTRAINT chk_available CHECK (available_count >= 0);
```
If a race condition somehow slips through application logic and would drive `available_count` negative, the database itself rejects the update. **This is the defense-in-depth principle** — never rely on application logic alone for an invariant this critical.

**Cross-question:** *"Why not just rely on your application-level locking and skip the DB constraint?"*
> Because application logic can have bugs, can be bypassed by a new code path someone adds later, or can run in multiple service instances that don't share the same in-memory lock. A DB constraint is enforced at the data layer regardless of which application code touched it — it's the last line of defense, not the primary mechanism, but it's cheap insurance against catastrophic correctness bugs.

### Option D (mention if asked about extreme scale): Distributed Lock / Queue-based Serialization

For extremely hot inventory (e.g., a viral event causing 100k people to try booking the same 10 rooms in the same second), route booking requests for that specific room+date through a **single queue/worker** (or a distributed lock via Redis/Zookeeper) so they're processed strictly one at a time, instead of hammering the DB with retries.

**Cross-question:** *"Doesn't serializing everything through one queue create a bottleneck?"*
> Yes, deliberately — but it's a *targeted* bottleneck only for the specific hot resource (that one room-date combination), not the whole system. It trades a small amount of latency for that specific booking for guaranteed correctness and avoids wasting resources on doomed retries. This only matters for the small fraction of highly-contended items; the vast majority of bookings never touch this path.

---

## 6. Booking Flow (End to End)

```
1. User clicks "Book" → Booking Service receives request
2. Booking Service checks inventory (optimistic lock read)
3. Reserve the room: decrement available_count (with version check)
   → creates Booking with status = PENDING_PAYMENT
4. Call Payment Service (external PSP like Stripe)
5a. Payment succeeds → Booking status = CONFIRMED → publish event to queue
5b. Payment fails/times out → release the room (increment available_count back),
    Booking status = CANCELLED
6. Async consumers of the "booking confirmed" event:
   → Notification Service sends confirmation email/SMS
   → Search index updater refreshes Elasticsearch
   → Analytics pipeline logs the booking
```

**Cross-question:** *"What happens if the payment step times out — did the payment actually go through or not?"*
> This is a classic distributed systems problem: you can't always know. The solution is **idempotency keys** — the booking request carries a unique idempotency key, and the Payment Service is called with that same key. If the client retries (because it didn't get a response), the PSP recognizes the same key and returns the original result instead of charging twice. On the booking side, the booking sits in `PENDING_PAYMENT` and a **reconciliation job** periodically checks with the PSP for any bookings stuck in that state beyond a timeout window, and resolves them (confirm or cancel + release inventory) based on the actual payment status.

**Cross-question:** *"Why reserve the room before payment completes instead of after?"*
> If you charge first and reserve second, there's a window where a customer paid but the room might no longer be available (someone else booked it in between) — a worse failure mode (refund a real payment) than a failed/abandoned reservation. Reserving first with a short expiry (e.g., 10–15 minute hold) means worst case is an unsold room sits "held" briefly — recoverable via a timeout job that releases unpaid holds. This is the same pattern as e-commerce checkout flows and concert ticket sales ("your seat is held for 10 minutes").

**Cross-question:** *"What releases an expired hold if the user just closes the tab?"*
> A background job (or a TTL-based mechanism, e.g., a Redis key with expiry mirroring the hold, or a scheduled sweep of `PENDING_PAYMENT` bookings older than N minutes) releases the inventory back and marks the booking as `EXPIRED`. This must be a durable, server-side timeout — never rely on the client to notify you it gave up.

---

## 7. Handling Cancellations

```sql
BEGIN;
UPDATE booking SET status = 'CANCELLED' WHERE booking_id = ?;
UPDATE inventory SET available_count = available_count + 1
WHERE room_type_id = ? AND date = ?;
COMMIT;
```
Both in the same transaction — either both happen, or neither does. If the inventory increment fails after the booking is already marked cancelled, you've "lost" a room from the available pool with no booking to explain it — a subtle bug that only shows up as slowly shrinking inventory over time.

**Cross-question:** *"How would you even detect a bug like that (inventory silently shrinking) in production?"*
> A periodic reconciliation job that recomputes `available_count` from first principles (total rooms minus confirmed, non-cancelled bookings overlapping that date) and compares it against the live counter, alerting on mismatches. This is the same "eventual audit" pattern used for financial ledgers — don't just trust the running counter forever, periodically verify it against source-of-truth data.

---

## 8. Caching Strategy

| Data | Cache? | TTL / Strategy |
|---|---|---|
| Hotel static info (name, photos, amenities) | Yes, aggressively | Long TTL (hours), invalidate on hotel update |
| Search results (popular queries) | Yes | Short TTL (seconds–minutes) |
| Room availability count | Cache cautiously, or skip | If cached, very short TTL; final check always hits DB before confirming |
| User session / auth | Yes | Redis, short-lived tokens |
| Booking confirmation | No (always live DB read) | Correctness-critical |

**Cross-question:** *"Would you ever cache the availability count shown in search results?"*
> Yes, for the *display* value shown while browsing (this is fine to be a few seconds stale — see the eventual consistency discussion in Section 4), but the actual booking confirmation step must always re-check the live database value in the transaction — never trust a cached number for the final write. Caching is for the read path's latency; the write path re-validates from source of truth regardless of what was cached.

---

## 9. Scaling Considerations

- **Search Service:** stateless, horizontally scalable behind a load balancer; Elasticsearch sharded by region/geography
- **Booking Service:** also stateless at the app layer, but the database is the real scaling bottleneck — mitigated by:
  - **Sharding the inventory/booking tables** by `hotel_id` (a booking always touches one hotel's data, so this is a clean shard key with no cross-shard transactions needed)
  - Read replicas for anything read-heavy that tolerates slight staleness
- **Database choice:** relational DB (Postgres/MySQL) for Booking and Inventory — you need real transactions and constraints here. NoSQL is a poor fit for this specific subsystem despite being tempting for "scale," because the correctness requirements (atomic decrement + constraint checks) are exactly what relational transactions are built for.

**Cross-question:** *"Why not use a NoSQL database for the booking/inventory data to get better scale?"*
> NoSQL databases generally trade away strong multi-row transactional guarantees for horizontal scale — but the double-booking problem is fundamentally a transactional correctness problem (atomically check-and-decrement + is this booking unique for this room/date). You'd end up reimplementing transaction-like guarantees in application code, badly. Sharding a relational DB by `hotel_id` gets you most of the horizontal scale benefit while keeping real ACID transactions, because bookings rarely need to span multiple hotels in one transaction.

**Cross-question:** *"What if one hotel (like a huge resort chain) gets so much traffic that its shard becomes a hotspot?"*
> Shard further by `hotel_id + room_type_id`, or add a caching/queueing layer in front of that specific hot shard (see Option D in Section 5). This is the classic hot-partition problem — the fix is usually finer-grained sharding or isolating the hot key, not resharding the entire system.

---

## 10. Reliability & Failure Handling

- **Circuit breakers** around the Payment Service call — if the PSP is down, fail fast and tell the user to retry, rather than piling up threads waiting on a dead dependency
- **Retries with exponential backoff** for transient failures (network blips), but **never retry a payment charge without an idempotency key**
- **Dead-letter queues** for async events (notification, index update) that repeatedly fail, so they don't get silently dropped
- **Health checks + auto-scaling** on the search service for traffic spikes (e.g., flash sale mornings)

**Cross-question:** *"If the Payment Service is down, should the whole booking flow fail?"*
> Yes for the payment step itself (you can't confirm a booking without payment), but the *inventory hold* should still succeed and be held for the normal timeout window, so the user isn't unfairly told "no rooms available" due to an unrelated downstream outage. Fail the specific step that's actually broken; don't cascade failure into parts of the system that are still healthy.

---

## 11. Summary of Key Tradeoffs

| Decision | Tradeoff |
|---|---|
| Split search vs. booking services | More operational complexity, but lets each scale/consistency-model independently |
| Eventually consistent search index | Occasional "sorry, just sold out" at checkout, in exchange for fast search at scale |
| Optimistic locking for most bookings | Higher throughput normally, but wasted retries under extreme contention |
| Reserve-then-charge flow | Risk of unsold "held" inventory for a few minutes, avoided refunding real payments for unavailable rooms |
| Relational DB for booking/inventory | Less "infinite scale" than NoSQL, but correctness (no double-booking) is non-negotiable here |
| Shard by hotel_id | Clean, mostly even shard key, but hot hotels need special-casing |

---

## 12. Rapid-Fire Cross-Questions (with one-line answers)

- **"How do you test for race conditions in the booking flow?"** → Load-test with concurrent requests against the same room/date in a staging environment; use chaos/fault injection to simulate slow payment responses during concurrent booking attempts.
- **"What happens during a database failover mid-transaction?"** → The in-flight transaction is lost/rolled back; the client should treat a timeout as "unknown outcome" and rely on idempotency keys + reconciliation, not assume failure.
- **"How do you handle multi-currency pricing?"** → Store canonical price in one currency (e.g., USD or hotel's local currency) and convert for display only; never store a room's price pre-converted, to avoid rounding/rate drift bugs.
- **"How would you implement 'hold for 10 minutes' without polling the DB constantly?"** → Redis key with TTL matching the hold duration; a background sweeper or Redis keyspace-notification triggers the release when it expires.
- **"How do you prevent a single user from bulk-booking all rooms via script (scalping)?"** → Rate limiting per user/IP at the API gateway, CAPTCHAs on suspicious patterns, and business-rule limits (e.g., max rooms per booking).
