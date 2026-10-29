# Battle State Machine & Question Lifecycle

This document provides the formal state transition specifications for BattleClass battles, timers, and question checkpoints.

---

## 1. Battle State Machine

The battle lifecycle transitions through sequential states managed authoritatively by the organizer console. State transitions are verified transactionally and broadcast to all connected clients.

```mermaid
stateDiagram-v2
    [*] --> PENDING: Battle Created

    state PENDING {
        [*] --> SetupDraft
        SetupDraft --> QuestionImport: JSON Upload
        QuestionImport --> Vetting: Review & Approve
        Vetting --> ReadinessCheck: Verify All Criteria
        ReadinessCheck --> [*]: All Checks Green
    }

    PENDING --> READY: Admin Finalizes Setup & Runs Randomizer
    READY --> PENDING: Admin Reopens Setup

    READY --> PRESENTING_1: Start First Speaker
    
    state PRESENTING_1 {
        [*] --> TimerRunning_1
        TimerRunning_1 --> CheckpointActive_1: Open Question Window
        CheckpointActive_1 --> TimerRunning_1: Close Window & Resume Timer
        TimerRunning_1 --> Overtime_1: Time Expires (Budget Exceeded)
        Overtime_1 --> [*]: End Speaker 1
        TimerRunning_1 --> [*]: End Speaker 1
    }

    PRESENTING_1 --> WAITING_FOR_2: Finish Speaker 1 (Changeover)

    WAITING_FOR_2 --> PRESENTING_2: Start Second Speaker

    state PRESENTING_2 {
        [*] --> TimerRunning_2
        TimerRunning_2 --> CheckpointActive_2: Open Question Window
        CheckpointActive_2 --> TimerRunning_2: Close Window & Resume Timer
        TimerRunning_2 --> Overtime_2: Time Expires (Budget Exceeded)
        Overtime_2 --> [*]: End Speaker 2
        TimerRunning_2 --> [*]: End Speaker 2
    }

    PRESENTING_2 --> ENDED: Finish Speaker 2

    ENDED --> VOTING: Open Voting Window
    VOTING --> CLOSED: Seal Ballots
    CLOSED --> REVEALED: Publish Results to Stage
    REVEALED --> [*]: Battle Complete / Export Ready
```

---

## 2. State Definitions & Permitted Triggers

| State | Role Responsible | Allowed Next States | Description & Behavior |
| :--- | :--- | :--- | :--- |
| **`PENDING`** | Admin / Organizer | `READY` | Battle draft. Admin configures presenter names, topics, speaking time, and imports the question bank. Questions are vetted and approved. |
| **`READY`** | Admin / Organizer | `PENDING`, `PRESENTING_1` | Setup locked. Order of play draw executed. Stage displays lobby with QR code and battle PIN. Audience joins and enters lobby. |
| **`PRESENTING_1`** | Admin / Organizer | `WAITING_FOR_2` | First speaker on stage. Speaking countdown active. Admin dispatches understanding checks (clock automatically pauses during questions). Overtime tracks if speech exceeds allocation. |
| **`WAITING_FOR_2`** | Admin / Organizer | `PRESENTING_2` | Changeover / intermission. Speaking clock reset for Challenger 2. Presenter transition on stage. |
| **`PRESENTING_2`** | Admin / Organizer | `ENDED` | Second speaker on stage. Speaking countdown active. Second set of understanding checks dispatched and answered. Overtime tracks if speech exceeds allocation. |
| **`ENDED`** | Admin / Organizer | `VOTING` | Presentations concluded. Both speakers have completed their talks and comprehension checks. |
| **`VOTING`** | Admin / Organizer | `CLOSED` | Final voting window open. Students submit ballots across criteria (Clarity, Technical Depth, Argument Strength). Stage shows live turnout percentage. |
| **`CLOSED`** | Admin / Organizer | `REVEALED` | Ballots sealed. System computes raw vote counts and weighted points based on earned student weights. |
| **`REVEALED`** | Admin / Organizer | *Terminal* | Victor announced on stage. Score breakdown, criteria comparisons, and participant results displayed. Battle is archived and available for Excel reporting. |

---

## 3. Question Checkpoint Lifecycle

During a presentation, the organizer triggers comprehension checks at strategic intervals. Each question follows a strict lifecycle to guarantee integrity:

```mermaid
sequenceDiagram
    autonumber
    participant Bank as Question Bank
    participant State as Battle State
    participant Timer as Speaking Timer
    participant Client as Audience Clients

    Note over Bank,Client: Question in "Approved" state in Question Bank

    Bank->>State: Admin triggers "Open Question"
    State->>Timer: Pause speaking clock (remaining_ms frozen)
    State->>State: Transition to CheckpointActive (window_ms set, e.g., 20s)
    State-)Client: Broadcast question prompt and randomized option choices

    Note over Client: Students submit choices before timer reaches 0:00

    Client->>State: POST answer submission (verified within window)
    State->>State: Record submission (1 attempt per participant)

    Note over State: Window reaches 0:00 OR Admin triggers "Close Question"

    State->>State: Aggregate option distribution and correct rate %
    State->>State: Award comprehension points (+1 check) to correct submissions
    State->>Bank: Mark question as "USED" (cannot be re-asked)
    State-)Client: Broadcast correct answer, explanation, and audience distribution
    
    State->>Timer: Resume speaking clock
    State->>State: Return to Presenting state
```

### Key Checkpoint Rules:
1. **Clock Freezing:** The speaking timer automatically pauses while a question checkpoint is active, ensuring presentation time is not consumed by audience comprehension checks.
2. **Single-Attempt Enforcement:** A participant can submit only one answer per question checkpoint. Additional submissions from the same session token are rejected.
3. **One-Time Use:** Questions are consumed upon closing and cannot be relaunched within the same battle.
4. **Late Submissions:** Answers arriving after the window closes are rejected with an expired window error.
