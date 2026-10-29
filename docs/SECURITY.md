# Security Model & Operational Limitations

This document describes the security architecture, authentication mechanisms, authorization controls, and known operational tradeoffs in BattleClass.

---

## 1. Authentication & Session Security

### 1.1 Password Hashing
- Administrative passwords are never stored in plaintext. They are hashed using **Argon2id** (`argon2-cffi`), configuring memory cost, time cost, and parallelism according to modern cryptographic recommendations.

### 1.2 Session Management
- **Token Generation:** When an administrator logs in, the backend generates a cryptographically secure 32-byte pseudo-random string using Python's `secrets` module.
- **Token Storage:** The raw token is returned to the client in an HTTP response header setting a cookie. Only the **SHA-256 hash** of the token is persisted in the `admin_sessions` table (`token_hash`), ensuring that a database dump does not expose active session tokens.
- **Cookie Security:**
  - `HttpOnly`: Cookie cannot be read or stolen via client-side JavaScript (mitigating XSS extraction).
  - `SameSite=Lax`: Cookie is omitted on cross-site subrequests, mitigating Cross-Site Request Forgery (CSRF).
  - `Secure`: Forced in production environments so cookies are transmitted strictly over HTTPS.
  - `Path=/`: Restricts scope to application domain.
- **Session Lifetime & Expiration:** Sessions carry an explicit expiration timestamp (`expires_at`). Revocation occurs immediately upon logout by deleting the session row.

---

## 2. Authorization & Routing Architecture

### 2.1 Deny-by-Default Design
All administrative routes and battle mutation endpoints require authentication by default. A global FastAPI dependency (`get_current_admin`) intercepts incoming requests:
1. Extracts the session cookie.
2. Hashes the incoming token with SHA-256.
3. Looks up the hash in `admin_sessions` joining `admin_users`.
4. Rejects invalid, expired, or inactive accounts with `401 Unauthorized`.

### 2.2 Public Endpoint Allowlist
To support anonymous student participation without requiring student logins, an explicit allowlist governs unauthenticated access. Only the following routes bypass admin authentication:
- `POST /api/auth/login` (Admin authentication entrypoint)
- `GET /api/events/{code}` & `/api/events/{code}/state` (Read public battle state)
- `GET /api/events/{code}/stream` (Public Server-Sent Events feed)
- `POST /api/events/{code}/participants` (Register student display name)
- `POST /api/events/{code}/participants/restore` (Re-establish student session from token)
- `POST /api/events/{code}/participants/heartbeat` (Keepalive tracking)
- `POST /api/events/{code}/answers` (Submit answer to active question checkpoint)
- `POST /api/events/{code}/ballot` (Submit final voting ballot)
- `GET /api/events/{code}/results` (Read final battle results post-reveal)

All other endpoints (creating battles, modifying timers, importing questions, dispatching checkpoints, exporting Excel analytics) strictly reject requests lacking a valid admin session.

---

## 3. Rate Limiting & Abuse Prevention

- **Login Brute-Force Protection:** The `/api/auth/login` endpoint tracks authentication attempts per IP address using an in-memory sliding window limiter. Repeated invalid attempts trigger HTTP `429 Too Many Requests` responses to deter automated credential stuffing.
- **Idempotent Control Room Actions:** Organizer console commands submit an `idempotency_key`. The `audit_log` table enforces uniqueness per battle code to prevent accidental double-submits (e.g., double-clicking "Start Timer" or "Open Checkpoint").

---

## 4. Known Limitations & Tradeoffs

### 4.1 Anonymous Student Participation vs Sybil Voting
- **Design Tradeoff:** BattleClass is built for friction-free classroom seminars. Requiring 40–100 students to create accounts or authenticate via OAuth during a live lecture creates onboarding delays and breaks debate momentum. Instead, students join by entering a code and are issued a participant token saved in browser `localStorage`.
- **Known Limitation:** A student can technically open an Incognito window, a secondary browser, or clear site data to obtain a fresh participant token, allowing multiple ballot submissions.
- **Classroom Mitigations:**
  1. *Live Turnout Monitor:* The stage display broadcasts the total votes cast alongside classroom headcount in real time (e.g., `37 OF 40 VOTES CAST`). Duplicate voting immediately causes turnout to exceed physical attendance, alerting the instructor.
  2. *Single-Submission Constraint:* Within each browser session, the database strictly enforces `UniqueConstraint("event_id", "participant_id")` on ballots and `UniqueConstraint("checkpoint_id", "participant_id")` on answers.

### 4.2 Network Latency on Question Windows
- Checkpoint answering windows run on a server-authoritative timer (typically 15–20 seconds).
- Students on congested Wi-Fi networks who submit in the final fraction of a second may have their HTTP request arrive after the server window has closed. The server strictly rejects late submissions.

### 4.3 In-Memory Broadcaster Scaling
- The realtime SSE engine utilizes an in-memory `Broadcaster` instance.
- This design provides near-zero latency fan-out for a single-server deployment (adequate for dozens of concurrent classrooms).
- Horizontally scaling across multiple backend instances behind a load balancer would require integrating an external publish/subscribe message broker (such as Redis Pub/Sub).
