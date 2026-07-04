# System Design Interview Prep — Booking.com Focus

## How to use this guide
This is built so you can **drive** the interview, not just react to it. It has four parts:
1. A universal framework to answer any system design question
2. The core topics you must know cold (with plain explanations)
3. Booking.com-specific problems you're likely to get, fully solved
4. A bank of extra practice questions with answer skeletons

---

## PART 1: The Framework (use this in every interview)

Interviewers grade *process* more than the final diagram. Use this 5-step flow, and narrate it out loud so the interviewer sees your thinking:

### Step 1 — Clarify Requirements (3-5 min)
Ask questions before drawing anything. Split into:
- **Functional requirements**: What must the system do? (e.g., "Users can search hotels by city/date, view availability, book a room, cancel a booking")
- **Non-functional requirements**: Scale, latency, consistency needs, availability targets
- **Out of scope**: Explicitly state what you won't cover (e.g., "I'll skip payment processing details and focus on search + booking")

Good questions to ask:
- "What's the scale? DAU, requests/sec, read vs write ratio?"
- "Is strong consistency required, or is eventual consistency okay?"
- "Global or single-region?"
- "Mobile + web, or API only?"

### Step 2 — Back-of-envelope Estimation (2-3 min)
Calculate:
- QPS (queries per second) for reads and writes
- Storage needs (data size × growth rate)
- Bandwidth
- Peak vs average load (travel sites spike around holidays, flash sales)

Example: 100M users, 10% search daily → 10M searches/day → ~115 QPS average, ~1000+ QPS at peak.

### Step 3 — High-Level Design (5-10 min)
Draw boxes: Client → Load Balancer → API Gateway → Services → Databases/Cache/Queue. Keep it simple first, then add complexity.

### Step 4 — Deep Dive (15-20 min)
This is where most marks are won. The interviewer will pick 1-2 components and ask you to go deep:
- Database schema and choice (SQL vs NoSQL, why)
- How you handle a specific hard problem (e.g., double-booking, search ranking, race conditions)
- Caching strategy
- How you scale a bottleneck

### Step 5 — Address Non-Functional Concerns
- Availability (replication, failover)
- Consistency (especially for booking/inventory — this is the classic Booking.com twist)
- Fault tolerance
- Monitoring/observability

**Golden rule:** Always state trade-offs. There's rarely one right answer — say *why* you picked X over Y.

---

## PART 2: Core Topics You Must Know Cold

### 2.1 Scalability Basics
- **Vertical vs horizontal scaling**: Vertical = bigger machine (simple, limited ceiling). Horizontal = more machines (complex, near-limitless).
- **Load balancing**: Round robin, least connections, consistent hashing. L4 (transport layer, fast) vs L7 (application layer, smarter routing).
- **Stateless services**: Keep servers stateless so any request can go to any server; store session/state in a shared store (Redis, DB).

### 2.2 Databases
- **SQL vs NoSQL**: SQL (Postgres/MySQL) for strong consistency, relational data, transactions (e.g., bookings, payments). NoSQL (DynamoDB, Cassandra, MongoDB) for scale, flexible schema, high write throughput (e.g., logs, reviews, session data).
- **Indexing**: Speeds up reads, slows down writes; B-trees for range queries, hash indexes for exact lookups.
- **Sharding**: Split data across nodes by key (e.g., hotel_id, user_id). Watch for hot shards.
- **Replication**: Leader-follower for read scaling; multi-leader/multi-region for write availability but conflict risk.
- **CAP Theorem**: In a network partition, choose Consistency or Availability. Booking systems often need consistency for inventory (don't oversell rooms) but availability for search (stale results are okay).
- **ACID vs BASE**: ACID for transactional correctness (bookings, payments). BASE (eventually consistent) for high-scale, less critical data.

### 2.3 Caching
- **Where**: Client, CDN, API gateway, application (Redis/Memcached), database query cache.
- **Strategies**: Cache-aside (lazy load), write-through, write-back, TTL-based expiry.
- **Cache invalidation** ("one of the two hard problems in CS"): TTL, event-based invalidation, versioned keys.
- **Hot key problem**: A popular hotel/destination could overload a single cache node — mitigate with local caching or key splitting.

### 2.4 Messaging & Async Processing
- **Message queues** (Kafka, SQS, RabbitMQ): Decouple services, smooth traffic spikes, enable retries.
- **Use cases**: Sending booking confirmation emails, updating search index, analytics events.
- **Pub/Sub vs point-to-point**: Pub/sub for fan-out (notify multiple services of a booking event).

### 2.5 Consistency & Concurrency (critical for booking systems)
- **Race conditions**: Two users booking the last room simultaneously — classic Booking.com interview trap.
- **Solutions**:
  - Pessimistic locking (DB row lock / `SELECT FOR UPDATE`) — simple, safe, can hurt throughput.
  - Optimistic concurrency control (version number, compare-and-swap) — better throughput, needs retry logic.
  - Distributed locks (Redis Redlock, Zookeeper) — when inventory spans multiple services.
  - Atomic decrement operations in the datastore (e.g., `UPDATE inventory SET available = available - 1 WHERE available > 0`).

### 2.6 Search & Ranking
- Full-text/faceted search engines: **Elasticsearch/Solr** for hotel search by location, price, amenities, dates.
- Geospatial indexing (geohashing, quad-trees) for "hotels near me."
- Ranking pipeline: relevance score + business signals (price, reviews, sponsored placement) — often a separate ranking service.

### 2.7 Availability & Fault Tolerance
- Redundancy at every layer, no single point of failure.
- Health checks + auto-failover.
- Circuit breakers (stop calling a failing downstream service) and retries with backoff.
- Rate limiting (token bucket, leaky bucket) to protect services from overload/abuse (e.g., scraping bots on search).

### 2.8 CDN & Static Content
- Hotel images, static assets served via CDN close to users, reducing latency and origin load.

### 2.9 API Design
- REST vs gRPC vs GraphQL trade-offs.
- Idempotency — crucial for booking APIs (a retried "confirm booking" request shouldn't create two bookings). Use idempotency keys.

### 2.10 Security Basics
- AuthN/AuthZ (OAuth2/JWT), rate limiting, encrypting PII (especially payment/travel data), GDPR considerations (Booking.com is EU-based — data residency and deletion rights may come up).

---

## PART 3: Booking.com-Specific Problems (Fully Solved)

Booking.com's core domain is **travel inventory + search + reservations at massive, spiky scale, with strict correctness requirements on inventory**. Expect questions built around these themes.

---

### Q1: "Design a Hotel Booking System" (the flagship question)

**Step 1 — Requirements**
Functional:
- Search hotels by location, dates, guests, filters
- View hotel details and room availability
- Book a room (with payment)
- Cancel/modify a booking
- Prevent double-booking

Non-functional:
- High availability for search (okay to be eventually consistent)
- Strong consistency for the actual booking/inventory decrement (no overselling)
- Handle huge read:write skew (millions search, thousands book)
- Global scale, low latency (<200ms search)

**Step 2 — Estimation**
- Assume 50M DAU, 20% search → 10M searches/day ≈ 115 QPS avg, ~5-10x at peaks (holidays) → ~1000 QPS.
- Bookings: much lower volume, maybe 1-2% of searches convert → ~100K bookings/day ≈ ~1-2 QPS avg but needs to be *correct*, not just fast.
- This asymmetry (huge read scale vs small-but-critical write scale) is the key insight to state out loud — it justifies separating the **search path** from the **booking/inventory path** architecturally.

**Step 3 — High-Level Architecture**
```
Client (Web/Mobile)
   |
Load Balancer / API Gateway
   |
   +-- Search Service --> Elasticsearch (denormalized hotel+room+price index)
   |         |
   |     Cache (Redis) for popular destinations/dates
   |
   +-- Booking Service --> Inventory DB (SQL, strongly consistent)
   |         |
   |     Payment Service (external/internal, idempotent)
   |         |
   |     Message Queue (Kafka) --> Notification Service, Analytics, Search Index Updater
   |
   +-- Hotel/Partner Service (manages listings, rates, availability updates from hotels)
```

**Step 4 — Deep Dive: Preventing Double-Booking (the question they most want to hear)**
- Inventory lives in a relational DB (SQL) per hotel/room-type/date — this is the source of truth, not the search index.
- Booking flow:
  1. User clicks "Book" → Booking Service starts a transaction.
  2. `UPDATE room_inventory SET available = available - 1 WHERE room_id = ? AND date = ? AND available > 0`
  3. If 0 rows affected → room is sold out → return failure immediately (fail fast, don't queue).
  4. If success → create booking record with status `PENDING_PAYMENT`, generate idempotency key, call Payment Service.
  5. On payment success → mark booking `CONFIRMED`; on failure/timeout → release inventory back (compensating transaction) — this is the **Saga pattern** for distributed transactions across booking + payment services.
- Idempotency: client sends an idempotency key with the booking request so retries (e.g., flaky mobile network) don't double-book or double-charge.
- Why not just lock at the search layer? Because search index (Elasticsearch) is a *read-optimized, eventually-consistent copy* — never treat it as ground truth for a scarce resource like room inventory.

**Deep Dive: Search Architecture**
- Hotel + room + rate data denormalized into Elasticsearch documents, updated via events from the Hotel/Partner Service (through Kafka) whenever prices/availability change.
- Geospatial query for "hotels in Paris" using geohash/bounding box.
- Cache popular queries (top destinations, weekend dates) in Redis with short TTL (minutes), since prices change often but staleness of a few minutes is acceptable for browsing.
- At booking time, always re-validate against the real inventory DB — never trust the cached/search price/availability blindly (re-confirm at checkout, this is the "price may have changed" experience you've likely seen on Booking.com).

**Deep Dive: Scaling reads vs writes**
- Search: horizontally scaled Elasticsearch cluster, CDN for static assets, Redis cache, read replicas.
- Booking: vertically consistent (few writes, must be correct) — shard inventory DB by hotel_id, since bookings for different hotels are independent.

**Trade-offs to mention out loud:**
- Eventual consistency in search (fine — worst case a user sees a room that's just sold out, and finds out at checkout) vs strong consistency in the inventory DB (mandatory — money and legal/contract implications of overselling).
- Saga pattern vs 2PC (two-phase commit) for the booking-payment flow: Saga scales better and doesn't hold locks across services, at the cost of temporary inconsistency and needing compensating actions.

---

### Q2: "Design the Hotel Search Autocomplete / Typeahead"

**Key points to hit:**
- Requirements: <100ms latency, prefix matching, ranked by popularity, handle typos.
- Data structure: **Trie** (prefix tree) for exact prefix match, or precomputed top-N suggestions per prefix stored in Redis (simpler and faster to implement at scale than a live trie).
- Ranking signal: search volume/popularity per city, personalization (recent searches), typo tolerance via Levenshtein distance or n-gram matching (or just delegate to Elasticsearch's "completion suggester").
- Scale: read-heavy, cache aggressively (results per prefix rarely change second-to-second) — precompute and refresh periodically (e.g., every few minutes/hours) rather than computing live.
- Trade-off: Trie in memory is fast but hard to update live at scale across regions; precomputed cache is simpler operationally.

---

### Q3: "Design the Review System (like Booking.com guest reviews)"

**Key points:**
- Requirements: only verified guests (who actually stayed) can review; reviews tied to a booking ID; aggregate rating shown on hotel page; handle spam/fraud.
- Data model: `reviews` table (review_id, booking_id, hotel_id, user_id, rating, text, created_at) in a scalable store (could be NoSQL given volume + simple access patterns).
- Aggregation: don't recompute average rating on every read — maintain a running aggregate (count + sum) updated asynchronously via event (Kafka) when a review is added, and cache the hotel's rating.
- Fraud/spam: rate limit reviews per user, verify against actual completed booking, possibly async ML moderation pipeline before publishing.
- Scale: reviews are write-light, read-heavy (shown on every hotel page) — cache aggressively, similar to search.

---

### Q4: "Design a Rate Limiter for the Booking.com Public API"

**Key points:**
- Algorithms: Token Bucket (allows bursts, common choice) vs Sliding Window Log/Counter (more accurate, more memory) vs Fixed Window (simple, has edge burst issue).
- Where: at API Gateway layer, per API key/user/IP.
- Distributed rate limiting: use Redis with atomic INCR + TTL, or Redis + Lua script for atomicity across multiple gateway nodes.
- Trade-off: Token bucket is a good default — memory-efficient, allows reasonable burstiness, easy to reason about.

---

### Q5: "Design the Notification System" (booking confirmations, reminders, price-drop alerts)

**Key points:**
- Event-driven: booking events published to Kafka → Notification Service consumes → sends via Email/SMS/Push providers.
- Retry with backoff + dead-letter queue for failed sends.
- Template service for localization (Booking.com supports many languages/currencies — mention this, it's a real product wrinkle).
- User preference store (opt-in/opt-out per channel) checked before sending — also a GDPR/compliance point.
- Scale: fan-out heavy during sales/holidays — queue-based buffering to avoid overwhelming email/SMS providers, rate limit outbound per provider.

---

### Q6: "How would you handle a flash sale / massive traffic spike (e.g., Black Friday deals)?"

**Key points:**
- Pre-scale infrastructure ahead of known events (predictable spike) using autoscaling + capacity planning.
- Queue-based load leveling: accept requests into a queue instead of processing synchronously, protecting the inventory DB from being hammered.
- Aggressive caching of static/semi-static content (deal pages) via CDN.
- Graceful degradation: show cached/slightly stale prices under extreme load rather than failing entirely; feature-flag off non-critical features (e.g., personalized recommendations) to shed load.
- Circuit breakers around payment/inventory services to avoid cascading failures.

---

## PART 4: Extra Practice Bank (skeleton answers — use Part 1's framework to expand)

| Question | Core idea to lead with |
|---|---|
| Design a URL shortener | Base62 encoding + counter/hash, key-value store, cache popular redirects |
| Design a distributed cache | Consistent hashing for node distribution, LRU eviction, replication for fault tolerance |
| Design a news feed / recommendations for "Hotels you might like" | Fan-out on write vs read, hybrid approach, ML ranking service, feature store |
| Design a payment system | Idempotency keys, Saga/2PC for distributed transactions, ledger table (append-only, source of truth), reconciliation jobs |
| Design a distributed ID generator (for booking IDs) | Snowflake-style IDs (timestamp + machine ID + sequence) — avoids central bottleneck, roughly sortable |
| Design a chat/support system (guest-host messaging) | WebSockets for real-time, message queue for offline delivery, read-receipt state, message store sharded by conversation ID |
| Design currency conversion for global pricing | Central exchange-rate service updated periodically, cache rates, store prices in a base currency + convert at display time, be careful about rate staleness at checkout |
| Design an inventory management system for hotel partners | Similar to Q1's inventory DB; focus on partner-side APIs, bulk update handling, conflict resolution when partner updates rates via multiple channels (extranet + channel manager API) |

---

## PART 5: How to "Drive" the Interview

- **Narrate your framework** at the start: "I'll clarify requirements, do rough estimation, sketch a high-level design, then we can deep-dive wherever you'd like." This signals structure and invites the interviewer to steer.
- **Ask which area to go deeper on** after the high-level design: "I can go deeper into the search/ranking pipeline, the booking consistency model, or scaling the database — which would be most useful?" This hands control back to them while showing you have depth in multiple areas.
- **State trade-offs explicitly**, don't just pick one solution silently. Interviewers are listening for *reasoning*, not memorized diagrams.
- **Tie back to Booking.com's actual domain constraints**: massive read/write asymmetry, global multi-currency/multi-language, strict correctness on inventory, partner (hotel) side APIs, and marketplace dynamics (price and availability come from third parties, not just internal data). Mentioning these shows you understand their business, not just generic system design.
- If you get stuck, think out loud: "The bottleneck here would be X, so I'd consider Y or Z, and I'd lean toward Y because..." — this is more valuable than silence or a guess with no reasoning.

---

## Quick Reference Cheat Sheet (last-minute review)

- **CAP**: pick 2 of Consistency, Availability, Partition tolerance — partition tolerance is non-negotiable in distributed systems, so really it's C vs A.
- **Booking's double-booking problem** → atomic DB decrement + idempotency key + Saga pattern with payment.
- **Search vs booking split**: search = eventually consistent, cached, Elasticsearch; booking = strongly consistent, SQL, source of truth.
- **Caching**: cache-aside is the default safe choice; watch for hot keys and stale data at checkout.
- **Scaling reads**: replicas + cache + CDN. **Scaling writes**: sharding + queues + async processing.
- **Idempotency** matters anywhere money or one-time actions are involved.
- **Always state trade-offs** — there is no single "correct" architecture.

Good luck — you've got a strong domain angle here (Booking.com is fundamentally an inventory + search + marketplace problem at scale), so lean into that in every answer.
