# AI Interview Platform - Backend Architecture Guide

## Table of Contents
1. [Backend Architecture Overview](#backend-architecture-overview)
2. [Request Flow](#request-flow)
3. [API Routes & Server Actions](#api-routes--server-actions)
4. [Database Architecture](#database-architecture)
5. [Authentication System](#authentication-system)
6. [AI Integration](#ai-integration)
7. [Interview Questions](#interview-questions)

---

## Backend Architecture Overview

### Tech Stack
```
Frontend: Next.js 16, React 19, TailwindCSS
Backend: Next.js (Server Actions + API Routes)
Database: PostgreSQL (Supabase)
ORM: Prisma
Authentication: Clerk
AI: Google Gemini 2.5 Flash
Video: Stream.io
Email: Resend
Rate Limiting: Arcjet
```

### Architecture Diagram
```
User Browser
    ↓
Next.js Frontend (React Components)
    ↓
Server Actions ("use server") / API Routes
    ↓
Prisma ORM
    ↓
PostgreSQL (Supabase)
    ↓
Stream.io (Video Calls)
    ↓
Gemini API (AI Questions)
```

### Key Principles
- **Server-Centric**: Most business logic runs on server (security)
- **Transactional Integrity**: Database transactions ensure consistency
- **Rate Limiting**: Arcjet protects against abuse
- **Authentication-First**: Clerk middleware on all protected routes
- **Single Responsibility**: Each action handles one business operation

---

## Request Flow

### Complete Request Journey: From Booking a Slot

```
1. FRONTEND → Interviewee clicks "Book Slot" button
   └─ Component: InterviewerCard.jsx or SlotPicker
   └─ Calls: bookSlot({ interviewerId, startTime, endTime })

2. NETWORK → Request sent to server (marked with "use server")
   └─ Serialized as JSON
   └─ Sent to Next.js server

3. AUTHENTICATION LAYER
   └─ Clerk middleware intercepts request
   └─ Validates session token from browser cookies
   └─ Extracts user identity and permissions
   └─ If invalid → Request rejected (401 Unauthorized)

4. SERVER ACTION: bookSlot() executes
   └─ File: actions/booking.js
   └─ Access: currentUser() from Clerk
   └─ Gets Clerk user ID

5. RATE LIMITING CHECK
   └─ Arcjet checks: "Has this user made 5+ booking attempts in 1 hour?"
   └─ Uses userId as fingerprint (not IP, for SPA consistency)
   └─ If limit exceeded → Error thrown

6. DATABASE QUERIES (Prisma)
   └─ Query 1: Find interviewee user record by Clerk ID
   └─ Query 2: Find interviewer by ID (verify exists and role is INTERVIEWER)
   └─ Query 3: Check for scheduling conflicts
   └─ Query 4: Verify interviewee has enough credits

7. EXTERNAL SERVICE: Stream.io Setup
   └─ Create video call session
   └─ Register both users with Stream
   └─ Configure recording, screensharing, transcription settings
   └─ Generate stream call ID

8. DATABASE TRANSACTION (Atomic)
   └─ BEGIN TRANSACTION
   └─ Create Booking record
   └─ Create CreditTransaction record
   └─ Deduct credits from interviewee
   └─ Add credit balance to interviewer
   └─ COMMIT (all succeed or all fail)

9. CACHE REVALIDATION
   └─ Revalidate Next.js cache for updated pages
   └─ Affects: /interviewers/[id], /dashboard

10. RESPONSE → Return to frontend
    └─ Success: { success: true, bookingId, streamCallId }
    └─ Error: Throw with error message

11. FRONTEND HANDLING
    └─ Show success toast/modal
    └─ Redirect to call page
    └─ Or display error message
```

### Alternative Flow: API Webhooks
```
Stream.io Event (e.g., call.ended)
    ↓
Webhook POST to /api/webhooks/stream
    ↓
Verify webhook signature
    ↓
Extract event data
    ↓
Update database (e.g., mark booking as COMPLETED)
    ↓
Trigger Gemini for feedback generation
    ↓
Send email notification via Resend
```

---

## API Routes & Server Actions

### Server Actions (Primary Pattern)

Server actions are functions marked with `"use server"` that run on the server. They're more secure than API routes because:
- Automatic CORS handling
- Built-in serialization
- Token automatically included
- Smaller bundle size (code stays on server)

**Action Files:**

#### 1. **user.js** - User Profile Management
```javascript
getCurrentUser()
  ├─ Gets current Clerk user
  ├─ Returns: User role, name, title, company, image
  └─ Used by: Every page to check user status
```

#### 2. **booking.js** - Booking Management
```javascript
getInterviewerProfile(interviewerId)
  ├─ Fetches full interviewer details
  ├─ Includes: availabilities, scheduled bookings, rates
  └─ Used by: Interviewer detail pages

bookSlot({ interviewerId, startTime, endTime })
  ├─ Rate limit check (5 bookings/hour)
  ├─ Verify credits & availability
  ├─ Create Stream.io call
  ├─ TRANSACTION: Create booking + transactions
  └─ Returns: { success, bookingId, streamCallId }
```

#### 3. **dashboard.js** - Interviewer Dashboard
```javascript
setAvailability({ startTime, endTime })
  ├─ Save/update interviewer's available time slot
  ├─ Validates: start < end
  └─ Returns: success status

getAvailability()
  ├─ Fetch interviewer's current availability
  └─ Returns: startTime, endTime

getInterviewerAppointments()
  ├─ Fetch all bookings for interviewer
  ├─ Include: interviewee details, feedback
  └─ Ordered by: startTime DESC

getInterviewerStats()
  ├─ Calculate: total earned, completion count
  ├─ Current credit balance
  └─ Used in: Earnings section

requestWithdrawal({ credits, paymentMethod, paymentDetail })
  ├─ Rate limit: 3 requests/hour
  ├─ Create Payout record
  ├─ Send email to admin
  └─ Returns: payout ID

getWithdrawalHistory()
  ├─ Fetch all payout requests
  └─ Returns: status, amounts, dates
```

#### 4. **appointments.js** - Interviewee Appointments
```javascript
getIntervieweeAppointments()
  ├─ Fetch all bookings for current user
  ├─ Include: interviewer details, feedback
  └─ Ordered by: startTime DESC
```

#### 5. **explore.js** - Interviewer Discovery
```javascript
getInterviewers()
  ├─ Fetch all active interviewers
  ├─ Include: categories, bio, rating, availabilities
  └─ Used by: Explore page
```

#### 6. **onboarding.js** - User Setup
```javascript
completeOnboarding(data)
  ├─ Update user role (INTERVIEWER or INTERVIEWEE)
  ├─ Save: title, company, bio, yearsExp, categories
  └─ Returns: updated user
```

#### 7. **aiQuestions.jsx** - AI Integration
```javascript
generateInterviewQuestions({ category })
  ├─ Call Gemini 2.5 Flash API
  ├─ Send category-specific prompt
  ├─ Parse JSON response
  └─ Returns: [{ question, answer }, ...]
```

#### 8. **call.js** - Video Call Data
```javascript
getCallData(callId)
  ├─ Fetch booking details
  ├─ Include: participants, recording URL
  └─ Used by: Call page
```

#### 9. **payout.js** - Admin Payout Approval
```javascript
approvePayout({ payoutId, adminPassword })
  ├─ Verify admin password
  ├─ Update payout status to PROCESSED
  ├─ Send confirmation email
  └─ Used by: Admin dashboard
```

### API Routes (Used Minimally)

Currently only used for webhooks:
```
/api/webhooks/stream
  ├─ Receives POST from Stream.io when call ends
  ├─ Verifies webhook signature
  ├─ Triggers feedback generation
  └─ Updates booking status
```

---

## Database Architecture

### Connection Pattern

**Prisma Configuration:**
```
Generator: @prisma/adapter-pg
Provider: postgresql
Adapter: PrismaPg (PostgreSQL-specific adapter)
Connection: pg Pool (PostgreSQL driver)

Pool Settings:
├─ DATABASE_URL: Supabase transaction-mode pooler (6543)
│  └─ Used for: Regular queries (95% of traffic)
│  └─ Faster, connection pooling
│
└─ DIRECT_URL: Supabase session-mode pooler (5432)
   └─ Used for: Database migrations
   └─ One-per-session connection
```

**Prisma Client Initialization:**
```javascript
// File: lib/prisma.js
const pool = new Pool({ connectionString: process.env.DATABASE_URL })
const adapter = new PrismaPg(pool)
const prismaClient = new PrismaClient({ adapter })

// Singleton pattern for dev environment
if (NODE_ENV !== 'production') {
  globalThis.prisma = prismaClient  // Prevent multiple instances
}
```

### Why This Architecture?

1. **Connection Pooling**: Prevents exhausting database connections
2. **PrismaPg Adapter**: Native PostgreSQL support vs. Node.js driver
3. **Separate URLs**: Migrations need session mode; queries use transaction mode
4. **Singleton in Dev**: Hot reload would create multiple clients without this

### Database Calls Pattern

```javascript
// Example: Fetch user with transactions
const user = await db.user.findUnique({
  where: { clerkUserId: userId },
  include: {
    bookingsAsInterviewee: {
      where: { status: "COMPLETED" },
      include: { feedback: true }
    },
    transactions: {
      orderBy: { createdAt: "desc" }
    }
  }
})

// Example: Transaction (booking creates 3 related records)
const result = await db.$transaction(async (tx) => {
  const booking = await tx.booking.create({ data: {...} })
  await tx.creditTransaction.create({ data: {...} })
  await tx.user.update({ data: {...} })
  return booking
})
// ↑ All succeed or all fail
```

---

## Complete Database Schema

### Table: User
**Purpose:** Core user account for both interviewees and interviewers

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `clerkUserId` | String | Unique reference to Clerk auth |
| `email` | String | User email (unique) |
| `name` | String | Display name |
| `imageUrl` | String | Profile picture URL |
| `role` | Enum | `UNASSIGNED`, `INTERVIEWEE`, or `INTERVIEWER` |
| `createdAt` | DateTime | Account creation timestamp |
| `updatedAt` | DateTime | Last profile update |
| `credits` | Int | Available credits for interviewees (resets monthly) |
| `currentPlan` | String | Subscription tier: `free`, `starter`, `pro` |
| `creditsLastAllocatedAt` | DateTime | Last monthly credit allocation date |
| `bio` | String | Profile bio (interviewers) |
| `title` | String | Job title (interviewers) |
| `company` | String | Company name (interviewers) |
| `yearsExp` | Int | Years of experience (interviewers) |
| `creditRate` | Int | Cost in credits per booking (interviewers, default 10) |
| `creditBalance` | Int | Earned credits available for payout (interviewers) |

**Relationships:**
- One-to-Many: `categories` → InterviewCategory (many)
- One-to-Many: `availabilities` → Availability (many)
- One-to-Many: `bookingsAsInterviewee` → Booking (many)
- One-to-Many: `bookingsAsInterviewer` → Booking (many)
- One-to-Many: `transactions` → CreditTransaction (many)
- One-to-Many: `payouts` → Payout (many)

**Indexes:**
- `clerkUserId` (UNIQUE)
- `email` (UNIQUE)

**Sample Data:**
```
{
  id: "uuid-123",
  clerkUserId: "clerk_user_123",
  email: "john@example.com",
  name: "John Doe",
  role: "INTERVIEWER",
  credits: 0,
  creditRate: 10,
  creditBalance: 150,
  yearsExp: 5,
  title: "Senior Frontend Engineer",
  company: "TechCorp"
}
```

---

### Table: Availability
**Purpose:** Time slots when interviewers are available for interviews

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `interviewerId` | String | Foreign key to User |
| `startTime` | DateTime | Slot start time (e.g., 9:00 AM) |
| `endTime` | DateTime | Slot end time (e.g., 5:00 PM) |
| `status` | Enum | `AVAILABLE`, `BOOKED`, `BLOCKED` |

**Relationships:**
- Many-to-One: `interviewer` ← User

**Indexes:**
- Composite: `(interviewerId, startTime)` — Used to find availability slots quickly

**How Slots Work:**
```
Stored: startTime=2026-03-25 09:00, endTime=2026-03-25 17:00
Frontend generates: 45-min slots
├─ 9:00 - 9:45
├─ 9:45 - 10:30
├─ ...
└─ 16:15 - 17:00

Each slot is checked against Booking table for conflicts
```

---

### Table: Booking
**Purpose:** Scheduled interviews between interviewee and interviewer

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `intervieweeId` | String | Foreign key to User (interviewee) |
| `interviewerId` | String | Foreign key to User (interviewer) |
| `startTime` | DateTime | Interview start time |
| `endTime` | DateTime | Interview end time |
| `status` | Enum | `SCHEDULED`, `COMPLETED`, `CANCELLED` |
| `creditsCharged` | Int | Credits deducted from interviewee |
| `streamCallId` | String | Unique reference to Stream.io video call (UNIQUE) |
| `recordingUrl` | String | URL to recorded interview video |
| `createdAt` | DateTime | Booking creation timestamp |
| `updatedAt` | DateTime | Last status update |

**Relationships:**
- Many-to-One: `interviewee` ← User
- Many-to-One: `interviewer` ← User
- One-to-Many: `transactions` → CreditTransaction (many)
- One-to-One: `feedback` ← Feedback

**Indexes:**
- `(status, createdAt)` — Find recent bookings by status
- `(interviewerId, status)` — Find interviewer's bookings
- `(intervieweeId, status)` — Find interviewee's bookings
- `streamCallId` (UNIQUE)

**Lifecycle:**
```
Created: Status = SCHEDULED
During call: Recording in progress, Transcription enabled
After call: Stream webhook → Status = COMPLETED
Timeout: If not completed in 30 days → Status = CANCELLED
Feedback generation triggers after COMPLETED
```

---

### Table: CreditTransaction
**Purpose:** Audit trail for all credit movements

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `userId` | String | Foreign key to User |
| `amount` | Int | Credits moved (+/- integer) |
| `type` | Enum | `CREDIT_PURCHASE`, `BOOKING_DEDUCTION`, `BOOKING_EARNING`, `ADMIN_ADJUSTMENT` |
| `bookingId` | String | Optional reference to Booking (nullable) |
| `createdAt` | DateTime | Transaction timestamp |

**Relationships:**
- Many-to-One: `user` ← User
- Many-to-One: `booking` ← Booking (optional)

**Transaction Types:**
```
CREDIT_PURCHASE: User bought credits (e.g., +5)
BOOKING_DEDUCTION: Interviewee paid for booking (e.g., -10)
BOOKING_EARNING: Interviewer earned credits (e.g., +10)
ADMIN_ADJUSTMENT: Manual admin adjustment (e.g., +50)
```

**Sample Transactions for One Booking:**
```
1. Interviewee: -10 (BOOKING_DEDUCTION)
2. Interviewer: +10 (BOOKING_EARNING)
```

---

### Table: Feedback
**Purpose:** AI-generated feedback after each completed interview

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `bookingId` | String | Foreign key to Booking (UNIQUE, 1:1) |
| `summary` | String | Overall performance summary |
| `technical` | String | Technical skills assessment |
| `communication` | String | Communication style feedback |
| `problemSolving` | String | Problem-solving approach |
| `recommendation` | String | Recommendations for improvement |
| `strengths` | String[] | Array of identified strengths |
| `improvements` | String[] | Array of improvement areas |
| `overallRating` | Enum | `POOR`, `AVERAGE`, `GOOD`, `EXCELLENT` |
| `sessionRating` | Int | User-provided rating (1-5) |
| `sessionComment` | String | User comment |
| `createdAt` | DateTime | Generation timestamp |

**Relationships:**
- One-to-One: `booking` ← Booking (unique)

**How Feedback is Generated:**
```
1. Interview ends (call.ended webhook from Stream)
2. Booking status = COMPLETED
3. Trigger: Call Gemini with interview context
4. Gemini analyzes: questions asked, answers given, behavior
5. Generate: structured feedback
6. Store: Feedback record linked to booking
```

---

### Table: Payout
**Purpose:** Withdrawal requests from interviewers to convert credits to cash

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `interviewerId` | String | Foreign key to User |
| `credits` | Int | Number of credits to withdraw |
| `platformFee` | Float | Platform fee (%) |
| `netAmount` | Float | Amount after fees |
| `paymentMethod` | String | e.g., `bank_transfer`, `upi`, `paypal` |
| `paymentDetail` | String | e.g., Bank account or UPI ID |
| `status` | Enum | `PROCESSING`, `PROCESSED` |
| `adminNote` | String | Admin comments |
| `createdAt` | DateTime | Request timestamp |
| `updatedAt` | DateTime | Last status update |
| `processedAt` | DateTime | Completion timestamp |
| `processedBy` | String | Admin user ID |

**Relationships:**
- Many-to-One: `interviewer` ← User

**Indexes:**
- `(status, createdAt)` — Find recent pending payouts
- `(interviewerId, status)` — Find user's payouts

**Payout Workflow:**
```
1. Interviewer requests withdrawal
   Status: PROCESSING
   Email sent to admin with details

2. Admin approves
   Status: PROCESSED
   processedAt & processedBy recorded
   Email sent to interviewer

3. Credits move from User.creditBalance → sent to user
```

---

### Enums

```javascript
UserRole:
├─ UNASSIGNED (new user, not yet chosen role)
├─ INTERVIEWEE (wants to prepare for interviews)
└─ INTERVIEWER (conducts interviews, earns credits)

BookingStatus:
├─ SCHEDULED (awaiting interview)
├─ COMPLETED (interview finished)
└─ CANCELLED (either party cancelled)

AvailabilityStatus:
├─ AVAILABLE (open for bookings)
├─ BOOKED (slot taken)
└─ BLOCKED (interviewer blocked this time)

PayoutStatus:
├─ PROCESSING (awaiting admin approval)
└─ PROCESSED (payment sent)

InterviewCategory:
├─ FRONTEND
├─ BACKEND
├─ FULLSTACK
├─ DSA (Data Structures & Algorithms)
├─ SYSTEM_DESIGN
├─ BEHAVIORAL
├─ DEVOPS
└─ MOBILE

FeedbackRating:
├─ POOR
├─ AVERAGE
├─ GOOD
└─ EXCELLENT

TransactionType:
├─ CREDIT_PURCHASE
├─ BOOKING_DEDUCTION
├─ BOOKING_EARNING
└─ ADMIN_ADJUSTMENT
```

---

### Database Relationships Diagram

```
User (1) ──────┐
               ├──> (Many) Booking (interviewer)
               ├──> (Many) Booking (interviewee)
               ├──> (Many) Availability
               ├──> (Many) CreditTransaction
               └──> (Many) Payout

Booking (1) ──┬──> (1) Feedback
              └──> (Many) CreditTransaction

CreditTransaction: FK to both User & Booking (optional)
```

---

### Sample Query Operations

**Get user's upcoming interviews:**
```javascript
const bookings = await db.booking.findMany({
  where: {
    intervieweeId: userId,
    status: "SCHEDULED",
    startTime: { gt: new Date() }
  },
  include: {
    interviewer: { select: { name, imageUrl, title } }
  },
  orderBy: { startTime: "asc" }
})
```

**Get interviewer's earnings:**
```javascript
const stats = await db.booking.aggregate({
  where: { interviewerId: userId, status: "COMPLETED" },
  _sum: { creditsCharged: true },
  _count: true
})
// Returns: { _sum: { creditsCharged: 250 }, _count: 5 }
```

**Atomic booking transaction:**
```javascript
await db.$transaction(async (tx) => {
  // All 3 succeed or all fail
  await tx.booking.create({ ... })
  await tx.creditTransaction.create({ ... })
  await tx.user.update({ ... })
})
```

---

## Prisma Integration

### How Prisma Works

**1. Schema Definition** (`prisma/schema.prisma`)
```
Defines:
├─ Data source (PostgreSQL)
├─ Generator (PrismaPg adapter)
└─ Models (User, Booking, etc.)
```

**2. Migrations** (`prisma/migrations/`)
```
Each migration = timestamped SQL file
├─ migration_lock.toml (database engine marker)
├─ 20260309142032_init/migration.sql
│  └─ Initial tables (User, Booking, etc.)
└─ 20260315191913_add_plan_credit_tracking/migration.sql
   └─ Added creditRate, creditBalance fields
```

**3. Prisma Client Generation**
```
$ prisma generate
  └─ Reads schema.prisma
  └─ Generates TypeScript types & query methods
  └─ Output: lib/generated/prisma/
     ├─ client.ts (main client)
     ├─ models.ts (TypeScript interfaces)
     ├─ enums.ts (enum types)
     ├─ commonInputTypes.ts (query filters)
     └─ index.d.ts (type definitions)
```

**4. Runtime Execution**
```javascript
import { db } from "@/lib/prisma"

// Prisma converts this to SQL:
const user = await db.user.findUnique({
  where: { id: "123" }
})

// Generated SQL:
SELECT * FROM "User" WHERE id = $1

// Returns: User object (type-safe)
```

### Key Prisma Features Used

**1. Relationships & Includes**
```javascript
const booking = await db.booking.findFirst({
  include: {
    interviewer: true,  // Join with User table
    feedback: true,     // Join with Feedback table
    transactions: true  // Join with CreditTransaction table
  }
})
```

**2. Where Filters**
```javascript
// Complex queries
await db.booking.findMany({
  where: {
    status: "SCHEDULED",
    startTime: { gt: new Date() },  // Greater than
    interviewer: { role: "INTERVIEWER" }  // Nested filter
  }
})
```

**3. Transactions**
```javascript
await db.$transaction(async (tx) => {
  // All operations in this block are atomic
  // Database guarantees: all succeed or all fail
})
```

**4. Aggregations**
```javascript
const stats = await db.booking.aggregate({
  _sum: { creditsCharged: true },
  _count: true,
  _avg: { creditsCharged: true }
})
```

**5. Upserts**
```javascript
await db.availability.upsert({
  where: { id: availId },
  update: { startTime, endTime },
  create: { interviewerId, startTime, endTime }
})
```

### Prisma Client Singleton Pattern

**Problem:** Hot reload in Next.js dev creates multiple PrismaClient instances

**Solution:**
```javascript
// lib/prisma.js
const globalForPrisma = globalThis

function createPrismaClient() {
  return new PrismaClient({ adapter: new PrismaPg(pool) })
}

export const db = globalForPrisma.prisma ?? createPrismaClient()

if (NODE_ENV !== 'production') {
  globalForPrisma.prisma = db  // Reuse in dev
}
```

This ensures:
- Production: 1 client per server instance
- Development: 1 client across hot reloads

---

## Supabase Integration

### What is Supabase?

Supabase is a Firebase alternative providing:
- **PostgreSQL Database** (hosted)
- **Authentication** (optional, but we use Clerk instead)
- **Real-time Subscriptions**
- **Vector Store** (for AI features)
- **Storage** (for files)

### Your Supabase Setup

**Connection Details:**
```
Host: aws-1-ap-south-1.pooler.supabase.com
Database: postgres
Region: Asia (ap-south-1)
Pooler: Yes (pgbouncer for connection management)

Two Connection URLs:
1. DATABASE_URL (Transaction mode, port 6543)
   └─ Used by: Regular app queries (95% of traffic)
   └─ Advantage: Faster, connection pooling

2. DIRECT_URL (Session mode, port 5432)
   └─ Used by: Database migrations only
   └─ Advantage: Full session context for schema changes
```

### Why Use Supabase?

1. **Managed PostgreSQL**
   - No server maintenance
   - Automatic backups
   - High availability

2. **Connection Pooling**
   - Prevent connection exhaustion
   - Better performance for serverless

3. **Row Level Security** (optional)
   - Database-level access control
   - Not currently implemented, but available

4. **Vector Store**
   - Future: semantic search for interview topics
   - Gemini embeddings stored here

### Supabase vs Alternatives

| Feature | Supabase | Firebase | PlanetScale |
|---------|----------|----------|------------|
| SQL | ✅ Full SQL | ❌ No SQL | ✅ SQL |
| PostgreSQL | ✅ Yes | ❌ NoSQL | ❌ MySQL |
| Connection Pool | ✅ Yes | ❌ Limited | ✅ Yes |
| Cost | ✅ Pay per use | ❌ Expensive | ✅ Good |
| AI Features | ✅ Vector support | ❌ Limited | ❌ Limited |

---

## Architecture Design Decisions

### 1. Why Server Actions Instead of REST API?

**Decision:** Use Next.js Server Actions for 95% of operations

**Why?**
```
Server Actions Benefits:
├─ Type Safety: End-to-end TypeScript
├─ Smaller Bundle: Code stays on server
├─ Automatic Serialization: JSON handled transparently
├─ CORS-free: No cross-origin issues
├─ Token Included: Auth automatic
└─ Dev Experience: Call server functions like client functions

REST API Benefits:
└─ Only: Multiple clients (mobile app, third-party integration)
   └─ Your case: Not needed (web-only for now)
```

**Trade-off:** REST API would add complexity without current benefit

---

### 2. Why Transactional Bookings?

**Problem:** Race conditions in booking flow
```
Without transaction:
1. Create booking
2. Deduct credits ← App crash here
3. Add credit balance
Result: Credits lost or duplicated!

With transaction:
1. BEGIN
2. All 3 operations
3. COMMIT or ROLLBACK (atomic)
Result: Consistency guaranteed
```

**Decision:** Use `db.$transaction()` for booking operations

---

### 3. Why Prisma Over Raw SQL?

**Reasons:**
```
Prisma:
├─ Type Safety: DB schema → TypeScript types
├─ Query Builder: Prevents SQL injection
├─ Migrations: Version control for schema
├─ Performance: Compiled queries
└─ Developer Experience: IntelliSense in editor

Raw SQL:
├─ Better for: Complex queries (not needed here)
└─ Worse for: Security, maintainability, types
```

---

### 4. Why Clerk Over Custom Auth?

**Decision:** Use Clerk for authentication

**Why?**
```
Custom Auth requires:
├─ Password hashing (bcrypt, argon2)
├─ Session management
├─ OAuth implementation
├─ Token refresh logic
├─ CSRF protection
├─ Rate limiting on login
├─ Password reset flows
├─ 2FA support
└─ 2-3 weeks development

Clerk provides:
├─ All above out-of-box
├─ Social login (Google, GitHub)
├─ User metadata management
├─ Session tokens automatically
└─ Sign-up: 1-2 hours integration
```

---

### 5. Why Google Gemini for AI?

**Decision:** Use Gemini 2.5 Flash for question generation

**Why?**
```
Cost Analysis:
├─ GPT-4 ($0.03/1K tokens) expensive for 1000+ users
├─ Gemini Flash ($0.075/1M tokens) 400x cheaper
├─ Claude ($0.003/1K tokens) comparable but slower

Speed:
├─ Gemini Flash: 2-3s (acceptable for UX)
├─ GPT-4: 5-10s (too slow)
└─ Claude: 3-5s (good but pricier)

Reliability:
├─ Question generation: Doesn't need complex reasoning
├─ Flash model: Sufficient quality
└─ No need for premium models
```

**Trade-off:** Flash model is less powerful than full GPT-4, but sufficient for interview Q&A

---

### 6. Why Rate Limiting with Arcjet?

**Problem:** Abuse scenarios
```
Without limits:
1. User makes 1000 bookings in 1 second
2. Database overwhelmed
3. Credit system corrupted

With Arcjet:
├─ Max 5 bookings/hour per user
├─ Max 3 withdrawal requests/hour
└─ Fingerprinting: By userId (not IP)
   └─ Survives load balancers & proxies
```

**Why Arcjet?**
- Built for Next.js
- Fingerprinting by userId (better than IP)
- Easy integration
- Generous free tier (100k requests/month)

---

### 7. Why Stream.io for Video?

**Decision:** Use Stream.io SDK for video calls

**Why?**
```
Alternatives considered:
├─ WebRTC vanilla: 2-3 weeks dev, fragile
├─ Jitsi: Self-hosted complexity
├─ Twilio: $0.04/minute (expensive)
├─ Zoom API: Limited customization
└─ Stream.io: $0.007/minute + features

Stream.io features:
├─ 1080p recording built-in
├─ Automatic transcription
├─ Screen sharing
├─ Webhooks (call.ended, etc.)
└─ React SDK (easy UI)
```

---

### 8. Why Separate Credit Systems?

**Interviewee Credits:**
```
├─ Input: Monthly subscription (Free: 1, Starter: 5, Pro: 15)
├─ Output: Book interviews at 10 credits each
├─ Mechanism: Automatic allocation on signup + monthly reset
└─ Purpose: Monetize the platform
```

**Interviewer Credit Balance:**
```
├─ Input: +10 credits per completed booking
├─ Output: Withdraw as payout
├─ Mechanism: Earnings accumulate, pay out on demand
└─ Purpose: Compensate interviewers
```

**Why Separate?**
- Interviewees: Spend credits, subscription model
- Interviewers: Earn credits, gig economy model

---

## Authentication System

### Clerk Overview

Clerk is a user management platform that handles:
- Authentication (email, password, social login)
- Session management
- User metadata
- Webhooks for user events

### Login Flow

```
1. FRONTEND → User clicks "Sign In"
   └─ Redirects to /sign-in (Clerk-hosted UI)

2. CLERK SERVERS
   └─ Renders Clerk pre-built sign-in form
   └─ User enters email + password
   └─ Clerk verifies credentials

3. SESSION TOKEN
   └─ Clerk generates JWT token
   └─ Token stored in httpOnly cookie (secure)
   └─ Middleware intercepts cookie

4. MIDDLEWARE
   └─ File: middleware.ts (if it exists, currently missing)
   └─ Purpose: Protect routes
   └─ Check: Is token valid?
   └─ Action: Allow / Redirect to login

5. FRONTEND → Logged in
   └─ currentUser() available in server actions
   └─ User can access protected pages
```

### How Clerk Middleware Works

**Typical Implementation:**
```javascript
// middleware.ts
import { clerkMiddleware, createRouteMatcher } from "@clerk/nextjs/server"

const isProtectedRoute = createRouteMatcher([
  "/dashboard",
  "/appointments",
  "/call/(.*)",
  "/payout/(.*)"
])

export default clerkMiddleware((auth, req) => {
  if (isProtectedRoute(req)) {
    auth().protect()  // Ensure user is authenticated
  }
})

export const config = {
  matcher: ["/((?!.*\\..*|_next).*)", "/"]
}
```

**How It Works:**
```
Every Request:
├─ Check: Is route protected?
├─ If yes: Is token valid?
│  ├─ Valid: Allow request
│  └─ Invalid: Redirect to /sign-in
└─ If no: Allow request (public route)
```

### User Session Maintenance

**Session Lifecycle:**
```
1. User logs in
   └─ Clerk creates JWT token
   └─ Token stored in httpOnly cookie

2. Every request
   └─ Cookie automatically sent
   └─ Middleware validates token
   └─ If valid: Continue
   └─ If expired: Attempt refresh

3. Token refresh
   └─ Clerk maintains refresh token separately
   └─ Automatically exchanges for new access token
   └─ User stays logged in (seamless)

4. User logs out
   └─ DELETE /api/auth/logout
   └─ Cookie cleared
   └─ Session ends

5. Browser close
   └─ Cookie persists (httpOnly)
   └─ Next visit: Session restored
```

### Authorization System

**Two-layer authorization:**

**Layer 1: Role Check**
```javascript
const dbUser = await db.user.findUnique({
  where: { clerkUserId: user.id }
})

if (dbUser.role !== "INTERVIEWER") {
  throw new Error("Forbidden: Only interviewers can set availability")
}
```

**Layer 2: Resource Ownership**
```javascript
// Can't edit another user's availability
if (availability.interviewerId !== dbUser.id) {
  throw new Error("Forbidden: Not your availability")
}
```

### Security Considerations

**1. Server-side Verification**
```
✅ Always verify user role/permissions on server
✌️ Never trust frontend role checking
❌ Bad: Check role only in React component
✅ Good: Check role in server action
```

**2. Token Validation**
```javascript
// Built-in: Clerk middleware validates token
// ✅ Invalid tokens rejected automatically
// ✅ Expired tokens refreshed automatically
```

**3. CSRF Protection**
```
Built-in: Next.js automatically handles CSRF for server actions
✅ Safe: No manual CSRF token management needed
```

**4. Rate Limiting**
```javascript
// Booking rate limit: 5/hour
// Purpose: Prevent automated attacks
// By: userId (not IP, for proxy safety)
```

---

## AI Integration

### Gemini API Integration

**API Choice:** Google Gemini 2.5 Flash
```
├─ Model: gemini-2.5-flash-lite (lightweight)
├─ Latency: ~2-3 seconds
├─ Cost: $0.075 per 1M input tokens
└─ Use Case: Interview question generation
```

### Question Generation Flow

**File:** `actions/aiQuestions.jsx`

```javascript
export const generateInterviewQuestions = async ({ category }) => {
  // 1. AUTHENTICATE
  const user = await currentUser()  // Get Clerk user
  if (!user) throw new Error("Unauthorized")
  
  // 2. VALIDATE CATEGORY
  if (!CATEGORY_PROMPTS[category])
    throw new Error("Invalid category")
  
  // 3. SETUP GEMINI CLIENT
  const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY)
  const model = genAI.getGenerativeModel({ 
    model: "gemini-2.5-flash-lite" 
  })
  
  // 4. BUILD PROMPT
  const prompt = `You are an expert technical interviewer.
Generate 6 interview questions for a ${category} role...
Respond ONLY with a valid JSON array...`
  
  // 5. CALL GEMINI
  const result = await model.generateContent(prompt)
  const text = result.response.text().trim()
  
  // 6. PARSE RESPONSE
  const clean = text.replace(/^```json|^```|```$/gm, "").trim()
  const questions = JSON.parse(clean)
  
  // 7. RETURN
  return { questions }
}
```

### Prompt Templates

**Structure:**
```javascript
const CATEGORY_PROMPTS = {
  FRONTEND: "React, JavaScript, CSS, performance, accessibility...",
  BACKEND: "Node.js, REST APIs, databases, authentication...",
  FULLSTACK: "full-stack architecture, API design, state management...",
  DSA: "data structures, algorithms, time complexity...",
  SYSTEM_DESIGN: "distributed systems, scalability, databases...",
  BEHAVIORAL: "leadership, teamwork, conflict resolution...",
  DEVOPS: "CI/CD, Docker, Kubernetes, cloud...",
  MOBILE: "React Native, iOS/Android, performance..."
}
```

### Response Parsing

**Why JSON is important:**
```javascript
// Gemini sometimes adds markdown backticks:
// ```json
// [{"question": "...", "answer": "..."}]
// ```

// Clean it:
const clean = text
  .replace(/^```json|^```|```$/gm, "")  // Remove backticks
  .trim()

// Parse:
const questions = JSON.parse(clean)
// Result: Array of question/answer objects
```

### Feedback Generation (Future)

**Conceptual Flow:**
```
1. Interview ends
2. Stream webhook: /api/webhooks/stream
3. Extract call data: recording, transcription
4. Call Gemini with context:
   ├─ Questions asked
   ├─ Time spent on each
   ├─ User's answers (from transcription)
   └─ Code written (if any)
5. Gemini generates structured feedback:
   ├─ Technical skills
   ├─ Communication
   ├─ Problem-solving
   └─ Rating
6. Store: Feedback record
7. Send: Email to interviewee
```

**Example Prompt:**
```
Interview analysis. User answered these questions:
1. "What is React?" - User said: "..."
2. "Explain useEffect" - User said: "..."

Provide feedback on:
- Technical understanding
- Communication clarity
- Code quality (if applicable)
- Overall assessment

Return JSON: {
  "summary": "...",
  "technical": "...",
  "communication": "...",
  "rating": "GOOD"
}
```

---

## Interview Questions

### 🏗️ BACKEND ARCHITECTURE QUESTIONS

1. **Request Flow**
   - Walk me through a booking request from frontend to database. What happens at each layer?
   - How does rate limiting protect your system? What could happen without it?
   - Why use server actions instead of REST APIs?

2. **Database Design**
   - Why separate credit systems for interviewees and interviewers?
   - What's the purpose of the CreditTransaction table? Why not just update User.credits directly?
   - How would you handle a booking failure mid-transaction? What guarantees does Prisma provide?

3. **Architecture Decisions**
   - Your app uses PostgreSQL + Prisma + Supabase. Why this combination?
   - What problems does connection pooling solve?
   - How would you scale this system if you had 1M concurrent users?

---

### 💾 DATABASE QUESTIONS

1. **Schema Design**
   - Draw the relationships between User, Booking, Feedback, and CreditTransaction tables.
   - Why is `streamCallId` marked as UNIQUE?
   - What does `@@index([status, createdAt])` do? When would you use it vs. a full table scan?

2. **Complex Queries**
   - Write a query to find all completed bookings for a user with their feedback.
   - How would you calculate an interviewer's total earnings and completed session count in one query?
   - What's the N+1 problem and how does Prisma solve it?

3. **Transactions**
   - Why is the booking operation wrapped in `db.$transaction()`?
   - What happens if the user's credits are insufficient during transaction? How is rollback handled?
   - How would you implement credit refunds after a cancelled booking?

---

### 🔐 AUTHENTICATION QUESTIONS

1. **Clerk Integration**
   - How does Clerk maintain user sessions across browser refreshes?
   - What's stored in the JWT token? Is it secure to store in cookies?
   - How would a hacker bypass Clerk authentication?

2. **Authorization**
   - You check `role !== "INTERVIEWER"` in `setAvailability()`. Why not check this in the frontend?
   - How does your app prevent an interviewee from accessing another user's appointments?
   - Design: How would you implement admin-only routes?

3. **Security**
   - What is CSRF and why doesn't your server actions need manual CSRF tokens?
   - If someone steals a user's session cookie, what damage could they do?
   - How would you implement 2FA (two-factor authentication)?

---

### 🤖 AI & GEMINI QUESTIONS

1. **Question Generation**
   - Why use Gemini Flash instead of GPT-4 for interview question generation?
   - What could go wrong if Gemini's response isn't valid JSON? How do you handle it?
   - How would you cache generated questions to avoid redundant API calls?

2. **Prompt Engineering**
   - Write a prompt to generate DSA interview questions that also provide solutions.
   - How would you make Gemini generate follow-up questions based on user answers?
   - What's the cost of generating 1000 questions for a large user base?

3. **Feedback System (Future)**
   - How would you use Gemini to analyze interview transcriptions and generate feedback?
   - What data would you send to Gemini? (Transcription, code, answers)
   - How do you prevent Gemini from hallucinating (making up feedback)?

---

### 🏪 SYSTEM DESIGN QUESTIONS

1. **Scaling**
   - Your current app has ~10 active interviews at any time. Design for 1000 concurrent interviews.
   - How would you handle database connection limits if you hit Supabase's max connections?
   - What bottlenecks would appear first? (DB, API rate limits, video infrastructure)

2. **Real-time Features**
   - Design: How would you show live slot availability updates without polling?
   - How would you notify an interviewee that their interviewer is running 5 minutes late?
   - Implement: Real-time interview notifications using WebSockets or Server-Sent Events

3. **Fault Tolerance**
   - What happens if Stream.io is down? Can bookings still be created?
   - How would you handle Gemini API failures during question generation?
   - Design: A retry strategy for failed credit transactions.

---

### ⚡ ADVANCED QUESTIONS

1. **Credit System**
   - Design a fair pricing model: How much should interviewees pay vs. how much interviewers earn?
   - How would you prevent credit fraud (e.g., interviewee asks for fake refunds)?
   - What does "platform fee" in Payout table represent? How do you calculate it?

2. **Concurrency**
   - Two users simultaneously book the last available slot. How does your system handle this?
   - What race conditions exist in your booking code? How are they prevented?
   - How does Prisma's transaction isolation level prevent dirty reads?

3. **Performance Optimization**
   - The `getInterviewerAppointments()` includes full interviewer and feedback data. Is this efficient?
   - How would you optimize queries that currently might cause N+1 problems?
   - Why use separate DATABASE_URL and DIRECT_URL for migrations?

---

### 📊 INTERVIEW SCENARIO QUESTIONS

1. **Booking Problem**
   ```
   Scenario: An interviewee books a slot, sees success, but:
   - Credits are deducted
   - Interviewer doesn't see the booking
   - Booking record in DB is incomplete
   
   What went wrong? How would you debug?
   ```

2. **Credit Discrepancy**
   ```
   Scenario: A user reports they lost 50 credits without a booking.
   
   How would you investigate using CreditTransaction?
   What audit trail should you implement?
   ```

3. **Scale Crisis**
   ```
   Scenario: You gain 100K users overnight. The database is slow.
   
   What queries would you optimize first?
   Would you need caching (Redis)?
   Should you shard the database?
   ```

---

## Difficult Interview Questions

### 🔴 EXTREMELY DIFFICULT

1. **Consistency Under Failure**
   > "In your booking transaction, if the Stream.io call creation fails after the Prisma transaction commits, you have a booking in the database but no actual call. How would you design around this?"
   
   **Expected Answer:**
   - Stream.io call must be created BEFORE transaction
   - Or: Idempotent operations (retry-safe)
   - Or: Background job to clean up orphaned bookings
   - Or: Two-phase commit pattern

2. **Distributed Transactions**
   > "You want to expand to multiple regions (US, EU, Asia) with database replicas. How would you maintain credit consistency across regions?"
   
   **Expected Answer:**
   - Event sourcing (log all credit changes)
   - Eventually consistent model
   - Region-specific credit ledgers with periodic reconciliation
   - Or: Keep credits centralized (single-region)

3. **Credit System Fairness**
   > "An interviewer with poor ratings complains they make less money than an interviewer with high ratings, even though they complete the same number of interviews. How do you design a fair system?"
   
   **Expected Answer:**
   - Dynamic pricing (adjust creditRate based on ratings)
   - Bonus pool for top performers
   - Base rate + tips system
   - User-selectable rate per interview

4. **AI Feedback Hallucination**
   > "Gemini generates feedback saying the interviewee 'solved the problem' when they actually didn't. The interviewee disputes it. How do you verify Gemini's feedback?"
   
   **Expected Answer:**
   - Store raw transcription data
   - Human review system for disputes
   - A/B test Gemini against human feedback
   - Multiple AI models voting
   - Manual verification for rating disputes

5. **Privacy & Data**
   > "GDPR: A user in Europe requests their data deleted. Your Booking table has a foreign key to User. What's deleted? What stays?"
   
   **Expected Answer:**
   - Delete User record (cascade deletes Availability, Payouts, Feedback)
   - Keep Booking records but anonymize (for financial auditing)
   - Or: Hard delete everything (but lose transaction history)
   - Design: Soft deletes with `isDeleted` flag

---

### 🟠 VERY DIFFICULT

6. **N+1 Query Problem Identification**
   > "In `getInterviewerAppointments()`, you include full interviewer, interviewee, and feedback data. If there are 100 bookings, how many database queries are made? Write an optimized version."
   
   **Current Code:**
   ```javascript
   include: {
     interviewer: true,
     interviewee: true,
     feedback: true
   }
   ```
   **Problem:** If `interviewer` and `interviewee` aren't explicitly fetched, Prisma may lazily load them (100 extra queries)
   
   **Solution:** Use explicit `select` or ensure relationships are loaded

7. **Arcjet Bypass Scenarios**
   > "Your rate limiter uses `userId` as fingerprint. What if a hacker creates multiple accounts? How do you prevent abuse at scale?"
   
   **Expected Answer:**
   - Track by phone number (harder to fake)
   - IP address + userId combo
   - Payment method (only one payment per card)
   - Machine learning fraud detection
   - Behavioral analysis (booking patterns)

8. **Webhook Reliability**
   > "Stream.io sends a call.ended webhook. If your server is down, what happens? How do you ensure feedback is generated?"
   
   **Expected Answer:**
   - Exponential backoff retries (Stream.io does this)
   - Dead letter queue (log failed webhooks)
   - Periodic reconciliation job (check for bookings without feedback)
   - Idempotent handler (safe to process twice)

9. **Credit System Inflation**
   > "You notice credit inflation: Total credits issued > Total credits spent. What debugging steps do you take?"
   
   **Expected Answer:**
   - Query CreditTransaction table (sum all transactions)
   - Check for unauthorized ADMIN_ADJUSTMENT
   - Look for duplicate BOOKING_EARNING entries
   - Audit stripe/payment system integration
   - Compare to Booking.creditsCharged sum

10. **Timezone Issues**
    > "An interviewer in Tokyo schedules availability 9:00-17:00. An interviewee in New York books a slot. The call happens at the wrong time. What went wrong?"
    
    **Expected Answer:**
    - DateTime fields must always be UTC in DB
    - Frontend converts local time to UTC before sending
    - Frontend converts UTC to local time for display
    - Bug: If local time stored directly, causes mismatch
    - Test: Verify timezone handling

---

## Summary: Architecture Strengths & Trade-offs

| Aspect | Strength | Trade-off |
|--------|----------|-----------|
| **Server Actions** | Type-safe, secure | No multi-client support |
| **Transactional Bookings** | Data consistency | Slightly higher latency |
| **Prisma ORM** | Developer experience | Less control than raw SQL |
| **Clerk Auth** | Quick setup | Vendor lock-in |
| **Gemini Flash** | Cost-effective | Less powerful than GPT-4 |
| **Arcjet Rate Limiting** | Easy integration | Billing depends on usage |
| **Stream.io Video** | Full-featured | Expensive at scale |
| **Supabase PostgreSQL** | Managed DB | Connection limits at scale |

---

## Next Steps for Production

1. **Add Middleware**: Implement route protection with Clerk
2. **Add Error Boundaries**: Better error handling in components
3. **Optimize Queries**: Use caching for frequently accessed data
4. **Add Logging**: Structured logging for debugging
5. **Set Up Monitoring**: Track API response times, error rates
6. **Add Tests**: Unit tests for business logic, integration tests for flows
7. **Security Audit**: OWASP top 10 review
8. **Load Testing**: Simulate 100+ concurrent users
9. **Backup Strategy**: Daily database snapshots
10. **Compliance**: GDPR, privacy policy, terms of service
