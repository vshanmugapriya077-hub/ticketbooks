# TicketFlow — Multi-Tenant Event & Ticket Reservation System

[![Node.js](https://img.shields.io/badge/Node.js-v24-green.svg)](https://nodejs.org/)
[![Database](https://img.shields.io/badge/Database-PostgreSQL%20%7C%20SQLite-blue.svg)](https://www.postgresql.org/)
[![Caching](https://img.shields.io/badge/Cache-Redis%20%7C%20In--Memory-red.svg)](https://redis.io/)
[![Concurrency](https://img.shields.io/badge/Concurrency-Zero%20Double--Booking-brightgreen.svg)]()
[![License](https://img.shields.io/badge/License-MIT-purple.svg)]()

> A production-grade multi-tenant event ticketing platform featuring **Role-Based Access Control (RBAC)**, a **10-minute temporary reservation hold engine**, **atomic concurrency locking** to eliminate double-booking, **Redis caching**, **asynchronous payment simulation**, and **real-time analytics dashboards**.

---

## 📋 Evaluation Deliverables Checklist

| Deliverable Required | Location in Repo | Description |
| :--- | :--- | :--- |
| **1. ER Diagram & Data Model** | [Section Below](#-entity-relationship-er-diagram) & [`docs/ERD.md`](docs/ERD.md) | Visual Mermaid schema of Users, Events, Tiers, Holds, Orders, Tickets, Transactions. |
| **2. Postman Collection** | [`postman/Multi-Tenant-Event-Ticket-System.postman_collection.json`](postman/Multi-Tenant-Event-Ticket-System.postman_collection.json) | 8 test folders with edge cases (409 Conflict, 410 Expired Hold, Webhooks, RBAC). |
| **3. Concurrency Stress Test** | [`tests/concurrency-test.js`](tests/concurrency-test.js) | 50 simultaneous parallel requests tested on 5 seats — 0 double bookings verified. |
| **4. Cloud Deployment Setup** | [`render.yaml`](render.yaml), [`docker-compose.yml`](docker-compose.yml), [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) | Ready for 1-click deployment on Render, Railway, or Docker. |

---

## 🔑 Demo Access & Live Credentials

Use the **1-Click Role Switcher** buttons in the navigation bar to test all three roles:

| Role | Email | Password | Primary Capabilities |
| :--- | :--- | :--- | :--- |
| **Attendee** | `attendee@user.com` | `attendeepass123` | Select tiers, lock seats for 10 minutes with live countdown, complete mock payment, print scannable ticket vouchers. |
| **Organizer** | `organizer@musicfest.com` | `organizerpass123` | Create multi-tier events, set capacities & cut-off dates, view live revenue charts & conversion funnels. |
| **Admin** | `admin@ticketflow.com` | `adminpassword123` | Platform oversight, monitor Redis cache and background cron status, trigger manual expired hold sweeps. |

---

## 📊 Entity-Relationship (ER) Diagram

GitHub renders this diagram natively:

```mermaid
erDiagram
    USERS ||--o{ EVENTS : "organizes / owns"
    USERS ||--o{ TICKET_HOLDS : "holds"
    USERS ||--o{ ORDERS : "places"
    USERS ||--o{ TICKETS : "holds individual pass"

    EVENTS ||--|{ TICKET_TIERS : "defines pricing tiers"
    EVENTS ||--o{ TICKET_HOLDS : "contains reservations"
    EVENTS ||--o{ ORDERS : "generates"
    EVENTS ||--o{ TICKETS : "issues"
    EVENTS ||--|| EVENT_ANALYTICS : "tracks metrics"

    TICKET_TIERS ||--o{ TICKET_HOLDS : "locks seats temporarily"
    TICKET_TIERS ||--o{ ORDERS : "specifies purchase tier"
    TICKET_TIERS ||--o{ TICKETS : "grants access privileges"

    TICKET_HOLDS ||--o| ORDERS : "converts into (within 10 mins)"

    ORDERS ||--|{ TICKETS : "issues N unique tickets"
    ORDERS ||--|{ TRANSACTIONS : "settles via payment gateway"

    USERS {
        text id PK
        text email UK "Unique login email"
        text password_hash "Bcrypt hash"
        text name "Full Name"
        text role "ADMIN | ORGANIZER | ATTENDEE"
        text tenant_id "Multi-tenant tenant identifier"
    }

    EVENTS {
        text id PK
        text organizer_id FK "References USERS(id)"
        text tenant_id "Tenant partition key"
        text title "Event name"
        text description "Full description"
        text category "Music | Tech | Gaming | Arts"
        text venue "Venue name"
        text location "City, State / Country"
        text event_date "ISO 8601"
        text sales_cut_off_date "Ticket sales closing deadline"
        text status "DRAFT | PUBLISHED | CANCELLED"
    }

    TICKET_TIERS {
        text id PK
        text event_id FK "References EVENTS(id)"
        text name "VIP | General Admission | Early Bird"
        real price "Cost per ticket"
        integer total_capacity "Hard limit on total available seats"
        text perks "Included benefits"
    }

    TICKET_HOLDS {
        text id PK
        text user_id FK "References USERS(id)"
        text event_id FK "References EVENTS(id)"
        text tier_id FK "References TICKET_TIERS(id)"
        integer quantity "Number of tickets held"
        text status "ACTIVE | CONFIRMED | EXPIRED | CANCELLED"
        text session_token UK "Cryptographic reservation token"
        text expires_at "created_at + 10 Minutes"
    }

    ORDERS {
        text id PK
        text user_id FK "References USERS(id)"
        text event_id FK "References EVENTS(id)"
        text hold_id FK "References TICKET_HOLDS(id)"
        text tier_id FK "References TICKET_TIERS(id)"
        integer quantity "Number of tickets ordered"
        real total_amount "Total charged in currency"
        text status "PENDING_PAYMENT | CONFIRMED | FAILED"
        text customer_name "Purchaser name"
        text customer_email "Purchaser email"
    }

    TICKETS {
        text id PK
        text order_id FK "References ORDERS(id)"
        text tier_id FK "References TICKET_TIERS(id)"
        text event_id FK "References EVENTS(id)"
        text user_id FK "References USERS(id)"
        text ticket_code UK "Unique scannable code e.g. TF-VIP-982143"
        text status "VALID | CHECKED_IN | CANCELLED"
    }

    TRANSACTIONS {
        text id PK
        text order_id FK "References ORDERS(id)"
        real amount "Amount processed"
        text gateway_transaction_id "Provider transaction reference"
        text payment_method "MOCK_CARD"
        text status "INITIATED | PROCESSING | SUCCESS | FAILED"
        text error_message "Failure reason if declined"
    }

    EVENT_ANALYTICS {
        text event_id PK "References EVENTS(id)"
        integer page_views "Total page impressions"
        integer reservations_count "Total 10-min holds initiated"
        integer completed_checkouts "Total successful ticket orders"
    }
```

---

## 🔒 Concurrency Invariant & Double-Booking Prevention

### The Math Invariant
For any ticket tier $T$ at any given timestamp $t$:

$$\text{SoldTickets}(T) + \text{ActiveUnexpiredHolds}(T, t) \le \text{TotalCapacity}(T)$$

### Prevention Strategy
1. **Per-Tier Distributed / Atomic Mutex Lock**:
   - Acquire `lock:tier:{tierId}` using Redis `SET key token NX PX 5000` (or high-speed memory mutex).
2. **Atomic Write Transaction**:
   - In PostgreSQL: `SELECT * FROM ticket_tiers WHERE id = $1 FOR UPDATE`
   - In SQLite: `BEGIN IMMEDIATE` (enforces serialized write locking in WAL mode).
3. **Availability Validation**:
   - Available capacity is calculated as:
     $$\text{Available} = \text{total\_capacity} - (\text{valid\_sold} + \text{active\_holds})$$
   - If $\text{requested\_qty} > \text{Available}$, the transaction aborts with **HTTP 409 Conflict**.
   - If sufficient, insert hold with `expires_at = now() + 10 minutes`.
4. **Automatic Hold Release Worker**:
   - Background worker runs every 15 seconds:
     ```sql
     UPDATE ticket_holds SET status = 'EXPIRED' WHERE status = 'ACTIVE' AND expires_at <= NOW()
     ```
   - Invalidates Redis cache, returning seats to the available pool instantly.

---

## 🧪 Concurrency Stress Test Results

Run the automated concurrency stress test:
```bash
npm run test:concurrency
```

### Verified Test Output:
```text
====================================================
🧪 RUNNING CONCURRENCY & DOUBLE-BOOKING STRESS TEST
====================================================
[Setup] Created test tier with strict total_capacity = 5
[Stress Test] Firing 50 simultaneous booking requests for 1 ticket each...

📊 Concurrency Results:
   ✅ Successful holds: 5
   🚫 Blocked by lock/capacity (409 Conflict): 45
   ⚠️ Unexpected errors: 0

🎯 CONCURRENCY SAFETY VERIFIED: ZERO DOUBLE-BOOKINGS OCCURRED!

💳 Testing Checkout of a valid active hold...
   ✅ Order confirmed with ticket code: TF-ULT-2AE636-2PA9

⏳ Testing Expired Hold Edge Case...
   🧹 Worker sweep released: 1 expired holds.
   ✅ Correctly rejected checkout on expired hold: HTTP 410 Gone

🔄 Testing Re-reservation of Released Capacity...
   ✅ Successfully reserved freed ticket with new hold ID!
```

---

## 🚀 Quick Start (Local Setup)

Runs with zero external dependencies (Node.js v22+ built-in SQLite & in-memory cache fallback):

```bash
# 1. Clone repository
git clone <your-repo-url>
cd event-ticket-reservation-system

# 2. Install dependencies
npm install

# 3. Build frontend
npm run build:client

# 4. Start Server
npm start
```

Visit: **`http://localhost:5000`**

To run tests:
```bash
npm test
```

---

## 📬 Postman API Collection

Import the included file into Postman:  
📂 **`postman/Multi-Tenant-Event-Ticket-System.postman_collection.json`**

### Pre-Configured Test Folders:
1. **`01. Authentication & RBAC`**: Admin, Organizer, Attendee logins, token injection.
2. **`02. Event Management`**: Create event with tiers, list cached events, 403 Attendee guard.
3. **`03. Reservation Engine`**: Place 10-minute hold, verify live countdown.
4. **`04. Concurrency & Edge Cases`**:
   - Exceed capacity $\rightarrow$ `409 Conflict`
   - Checkout expired hold $\rightarrow$ `410 Gone`
   - Non-existent hold $\rightarrow$ `404 Not Found`
5. **`05. Payment Simulation`**:
   - Card ending in `4242` $\rightarrow$ `201 Approved`
   - Card ending in `0002` $\rightarrow$ `402 Declined`
   - Asynchronous delay & Webhook callbacks
6. **`06. Attendee Experience`**: View my bookings & scannable QR ticket codes.
7. **`07. Analytics Dashboard`**: Organizer revenue breakdown, conversion rate.
8. **`08. Admin Platform Controls`**: Infrastructure stats, tenant volume, manual hold sweep.

---

## ☁️ Cloud Deployment

Detailed step-by-step instructions are in [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md).

### Option 1: Render (1-Click Blueprint)
1. Fork or push this repository to GitHub.
2. Go to **Render Dashboard** $\rightarrow$ **New +** $\rightarrow$ **Blueprint**.
3. Select this repository. Render automatically reads [`render.yaml`](render.yaml) and provisions:
   - Web service (`Node.js`)
   - PostgreSQL 16 database
   - Redis service
4. Click **Apply**.

### Option 2: Railway
1. Create new project from GitHub repo.
2. Add PostgreSQL and Redis plugins.
3. Railway auto-detects [`railway.json`](railway.json) and [`Procfile`](Procfile) and deploys.

### Option 3: Docker Compose
```bash
docker compose up -d --build
```
Runs Web App on port `5000`, PostgreSQL on `5432`, and Redis on `6379`.

---

## 🛡️ License
MIT License. Created for the Multi-Tenant Event & Ticket Reservation System assignment.

