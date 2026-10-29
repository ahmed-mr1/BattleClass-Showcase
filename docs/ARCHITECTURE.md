# System Architecture

This document describes the architectural design, component structure, data flow, and technology stack of BattleClass.

---

## 1. High-Level Architecture

BattleClass is organized as a decoupled web application comprising a Next.js client, a FastAPI asynchronous backend, and a PostgreSQL relational database. Live state synchronization between the organizer console, stage display, and audience smartphones is handled via Server-Sent Events (SSE).

```mermaid
graph TD
    subgraph Clients ["Client Applications"]
        Student["Student Mobile Client<br/>(/e/{code})"]
        Stage["Stage Projector View<br/>(/e/{code}/live)"]
        Admin["Organizer Console<br/>(/e/{code}/control & /admin)"]
    end

    subgraph Edge ["Edge & Reverse Proxy"]
        Proxy["Reverse Proxy (Caddy / Nginx)<br/>TLS Termination & Path Routing"]
    end

    subgraph WebTier ["Web & Application Tier"]
        NextServer["Frontend Server<br/>Next.js 15 (Node.js runtime)"]
        FastAPIServer["API Engine<br/>FastAPI / Starlette (Uvicorn)"]
    end

    subgraph ServiceLayer ["Backend Service Layer"]
        AuthService["Auth & Session Manager<br/>(Argon2id + HTTP-only cookies)"]
        StateEngine["Battle State Machine<br/>(Deterministic transitions)"]
        TimerEngine["Timer & Overtime Engine<br/>(Server authoritative clock)"]
        Broadcaster["In-Memory Broadcaster<br/>(Async event fan-out)"]
        AnalyticsEngine["Reporting Engine<br/>(OpenPyXL multi-sheet export)"]
    end

    subgraph Persistence ["Data Tier"]
        DB[(PostgreSQL 16+<br/>SQLAlchemy 2.0 + Alembic)]
    end

    Student -->|HTTPS / WSS| Proxy
    Stage -->|HTTPS / SSE| Proxy
    Admin -->|HTTPS / Session Cookie| Proxy

    Proxy -->|Path: / & static| NextServer
    Proxy -->|Path: /api/*| FastAPIServer

    FastAPIServer --> AuthService
    FastAPIServer --> StateEngine
    FastAPIServer --> TimerEngine
    FastAPIServer --> Broadcaster
    FastAPIServer --> AnalyticsEngine

    AuthService --> DB
    StateEngine --> DB
    AnalyticsEngine --> DB
    TimerEngine -.->|Syncs ends_at| DB
    StateEngine -->|State versions| Broadcaster
    Broadcaster -.->|Server-Sent Events| Stage
    Broadcaster -.->|Server-Sent Events| Student
    Broadcaster -.->|Server-Sent Events| Admin
```

---

## 2. Core Components

### 2.1 Frontend (Next.js 15)
- **Framework:** Next.js 15 using App Router and React 19.
- **Styling:** Tailwind CSS with high-contrast stage layouts and mobile-first student views.
- **Client Synchronization:** The `useBattleState` hook establishes an `EventSource` connection to the SSE feed, manages clock drift calibration against `server_time`, and triggers reactive view re-renders on version bumps.
- **View Modes:**
  - **Student View (`/e/[code]`):** Mobile-optimized responsive interface for entering join PINs, answering real-time understanding checks, monitoring earned vote power, and submitting weighted ballots.
  - **Stage Projector View (`/e/[code]/live`):** Fullscreen 1080p display for classroom projectors showing join QR codes, dual-challenger cards, circular countdown timer rings, live check accuracy distributions, and final winner celebration reveals.
  - **Organizer Console (`/e/[code]/control`):** Desktop command console for configuring battle parameters, importing and vetting question banks, controlling the speaking clock, dispatching questions, and opening/closing voting.
  - **Admin Dashboard (`/admin`):** Multi-battle management interface for creating tournaments, bulk approvals, status filtering, and downloading Excel reports.

### 2.2 Backend (FastAPI & Starlette)
- **Framework:** FastAPI with Python 3.12+, utilizing Starlette for ASGI routing and Pydantic v2 for payload validation.
- **Concurrency Model:** Fully asynchronous request handling powered by `asyncio` and `uvicorn`.
- **Authentication:** Admin authentication uses Argon2id password hashing stored in the database. Successful logins issue cryptographically signed, HTTP-only session cookies with `SameSite=Lax`.
- **Public Allowlist Security:** All API endpoints are protected by default under a global dependency check (`get_current_admin`), with explicit allowlist routing for public student interactions (`/api/events/{code}/*`).
- **Rate Limiting:** IP-based tracking limits brute-force attempts on admin login endpoints.

### 2.3 Realtime Event Streaming
- **Protocol:** HTTP Server-Sent Events (`text/event-stream`).
- **Broadcasting Mechanism:** An in-memory `Broadcaster` registry maintains active subscriber queues per battle code. When a battle transition occurs, the updated canonical state object is serialized and dispatched concurrently to all connected stage, student, and admin clients.
- **Clock Drift Mitigation:** Broadcasted payloads carry `server_time` (ISO-8601 UTC) and `server_now` (millisecond timestamp). Clients compute the offset (`server_now - Date.now()`) to ensure countdown rings and answer windows expire in strict lockstep across all devices regardless of client clock skew.

### 2.4 Data Tier (PostgreSQL)
- **Database Engine:** PostgreSQL 16+.
- **Database Access:** SQLAlchemy 2.0 with asynchronous drivers (`psycopg`).
- **Migrations:** Alembic version-controlled migration scripts.
- **Integrity Constraints:** Foreign key relations with cascading deletes, uniqueness constraints on battle codes and student participation tokens, and transactional state version checks.

---

## 3. End-to-End Data Flow

### 3.1 Presentation & Understanding Check Cycle
```mermaid
sequenceDiagram
    autonumber
    actor Admin as Organizer Console
    actor Stage as Stage Projector
    actor Student as Student Mobile
    participant API as FastAPI Backend
    participant DB as PostgreSQL

    Admin->>API: POST /api/battles/{code}/timer (action: "start")
    API->>DB: Update timer.state="running", timer.ends_at
    API-->>Admin: 200 OK
    API-)Stage: SSE event: state (timer running)
    API-)Student: SSE event: state (timer running)

    Note over Stage,Student: Clock counts down in lockstep with server offset

    Admin->>API: POST /api/battles/{code}/checkpoint (action: "open", question_id)
    API->>DB: Pause speaking timer, create Checkpoint record, set window_ms
    API-)Stage: SSE event: state (checkpoint active, prompt visible)
    API-)Student: SSE event: state (answer options enabled, timer started)

    Student->>API: POST /api/events/{code}/answers (participant_token, selected_option)
    API->>DB: Insert Submission record (validates window open & single submission)
    API-->>Student: 200 OK (accepted)

    Note over API: Answer window expires or Admin forces close

    Admin->>API: POST /api/battles/{code}/checkpoint (action: "close")
    API->>DB: Compute distribution, accuracy %, update attendee scores
    API-)Stage: SSE event: checkpoint_closed (distribution bar chart, correct answer)
    API-)Student: SSE event: checkpoint_closed (personal result + weight gain)
    API-)Admin: SSE event: state (checkpoint completed)

    Admin->>API: POST /api/battles/{code}/timer (action: "resume")
    API->>DB: Resume speaking timer with remaining duration
    API-)Stage: SSE event: state (timer running)
```

### 3.2 Voting & Results Calculation
```mermaid
sequenceDiagram
    autonumber
    actor Admin as Organizer Console
    actor Stage as Stage Projector
    actor Student as Student Mobile
    participant API as FastAPI Backend
    participant DB as PostgreSQL

    Admin->>API: POST /api/battles/{code}/phase (phase: "VOTING")
    API->>DB: Update battle.state="VOTING"
    API-)Stage: SSE event: state (voting turnout view)
    API-)Student: SSE event: state (ballot view unlocked)

    Student->>API: POST /api/events/{code}/ballot (token, criteria votes)
    API->>DB: Retrieve student's earned vote weight
    API->>DB: Insert Ballot record (per-criterion choices)
    API-->>Student: 200 OK
    API-)Stage: SSE event: state (increment turnout count)

    Admin->>API: POST /api/battles/{code}/phase (phase: "CLOSED")
    API->>DB: Update battle.state="CLOSED"
    API-)Stage: SSE event: state (ballots sealed view)

    Admin->>API: POST /api/battles/{code}/phase (phase: "REVEALED")
    API->>DB: Sum raw votes & weighted points per challenger
    API->>DB: Determine tournament victor
    API-)Stage: SSE event: state (victory celebration, score breakdown)
    API-)Admin: SSE event: state (final results & export ready)
```

---

## 4. Key Architectural Decisions

1. **Server-Sent Events over WebSockets:** The application requires low-latency server-to-client broadcasts (timer ticks, checkpoint launches, phase shifts), while student interactions are standard discrete HTTP POST requests (submitting an answer, casting a ballot). SSE provides native HTTP/2 multiplexing, automatic reconnection handling, and firewall compatibility without the protocol overhead of full-duplex WebSockets.
2. **Server-Authoritative Clock:** Clocks on student phones and classroom laptops often drift by several seconds. Timers never rely on client-side intervals for remaining time; the server broadcasts authoritative timestamps (`ends_at`, `remaining_ms`, `server_now`), and clients calculate remaining duration relative to the server clock.
3. **Optimistic Locking via State Versions:** All mutations to battle state check an integer `state_version`. If a concurrent request attempts to modify the battle from an outdated state, the backend rejects the transaction, preventing race conditions during fast-paced classroom presentations.
4. **Deny-by-Default Security:** The API routing layer enforces administrative authentication across all routes by default. Only endpoints explicitly designated for student participation are attached to an open public router.
