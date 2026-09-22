Absolutely. Here is the **interview revision version** of the HLD. This is what I would read once before the interview rather than going through the entire detailed explanation.

# 🎟️ Event Ticket Booking / BookMyShow HLD — Quick Revision

## 1. Requirements

### Functional

1. User can **search events** by title, location and date.
2. User can **view event details** — description, metadata, venue, show timings and seat layout.
3. User can **book tickets**.

### Non-functional

* High availability for search/view.
* Low latency for read operations.
* Strong consistency for booking.
* **No double booking.**
* Scale: assume ~100M DAU if required by the interviewer.

---

# 2. High-Level Architecture

```text
Client
   ↓
API Gateway
   ↓
 ┌───────────────┬───────────────┬───────────────┐
 ↓               ↓               ↓
Search         Event          Booking
Service        Service         Service
 ↓               ↓               ↓
Elastic        Cassandra     PostgreSQL
Search                         + Redis
                                  ↓
                             Payment Service
                                  ↓
                           Payment Gateway
                                  ↓
                           Payment Success
                                  ↓
                             Booking Service
                                  ↓
                           Booking DB
                                  ↓
                                Kafka
                                  ↓
                         Notification Service
                            /    |    \
                         Email  SMS   Push
```

---

# 3. Core Entities

```text
User
Event
Venue
Screen
Seat
Show
ShowSeat
Booking
BookingSeat
```

### Most important distinction

```text
Seat
 ↓
Physical/static seat

ShowSeat
 ↓
Seat for a particular show
 ↓
AVAILABLE / LOCKED / BOOKED
```

For example:

```text
A10

Show 1 → BOOKED
Show 2 → AVAILABLE
Show 3 → LOCKED
```

---

# 4. Database Design

### Cassandra — Event DB

Stores:

```text
Events
Venues
Screens
Static seat layout
Show information
```

Why Cassandra?

* High scale
* Horizontally scalable
* Predictable query patterns
* Good for event/show discovery

---

### Elasticsearch

Used for:

```text
Search by:
- title
- location
- date
- filters
- autocomplete
```

Flow:

```text
Cassandra
   ↓
CDC
   ↓
Elasticsearch
```

Cassandra is the source of event data; Elasticsearch is the search index.

---

### PostgreSQL — Booking DB

Stores:

```text
ShowSeat
Booking
BookingSeat
```

This is the **authoritative source for booking state**.

Why PostgreSQL?

> We need strong consistency and transactional/concurrency guarantees to prevent double booking.

---

### Redis

Used for:

```text
Cache
Temporary seat locks
Frequently accessed data
```

Seat lock example:

```text
show:S1:seat:A10
       ↓
    USER123
       ↓
    TTL = 5 min
```

---

# 5. APIs

### Search

```http
GET /v1/search?q={term}&location={location}&date={date}
```

### Event details

```http
GET /v1/events/{eventId}
```

### Reserve seats

```http
POST /v1/booking/reserve
```

```json
{
  "showId": "S1",
  "seats": ["A1", "A2"]
}
```

### Confirm booking

```http
POST /v1/booking/confirm
```

### Get booking

```http
GET /v1/bookings/{bookingId}
```

---

# 6. End-to-End Flow

## Step 1 — Search

```text
Client
 ↓
API Gateway
 ↓
Search Service
 ↓
Elasticsearch
```

Returns matching events.

---

## Step 2 — View event

```text
Client
 ↓
API Gateway
 ↓
Event Service
 ↓
Cassandra
```

Returns:

```text
Event details
Venue
Show timings
Seat layout
Metadata
```

---

## Step 3 — Select seats

```text
Client
 ↓
Booking Service
 ↓
Check ShowSeat
```

Example:

```text
A10 → AVAILABLE
A11 → AVAILABLE
```

---

## Step 4 — Lock seats

```text
AVAILABLE
    ↓
  LOCKED
```

Temporary lock, e.g. 5 minutes.

Redis can maintain the temporary lock, while PostgreSQL remains the authoritative booking store.

---

# 7. Most Important Deep Dive — Two Users Book Same Seat

Suppose:

```text
A10 = AVAILABLE
```

Two users simultaneously try to book it.

### Bad approach

```text
User A → READ → AVAILABLE
User B → READ → AVAILABLE

User A → BOOKED
User B → BOOKED
```

This causes double booking.

### Correct approach

Use an atomic conditional update:

```sql
UPDATE show_seat
SET status = 'LOCKED',
    locked_by = :userId,
    locked_until = :expiry
WHERE show_id = :showId
AND seat_id = :seatId
AND status = 'AVAILABLE';
```

Result:

```text
User A → 1 row affected → SUCCESS

User B → 0 rows affected → SEAT UNAVAILABLE
```

Therefore:

> **Only one user can acquire the seat.**

Alternative approach:

```sql
SELECT ... FOR UPDATE
```

which uses pessimistic database locking.

---

# 8. Booking State

After seats are locked:

```text
Booking = PENDING
```

Then payment starts.

```text
Booking Service
      ↓
Payment Service
      ↓
Payment Gateway
```

---

# 9. Payment Success Flow

This is very important to remember:

```text
Payment Gateway
      ↓
Payment Service
      ↓
PaymentSucceeded
      ↓
Booking Service
      ↓
Booking DB
```

Booking Service updates:

```text
Booking:
PENDING → CONFIRMED

ShowSeat:
LOCKED → BOOKED
```

### Important ownership rule

> **Payment Service does not directly update Booking DB.**

Payment Service tells Booking Service that payment succeeded.

**Booking Service owns Booking DB and updates it.**

We don't need a separate Payment DB for our current interview scope.

---

# 10. Kafka + Notification

After booking is confirmed:

```text
Booking Service
      ↓
BookingConfirmed
      ↓
Kafka
      ↓
Notification Service
      ↓
 ┌────┼────┐
 ↓    ↓    ↓
Email SMS Push
```

Why Kafka?

Because notification should be asynchronous.

The booking should not wait for:

```text
Email
SMS
Push
```

to complete.

---

# 11. Failure Scenarios

### Payment fails

```text
LOCKED
   ↓
Payment FAILED
   ↓
Lock expires
   ↓
AVAILABLE
```

### User abandons payment

```text
LOCKED
   ↓
TTL expires
   ↓
AVAILABLE
```

### Payment succeeds but Booking Service crashes

Payment result must be durable/retriable.

Eventually:

```text
Payment SUCCESS
Booking PENDING
       ↓
Reconciliation / event retry
       ↓
Booking CONFIRMED
```

### User retries booking request

Use:

```http
Idempotency-Key: abc123
```

so the same request doesn't create duplicate bookings.

---

# 12. Scalability

### Search/View

Very read-heavy:

```text
Cache
+
Elasticsearch
+
Cassandra
```

### Booking

Consistency-sensitive:

```text
Booking Service
      ↓
PostgreSQL
```

### High traffic

For a blockbuster/concert:

```text
API Gateway
 ↓
Load Balancer
 ↓
Multiple service instances
 ↓
Cache
 ↓
DB
```

Potential additions:

* Rate limiting
* Queue/waiting room
* Horizontal scaling
* DB partitioning/sharding if scale requires it

---

# 13. The 5 things you absolutely need to remember

If you're short on preparation time, focus on these:

### 1. Seat vs ShowSeat

```text
Seat = physical seat
ShowSeat = seat availability for a particular show
```

### 2. Database choices

```text
Cassandra → Event data
Elasticsearch → Search
PostgreSQL → Booking/seat consistency
Redis → Cache + temporary locks
```

### 3. Double booking

```text
Two users
   ↓
Atomic conditional update
   ↓
Only one succeeds
```

### 4. Payment flow

```text
Booking
 ↓
Payment Service
 ↓
Payment Gateway
 ↓
SUCCESS
 ↓
Payment Service
 ↓
Booking Service
 ↓
Booking DB = CONFIRMED
```

### 5. Notification

```text
Booking Confirmed
       ↓
     Kafka
       ↓
Notification Service
       ↓
Email / SMS / Push
```

---

# 14. Your 45-minute interview sequence

```text
0–5 min
Requirements + NFR
        ↓
5–10 min
Entities + DB
        ↓
10–15 min
APIs
        ↓
15–23 min
HLD
        ↓
23–35 min
Concurrency + seat locking
        ↓
35–40 min
Payment + failure scenarios
        ↓
40–45 min
Scalability + follow-up questions
```

### Your one-line architecture summary

If the interviewer asks you to summarize the entire design:

> **"I use Cassandra for scalable event/show data, Elasticsearch for search, PostgreSQL as the authoritative booking and seat-inventory store, Redis for caching and temporary locks, an external payment gateway through a Payment Service, and Kafka to asynchronously notify users after a booking is confirmed."**

That is the **core story** you should be able to reproduce on the whiteboard without looking at the diagram.
