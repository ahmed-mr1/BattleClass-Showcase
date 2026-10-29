# BattleClass Feature and Function Inventory

A comprehensive technical and functional inventory of the BattleClass presentation arena platform, derived strictly from codebase analysis.

---

## Table of Contents

1. [Features Grouped by Role](#1-features-grouped-by-role)
   - [Visitor / Student (Audience)](#visitor--student-audience)
   - [Admin / Organizer](#admin--organizer)
   - [Stage / Projector Screen](#stage--projector-screen)
2. [End-to-End Workflows](#2-end-to-end-workflows)
   - [Admin Workflow](#admin-workflow-end-to-end)
   - [Student Workflow](#student-workflow-end-to-end)
3. [Battle State Machine, Question Lifecycle & Timer Engine](#3-battle-state-machine-question-lifecycle--timer-engine)
   - [State Machine & Transitions](#state-machine--transitions)
   - [Question Lifecycle](#question-lifecycle)
   - [Timer Engine Rules](#timer-engine-rules)
4. [Vote Weighting Formula & Edge Cases](#4-vote-weighting-formula--edge-cases)
5. [Security Model & Known Limitations](#5-security-model--known-limitations)
   - [Authentication & Session Architecture](#authentication--session-architecture)
   - [CSRF & Deny-by-Default Authorization](#csrf--deny-by-default-authorization)
   - [Rate Limiting](#rate-limiting)
   - [Known Limitations](#known-limitations)
6. [Realtime SSE Event Architecture](#6-realtime-sse-event-architecture)
7. [Export and Analytics Engine](#7-export-and-analytics-engine)
   - [Endpoints & Generators](#endpoints--generators)
   - [Metric Definitions: Code vs Intent](#metric-definitions-code-vs-intent)
   - [Exported Reports & Column Schemas](#exported-reports--column-schemas)
8. [Technical Stack, Testing & Deployment](#8-technical-stack-testing--deployment)
9. [By the Numbers](#9-by-the-numbers)

---

## 1. Features Grouped by Role

### Visitor / Student (Audience)
- **Frictionless Join:** Join a live battle by scanning a stage QR code or typing a 6-character alphanumeric join code without creating an account.
- **Participant Profile Persistence:** Choose an audience nickname/display name; session identity is maintained via a hashed browser-stored participant token with automatic reconnection and heartbeat tracking.
- **Tournament Schedule Viewer:** Browse the full roster of scheduled and completed battles, active order, and battle status directly from the landing page or mobile modal.
- **Interactive Rules Modal:** View the exact vote weighting rules, the matched-pairs concept, and scoring transparency directly on mobile.
- **Synchronized Speaker Clock:** View a real-time countdown timer synchronized with the speaker clock during presentations, with responsive visual cues (amber under 60s, red under 10s).
- **Live Understanding Checks:** Receive prompt interactive multiple-choice questions (2 to 4 options) pushed to mobile devices when a check opens.
- **Immediate Response Feedback:** Submit an answer within a strict countdown window; receive instant visual confirmation of selection and wait for the synchronized answer reveal.
- **Answer & Accuracy Reveal:** Inspect the correct answer, option-by-option distribution percentages, and the author's explanation once the answer window closes.
- **Vote Power Tracker:** Track earned vote power in real time with visual status icons showing correct/missed answers for each presenter and current matched pairs.
- **Split-Screen Voting Ballot:** Cast a vote for Presenter A or Presenter B upon conclusion of both talks, with real-time feedback showing the student's earned multiplier.
- **Live Waiting & Turnout Indicator:** Observe real-time audience voting turnout percentages and ballot confirmation while awaiting the official reveal.
- **Final Results Display:** View official battle results including winner, raw vote counts, weighted point totals, and vote percentages once revealed by the organizer.

### Admin / Organizer
- **Admin Authentication:** Secure login using email and password verified with Argon2id password hashing, protected by brute-force rate limiting and HttpOnly session cookies.
- **Battle Creation:** Create new head-to-head battles with topic title, presenter names, talk titles, and duration settings.
- **Batch Battle & Question Import:** Upload Excel (`.xlsx`), CSV, or scanned sheets to populate multiple battles, presenter assignments, and questions in bulk with auto-sanitization.
- **Tournament Schedule Reordering:** Drag, drop, or save custom battle presentation order sequences applied dynamically across audience and stage views.
- **Question Pool Management:** Author questions manually, import JSON pools (2–4 options, single correct index, explanation), edit question prompts/options, or discard questions.
- **AI Question Drafting (Optional):** Extract text from uploaded presentation files (`.pdf`, `.pptx`, `.txt`, `.md`) and automatically generate grounded candidate quiz questions using an LLM backend (or deterministic mock mode).
- **First Speaker Randomizer:** An animated, physics-simulated roulette wheel that fairly determines whether Presenter A or Presenter B presents first and locks the chosen order.
- **Mission Control Room:** Unified dashboard to execute one-way phase transitions, monitor connected audience count, pause/resume timers, and trigger understanding checks.
- **Timer Extension & Control:** Adjust speaking clocks on the fly (+30s / +60s / -30s), trigger manual pauses, or allow presentations to roll into recorded overtime.
- **Question Scheduling & Sweeper:** Automatically schedule question breaks evenly across speaking duration or fire them manually on demand.
- **Question Invalidation (Void):** Invalidate ambiguous or flawed questions during or after a battle, automatically recalculating student vote weights in real time.
- **Battle Skip & Bulk Operations:** Skip unready or postponed battles with automatic state cleanup, or perform bulk deletion of test events.
- **Multi-Sheet Analytics Export:** Export comprehensive battle-level or tournament-wide Excel workbooks containing detailed timing, audience accuracy, vote logs, and full audit trails.

### Stage / Projector Screen
- **Lobby Arena Display:** Fullscreen fight-night style display presenting the battle title, challenger cards, join code, and a high-resolution QR code for quick scanning.
- **Synchronized Stage Countdown Ring:** SVG circular timer displaying remaining speaker time, active speaker indicator, overtime tracking, and color-coded urgency states.
- **Synchronized Checkpoint View:** Fullscreen question display showing question prompt, multiple-choice options, live respondent counter, and ticking answer countdown.
- **Audience Distribution Reveal:** Animated stage reveal displaying correct answer highlight, audience selection breakdown, and explanatory notes.
- **Live Voting Arena:** Stage countdown during the voting phase displaying real-time turnout progress without prematurely leaking vote tallies.
- **Celebratory Results Reveal:** Confetti-animated winner announcement displaying official weighted scores, raw vote counts, and audience engagement statistics.

---

## 2. End-to-End Workflows

### Admin Workflow (End-to-End)
1. **Login:** Admin navigates to `/login`, enters credentials, and authenticates against the server. On success, an `admin_session` cookie is issued.
2. **Create or Import Battle:**
   - *Single Battle:* Navigate to `/admin/new`, supply title, presenter names, talk topics, presentation duration (default 5 min), answer window (default 30s), and questions per presenter (default 3).
   - *Batch Import:* In `/admin`, upload a multi-battle spreadsheet or JSON roster. Battles are staged in a pending state and activated upon admin approval.
3. **Setup & Question Preparation:**
   - In `/e/[code]/control`, review presenter details and upload slide decks (`.pdf`, `.pptx`, `.txt`, `.md`) to extract text.
   - Click "Generate questions" (if AI configured or in mock mode) or import a structured JSON question pool.
   - Review candidate questions, edit options or explanations, and approve at least the target number of questions per presenter.
4. **First Speaker Determination (Randomizer):**
   - Verify the readiness checklist (presenter names, titles, approved questions).
   - Open the Randomizer modal and spin the wheel to randomly assign whether Side A or Side B presents first. The battle transitions from `PENDING` to `READY`.
5. **Open Lobby:**
   - Click "Open Lobby". The battle transitions to `lobby`. The stage screen displays the battle title, presenter sides, and join QR code.
6. **Presenter 1 Session:**
   - Click "Start Presenter 1". The clock ticks down from allocated duration.
   - Questions fire either automatically via scheduled timestamps or manually when the admin taps "Ask Question".
   - During a question break, the speaker's clock automatically pauses.
   - The question window counts down (e.g., 30s). When closed, the correct answer and audience selection statistics reveal for a configurable duration (e.g., 8s) before normal speaker time resumes.
   - When finished, click "Finish Presenter 1". The battle transitions to `changeover`.
7. **Presenter 2 Session:**
   - Click "Start Presenter 2". Presenter 2 presents under the exact same timer and question mechanics.
   - When finished, click "Finish Presentation". The battle transitions to `ended` or directly to `voting`.
8. **Voting Phase:**
   - Click "Start Voting". The voting ballot unlocks on all student devices.
   - Admin monitors live turnout percentage (ballots submitted vs total connected participants).
   - When turnout is satisfactory, click "Close Voting". Ballots are locked, and final weights are recalculated.
9. **Reveal Results:**
   - Click "Reveal Results". Results become public across stage, student, and admin screens, accompanied by a confetti celebration.
10. **Export & Archive:**
    - From the admin dashboard, download the detailed Excel report for the battle or the aggregate tournament summary.

### Student Workflow (End-to-End)
1. **Join Battle:** Student scans the stage QR code or visits the homepage, enters the 6-character battle code, and provides a display nickname.
2. **Lobby Waiting Screen:** The phone displays a live battle lobby showing battle metadata, the two presenters, a tournament schedule button, and an interactive rules explanation modal.
3. **Live Presentation Tracking:**
   - As presentations run, the student's screen displays a synchronized presentation clock matching the stage display.
   - A persistent Vote Power Tracker card at the top shows Question icons for Presenter A and Presenter B.
4. **Answering Understanding Checks:**
   - When an understanding check opens, the phone screen immediately displays the question prompt and multiple-choice options with a countdown ring.
   - The student taps an option. The selection is highlighted and locked on the server.
5. **Reviewing Explanations:**
   - Once the timer expires, the answer screen reveals whether the student's choice was correct (green) or incorrect (red), displays the overall audience distribution bar, and presents the explanation text.
6. **Vote Power Accumulation:**
   - Correctly answering a question marks that presenter's question slot as correct.
   - When the student scores at least one correct answer for Presenter A AND at least one for Presenter B, the tracker registers a "Matched Pair" (+0.1x multiplier boost).
7. **Voting for the Winner:**
   - When voting opens, a split-screen ballot displays Presenter A and Presenter B cards with their total correct checkpoints.
   - A prominent badge displays the student's personal voting power (e.g., "1.2x Vote Power — 2 Matched Pairs").
   - The student taps their preferred presenter and submits the ballot.
8. **Waiting & Final Results:**
   - The phone screen confirms ballot receipt and shows live audience turnout.
   - Once the organizer triggers the reveal, the final outcome (winner, score breakdown, and percentage) renders in real time.

---

## 3. Battle State Machine, Question Lifecycle & Timer Engine

### State Machine & Transitions

The battle state machine enforces a strict, irreversible, one-way progression:

```
PENDING (Draft) -> READY -> LOBBY -> PRESENTING_A -> CHANGEOVER -> PRESENTING_B -> ENDED -> VOTING -> CLOSED -> REVEALED
```

*(Note: If the randomizer selects Side B as the first presenter, presentation phases logically map Presenter B to the first slot and Presenter A to the second slot).*

#### Allowed Transitions Matrix
| Current Phase | Allowed Action | Target Phase | Triggered By |
|---|---|---|---|
| `PENDING` | `prepare` or `randomize` | `READY` | Admin (organizer) |
| `READY` | `open_lobby` | `lobby` | Admin |
| `lobby` | `start_a` (or `start_presenter`) | `presenting_a` | Admin |
| `presenting_a` | `finish_a` (or `finish_presentation`) | `changeover` | Admin |
| `changeover` | `start_b` (or `start_presenter`) | `presenting_b` | Admin |
| `presenting_b` | `finish_b` (or `finish_presentation`) | `ended` or `voting` | Admin |
| `ended` | `start_voting` | `voting` | Admin |
| `voting` | `close_voting` (or `finish_voting`) | `closed` | Admin |
| `closed` | `reveal` | `revealed` | Admin |

- **One-Way Protection:** Backward transitions are strictly rejected with HTTP 409 Conflict.
- **Optimistic Concurrency:** State modifications validate `state_version`. If concurrent requests attempt conflicting transitions, the stale update is rejected.
- **Idempotency Support:** State transitions support an optional `Idempotency-Key` header, allowing safe re-execution of network-interrupted admin actions.

### Question Lifecycle

A question break is a sub-state of `presenting_a` or `presenting_b`. It is **not** a separate top-level battle phase.

1. **Unasked Pool:** Approved questions reside in the question pool associated with Presenter A or Presenter B.
2. **Triggering (Opening):**
   - Triggered either manually by admin or automatically by the background sweeper when presentation speaking time hits a scheduled offset.
   - The speaker's countdown timer is paused (`timer_paused_by_checkpoint = true`).
   - A `Checkpoint` record is created in `status = "open"` with an expiration timestamp `answer_closes_at = now + answer_window_s` (default 30s).
   - Event broadcast `checkpoint_opened` is pushed to all SSE clients.
3. **Answering Window:**
   - Students submit single answers via HTTP `POST /api/events/{code}/answers`.
   - Answers after `answer_closes_at` are rejected with HTTP 409 Conflict. Duplicate answers per student per checkpoint are rejected.
4. **Reveal Window:**
   - When `answer_closes_at` is reached, the sweeper advances checkpoint `status = "revealed"`.
   - The correct answer index and audience answer distribution are revealed to clients.
   - The reveal remains visible for `reveal_s` seconds (default 8s).
5. **Completion & Resume:**
   - When the reveal period expires, the sweeper advances checkpoint `status = "done"`.
   - `timer_paused_by_checkpoint` is set to `false`, and the speaker's presentation timer resumes exactly where it left off.
6. **Void / Invalidation:**
   - An organizer can void a flawed question (`status = "invalidated"`).
   - Voided questions are completely excluded from student accuracy counts, matched-pair totals, and vote weight calculations.

### Timer Engine Rules

- **Server-Authoritative:** Timers are never ticking integer counters on the server. They are calculated dynamically from wall-clock timestamps:
  - `remaining_ms = max(0, duration_ms - (now - running_since))`
- **Clock Pauses During Questions:** When a question break opens, `running_since` is cleared, and accumulated speaking time freezes. Time elapsed during questions does not count against the presenter's speaking budget.
- **Extra Time Adjustments:** Admin can adjust presentation time via `extend(ms)` (+30s, +60s, -30s). Adjustments are tracked cumulatively in `Presenter.extra_time_ms`.
- **Live Overtime Tracking:** If a presentation exceeds its allotted duration (`remaining_ms == 0`) while still in `running` state, overtime accrues live:
  - `overtime_ms = max(0, elapsed - duration)`
  - Overtime is finalized upon presentation finish and recorded in the audit log and analytics reports.

---

## 4. Vote Weighting Formula & Edge Cases

The voting weight formula enforces attention across **both** presentations to prevent partisan voting:

$$\text{pairs} = \min(\text{correct\_P1}, \text{correct\_P2})$$
$$\text{weight\_x10} = 10 + \text{pairs}$$
$$\text{Display Multiplier} = \frac{\text{weight\_x10}}{10} = 1.0 + 0.1 \times \text{pairs}$$

### Implementation Details
- **Integer Tenths Arithmetic:** Stored in the database as integer tenths (`weight_x10`) to eliminate floating-point rounding errors (e.g., `10` = 1.0x, `11` = 1.1x, `13` = 1.3x).
- **Matched Pair Requirement:** A student must correctly answer at least one question for Presenter 1 AND at least one question for Presenter 2 to earn a boost.
- **Symmetric & Neutral:** The weight multiplier is calculated strictly from engagement and never depends on which presenter the student votes for.
- **Theoretical Maximum:**
  $$\text{max\_weight\_x10} = 10 + \min(\text{asked\_P1}, \text{asked\_P2})$$

### Edge Cases
1. **One-Sided Engagement:** A student who answers all questions correctly for Presenter A (e.g., 3/3) but zero questions for Presenter B earns `pairs = min(3, 0) = 0`. Their vote power remains 1.0x (unboosted, never penalized).
2. **Late Joiners:** A student joining mid-battle who misses Presenter A entirely still participates with a baseline 1.0x vote.
3. **Voided Questions:** If a question is voided by the organizer, it is flagged as `invalidated`. It is removed from both `correct_count` and `asked_count`, and all ballots are re-weighted immediately.
4. **Tie Resolution:** Winner is determined by higher weighted points. If weighted points are equal, the official result is recorded as a **draw** (there is no arbitrary raw vote tie-break).
5. **Decisive Voter Flag (`focused_voters_made_difference`):** Evaluated as `true` if and only if:
   - Weighted result is not a draw,
   - Raw result is not a draw,
   - The weighted winner differs from the raw vote winner.

---

## 5. Security Model & Known Limitations

### Authentication & Session Architecture
- **Admin Accounts:** Credentials stored in `admin_users` table (`email`, `password_hash`, `is_active`, `created_at`, `last_login_at`).
- **Argon2id Password Hashing:** Passwords hashed using Argon2id (RFC 9106 profile). Minimum password length enforced at 12 characters.
- **Constant-Time Timing Attack Mitigation:** Failed lookups for nonexistent admin emails execute a dummy Argon2id hash verification (`verify_dummy_password`) to ensure response times are identical.
- **HttpOnly Cookies:** Session tokens are 32-byte cryptographically secure hex strings. The server stores only the SHA-256 digest (`token_hash`) in `admin_sessions`. The cookie is marked `HttpOnly`, `SameSite=Lax`, and `Secure` (in production).
- **No Public Admin Registration:** Admin accounts can only be provisioned or reset via server CLI commands (`python -m app.cli create-admin`), preventing automated signup exploits.

### CSRF & Deny-by-Default Authorization
- **Public Allowlist (`PUBLIC_ALLOWLIST`):** The application implements a strict deny-by-default access policy. Only routes explicitly declared in `PUBLIC_ALLOWLIST` (health, student join/answer/vote, public state stream) permit unauthenticated access.
- **Mandatory Admin Dependency:** All administrative and mutative battle management endpoints require valid admin session resolution via `require_admin`.
- **CSRF Verification (`check_csrf`):** State-changing HTTP methods (`POST`, `PUT`, `PATCH`, `DELETE`) validate `Origin` and `Referer` headers against configured allowed origins. Mismatches are rejected with HTTP 403 Forbidden.

### Rate Limiting
- **Login Rate Limiter:** An in-memory sliding window limiter tracks failed login attempts. Exceeding 5 failed attempts per email + IP within a 15-minute window results in an immediate HTTP 429 Too Many Requests lockout for 15 minutes.

### Known Limitations
- **Pseudonymous Student Identity:** Students join anonymously by providing a display nickname. Devices are tracked via a browser-stored participant token. An individual user could open multiple private/incognito tabs or use multiple physical devices to vote more than once.
- **Single-Worker Process Constraint:** The real-time in-memory pub/sub hub, active speaker countdown timers, login rate limiter, and background sweeper loop reside in the application process. The backend must run with a single worker (`--workers 1`).

---

## 6. Realtime SSE Event Architecture

Real-time communication is powered by Server-Sent Events (SSE) using HTTP streaming with unbuffered reverse proxy delivery (`flush_interval -1`).

### Channels
1. **Public Stream (`GET /api/events/{code}/stream`):** Consumed by audience mobile devices and the stage projector screen. Requires only a valid battle code.
2. **Admin Stream (`GET /api/events/{code}/admin-stream`):** Consumed by the admin control room. Requires an active admin session cookie.

### Event Types & Payload Schemas
- **`state` (Battle State Snapshot):** Broadcast on every state transition, timer status change, or presenter update.
  - *Payload includes:* battle code, phase, status, version, current side, first side, timer remaining/duration, active checkpoint (without correct answer), public vote results (only if revealed).
- **`checkpoint_opened`:** Broadcast when an understanding check opens.
  - *Payload includes:* checkpoint ID, presenter side, question prompt, options array (option ID and text), window duration in milliseconds, and closing timestamp.
  - *Safety:* Correct answer index and explanation are strictly stripped from the payload.
- **`checkpoint_revealed`:** Broadcast when the question window closes.
  - *Payload includes:* checkpoint ID, correct index, correct option ID, author explanation, and audience option selection distribution counts.
- **`checkpoint_closed`:** Broadcast when the reveal phase finishes and the question break concludes.
- **`: ping` (Heartbeat):** Comment keep-alive line transmitted at regular intervals to prevent intermediate proxy timeout.

### Reconnection Resilience
- The client-side hook (`useBattleState`) reconnects automatically upon connection drop using an exponential backoff strategy.
- Upon reconnect, the server immediately pushes a complete `state` snapshot, resynchronizing the client to the current server timestamp and phase.

---

## 7. Export and Analytics Engine

### Endpoints & Generators
- `GET /api/admin/battles/{code}/export`: Generates a battle-specific multi-sheet Excel workbook (`battle_{code}_report.xlsx`) via `build_battle_xlsx()`.
- `GET /api/admin/battles/export-all`: Generates a tournament-wide multi-sheet Excel workbook (`battleclass_all_battles_{timestamp}.xlsx`) via `build_all_battles_xlsx()`.
- `GET /api/events/{code}/results`: Returns battle outcome statistics in JSON format.

### Metric Definitions: Code vs Intent

| Metric | Intended Meaning | What the Code Actually Computes | Code Location |
|---|---|---|---|
| **Students Joined** | Number of unique students in attendance. | Count of rows in `participants` table associated with the battle code (`len(participants)`). Every device that executed a join request is counted. | `report_export.py:185`, `report_export.py:545` |
| **Speaking Time** | Net time the presenter spoke. | If start/finish `TimerEvent` records exist: `elapsed = abs(finish[-1] - start[0])`. Otherwise, calculated as `(duration + extra_time + overtime) - remaining`. | `report_export.py:60-74` |
| **Presenters Within Time** | A pass/fail summary of time adherence. | The code does **not** compute a single aggregated metric or boolean flag for "stayed within time". Instead, the export records `Allocated Time`, `Actual Speaking Time`, `Time Adjustments`, and `Overtime` (ms) per presenter for manual review. | `report_export.py:202-227`, `report_export.py:548-555` |
| **Students Answering Q[X]** | Number of participants answering a specific question. | Count of `Answer` records matching the specific `Checkpoint.id` (`len(cp_ans)`). | `report_export.py:339`, `report_export.py:712` |
| **Per-Question Correct Rate** | Percentage of students who got a question right. | `round((correct_responses / total_responses) * 100, 1)` if `total_responses > 0` else `0.0%`. | `report_export.py:341`, `report_export.py:714` |
| **Overall Audience Accuracy** | Overall battle accuracy across all questions. | Count of correct answers divided by total answers submitted across all battle checkpoints: `round((correct_answers / total_answers) * 100, 1)`. | `report_export.py:587` |
| **Struggled Topics** | Identify topics students found most difficult. | The code contains **no automated topic struggle ranking function or struggle threshold rule**. Instead, the export lists `Audience Question Accuracy (%)` and per-question accuracy alongside `Topic Title`, allowing the educator to sort and identify low-accuracy topics manually. | `report_export.py:589-612` |
| **Audience Turnout** | Voter participation rate. | `round(total_ballots / participant_count, 3)` expressed as a percentage. | `scoring.py:86`, `report_export.py:244` |
| **Vote Weight Multiplier** | Engagement bonus factor. | `(ballot.weight_x10 / 10)` based on `matched_pairs(correct_A, correct_B)`. Range: `1.0x` to `1.0 + 0.1 * min(asked_A, asked_B)`. | `report_export.py:295`, `report_export.py:666` |

### Exported Reports & Column Schemas

#### 1. Single Battle Report (`build_battle_xlsx`)
- **Sheet 1: "Battle Summary"**
  - *Details:* Battle Code, Topic Title, Status, Current Phase, Question Mode, Created At (UTC), Audience Size, Total Ballots Cast.
  - *Presenters Table:* Side, Presenter Name, Topic, Allocated Time, Time Adjustments, Actual Speaking Time, Overtime.
  - *Results Table:* Official Winner, Raw Vote Winner, Audience Turnout, Side A Weighted Points, Side B Weighted Points, Side A Raw Votes, Side B Raw Votes.
- **Sheet 2: "Attendee Performance"**
  - *Columns:* `Student Name`, `Joined At (UTC)`, `Total Answered`, `Side A Correct`, `Side B Correct`, `Total Correct`, `Accuracy (%)`, `Earned Vote Multiplier`, `Voted For`, `Vote Timestamp`.
- **Sheet 3: "Questions & Accuracy"**
  - *Columns:* `Question #`, `Presenter Side`, `Presenter Name`, `Question Prompt`, `Correct Answer`, `Total Responses`, `Correct Responses`, `Accuracy (%)`, `Avg Response Time (s)`.
- **Sheet 4: "Full Answer Log" (Audit Trail)**
  - *Columns:* `Answered At (UTC)`, `Student Name`, `Side`, `Presenter`, `Question Prompt`, `Selected Option`, `Is Correct?`, `Response Time (s)`.

#### 2. Tournament Summary Report (`build_all_battles_xlsx`)
- **Sheet 1: "Tournament Summary"**
  - *Columns:* `Code`, `Topic Title`, `Status`, `Phase`, `Winner`, `Challenger 1 (Side A)`, `Challenger 2 (Side B)`, `Allocated Time A`, `Allocated Time B`, `Actual Spoken Time A`, `Actual Spoken Time B`, `Time Adj A`, `Time Adj B`, `Questions Asked`, `Audience Size`, `Total Votes Cast`, `Votes Side A`, `Votes Side B`, `Vote % Side A`, `Vote % Side B`, `Weighted Points A`, `Weighted Points B`, `Audience Question Accuracy (%)`.
- **Sheet 2: "All Attendees & Scores"**
  - *Columns:* `Battle Code`, `Topic Title`, `Student Name`, `Joined At (UTC)`, `Questions Answered`, `Side A Correct`, `Side B Correct`, `Total Correct`, `Student Accuracy (%)`, `Vote Multiplier`, `Voted For`, `Vote Timestamp`.
- **Sheet 3: "All Questions & Accuracy"**
  - *Columns:* `Battle Code`, `Topic Title`, `Question #`, `Presenter Side`, `Presenter Name`, `Question Prompt`, `Correct Answer`, `Total Responses`, `Correct Responses`, `Accuracy (%)`, `Avg Time (s)`.
- **Sheet 4: "Full Answer Log"**
  - *Columns:* `Answered At (UTC)`, `Battle Code`, `Topic Title`, `Student Name`, `Side`, `Presenter`, `Question Prompt`, `Selected Option`, `Result`, `Response Time (s)`.

---

## 8. Technical Stack, Testing & Deployment

### Verified Technology Stack & Versions
- **Backend Framework:** FastAPI `0.157.0` running under Uvicorn `0.54.0` (Python 3.12+).
- **ORM & Database Tooling:** SQLAlchemy `2.1.5` (asyncio) with AsyncPG/Psycopg `3.3.6` driver and Alembic `1.20.0` for schema migrations.
- **Data Validation & Settings:** Pydantic `2.14.1` and Pydantic-Settings `2.16.1`.
- **Cryptography & Security:** Argon2-cffi `25.1.0` (Argon2id password hashing).
- **Document & Spreadsheet Processing:** OpenPyXL `3.1.5` (Excel exports), PyPDF `6.21.1`, python-pptx `1.0.2`.
- **Frontend Architecture:** Next.js `16.4.2` (App Router) on React `19.3.0` and React-DOM `19.3.0`.
- **Styling & Visuals:** Tailwind CSS `v4`, Canvas-Confetti `1.9.4`, QRCode.react `4.2.0`.
- **Testing Suite:** Pytest `9.1.1` with pytest-asyncio `1.4.0` and HTTPX `0.28.1`.

### Test Suite Structure & Execution
- **Coverage Areas:**
  - `test_admin.py`: Session authorization and battle administration.
  - `test_ai_and_files.py`: Slide deck text extraction and AI drafting fallbacks.
  - `test_api_flow.py`: Full happy-path battle lifecycle and edge cases.
  - `test_auth_unit.py`: Password hashing, timing protection, CSRF, and rate limiting.
  - `test_auto_checkpoints.py`: Automatic schedule generation and sweeper firing.
  - `test_battle_import.py` & `test_battle_import_api.py`: Excel/CSV parsing and approval workflow.
  - `test_battle_state.py` & `test_state_machine_api.py`: Phase transitions and concurrency rejection.
  - `test_hardening.py`: Reconnections, server restart recovery, and idempotency.
  - `test_question_engine.py`: Secret shielding and option distributions.
  - `test_report_export.py`: Multi-sheet Excel export validation.
  - `test_route_security.py`: Allowlist verification and deny-by-default enforcement.
  - `test_scoring.py`, `test_timer.py`, `test_vote_weight.py`, `test_voting.py`: Pure scoring formulas, clock drift, and matched pairs.
- **Running Tests:**
  ```bash
  cd backend
  pytest
  ```

### Deployment Architecture
- **Containerization:** Multi-stage Dockerfiles for backend (Python slim running as non-root `appuser`) and frontend (Next.js standalone output running as non-root `nextjs`).
- **Reverse Proxy:** Automated Caddy reverse proxy providing automatic HTTPS, routing `/api/*` to the backend with unbuffered streaming (`flush_interval -1`) and `/*` to the Next.js frontend.
- **Database:** PostgreSQL 17 with persistent volume storage and automated pre-flight Alembic migrations (`alembic upgrade head`).
- **CI/CD Automation:** Container images built via GitHub Actions and deployed over SSH with zero-downtime healthcheck verification against `/api/health`.

---

## 9. By the Numbers

- **FastAPI Endpoints:** 38 distinct logical endpoints (15 public, 22 admin-protected, 1 healthcheck) mounted under both canonical `/api/*` and root paths.
- **Database Tables:** 13 tables (`events`, `presenters`, `questions`, `criteria`, `checkpoints`, `participants`, `answers`, `ballots`, `timer_events`, `audit_log`, `admin_users`, `admin_sessions`, `system_settings`).
- **Automated Backend Tests:** 245 passing tests across 20 test suites.
- **Frontend Pages / Screens:** 7 distinct routes:
  1. `/` (Public Landing & Tournament Roster)
  2. `/login` (Admin Authentication)
  3. `/admin` (Tournament Management & Batch Import)
  4. `/admin/new` (Single Battle Creator)
  5. `/e/[code]` (Visitor / Student Arena View)
  6. `/e/[code]/control` (Organizer Mission Control Room)
  7. `/e/[code]/live` (Stage / Projector Display Screen)
