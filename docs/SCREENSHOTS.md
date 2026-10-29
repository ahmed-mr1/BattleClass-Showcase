# BattleClass Showcase Screenshot Plan

This document establishes the official shot list, viewport standards, required interface states, and capture methodology for generating visual assets for the BattleClass public showcase repository.

---

## 1. Technical Standards & Viewports

All screenshots must be captured from a **local instance running on localhost with synthetic demo data**. Capturing from a live or production environment is strictly prohibited.

| Viewport Type | Resolution | Target Audience / Use Case |
|---|---|---|
| **Desktop** | `1440 × 900` | Administrative workflows, battle setup, mission control room, analytics exports |
| **Stage / Projector** | `1920 × 1080` | High-definition auditorium display, stage countdown, live quiz reveal, victory confetti |
| **Phone (Mobile)** | `390 × 844` (iPhone 14 / modern smartphone) | Student audience participant interface, live questions, vote power tracker, ballot |

---

## 2. Numbered Shot List

### 01. Public Homepage & Tournament Roster
- **`01a-home-desktop.png`**
  - **Page / Route:** `/` (Logged out)
  - **Viewport:** Desktop `1440 × 900`
  - **State / Visible Elements:** Full hero presentation ("Present. Participate. Compete."), 6-character battle join code input form with "Join Battle" button, embedded Tournament Schedule list showing upcoming, active, and completed demo battles with fake topic titles, theme toggle, and footer.
  - **Demo Data State:** Seeded with 3 demo battles (e.g., "Gradient Descent vs Newton-Raphson", "Supervised vs Self-Supervised Learning", "CNNs vs Vision Transformers").
- **`01b-home-mobile.png`**
  - **Page / Route:** `/` (Logged out)
  - **Viewport:** Phone `390 × 844`
  - **State / Visible Elements:** Mobile-optimized join form prominently centered, floating or tabbed tournament schedule trigger, clean touch typography.
  - **Demo Data State:** Same seeded demo battles.

### 02. Admin Authentication
- **`02-login.png`**
  - **Page / Route:** `/login`
  - **Viewport:** Desktop `1440 × 900`
  - **State / Visible Elements:** Centered authentication card, email address field (`demo@example.com`), masked password input, "SHOW/HIDE" password reveal toggle button, "Sign In" action button, security notice.
  - **Demo Data State:** Ready for local admin authentication.

### 03. Admin Battle Management Dashboard
- **`03-admin-battle-list.png`**
  - **Page / Route:** `/admin` (Authenticated)
  - **Viewport:** Desktop `1440 × 900`
  - **State / Visible Elements:** Header with admin email badge and logout action; "New Battle" and "Upload sheet" action buttons; battle overview table displaying Battle Code, Topic Title, Presenter A and B names, Question Counts per side, Participant Count, Phase/Status badges (`READY`, `IN PROGRESS`, `DONE`), and action shortcuts (Control Room, Export Report, Skip, Delete).
  - **Demo Data State:** 3 demo battles in varied states (one `DONE` with participant count ~40, one `READY`, one `PENDING`).

### 04. Single Battle Creation Form
- **`04-new-battle-form.png`**
  - **Page / Route:** `/admin/new`
  - **Viewport:** Desktop `1440 × 900`
  - **State / Visible Elements:** Form with fields: Battle Title / Topic, Presenter A Name & Presentation Title, Presenter B Name & Presentation Title, Presentation Duration slider (5 minutes), Question Window slider (30 seconds), Questions Per Presenter counter (3), "Create Battle" button.
  - **Demo Data State:** Partially filled with clean synthetic topic and presenter placeholders.

### 05. Control Room Setup & Presenter Configuration
- **`05-setup-page-filled.png`**
  - **Page / Route:** `/e/[code]/control`
  - **Viewport:** Desktop `1440 × 900`
  - **State / Visible Elements:** Phase = `PENDING`. Top header showing join code, status badge, and stage display link; "Setup & Questions" tab active; Presenter A and Presenter B detail cards populated with names, titles, topics, and slide deck extraction summaries ("Extracted 4,210 characters from presentation_a.pdf"); evaluation criteria list.
  - **Demo Data State:** Battle 1 in `PENDING` state with full details filled.

### 06. Structured Question Import & Approval
- **`06-json-import-preview.png`**
  - **Page / Route:** `/e/[code]/control`
  - **Viewport:** Desktop `1440 × 900`
  - **State / Visible Elements:** Question pool management section; JSON import drawer/modal showing validated question pool; 3 candidate questions per side displaying prompt, 4 options, green highlighted correct option badge, and explanation; "Approve All" and individual question toggle buttons.
  - **Demo Data State:** Battle 1 with 3 approved questions for Side A and 3 for Side B.

### 07. Battle Readiness Verification
- **`07-readiness-checklist-complete.png`**
  - **Page / Route:** `/e/[code]/control`
  - **Viewport:** Desktop `1440 × 900`
  - **State / Visible Elements:** "Battle Readiness" status card showing all green checkmarks: Presenter A Details (Complete), Presenter B Details (Complete), Question Pool A (3 Approved), Question Pool B (3 Approved); enabled and highlighted "Determine First Speaker (Randomizer)" button.
  - **Demo Data State:** All configuration requirements satisfied.

### 08. First Speaker Randomizer Roulette
- **`08a-randomizer-spinning.png`**
  - **Page / Route:** `/e/[code]/control` (Modal)
  - **Viewport:** Desktop `1440 × 900`
  - **State / Visible Elements:** Full modal overlay displaying the canvas-rendered roulette wheel in rapid motion blur, alternating red and blue side sectors, center spin hub, and ticking audio/visual indicator.
  - **Demo Data State:** Wheel triggered and spinning.
- **`08b-randomizer-landed.png`**
  - **Page / Route:** `/e/[code]/control` (Modal)
  - **Viewport:** Desktop `1440 × 900`
  - **State / Visible Elements:** Roulette wheel stopped cleanly on winning sector (e.g., Side A); victory banner: "Presenter A1 Presents First!"; "Confirm & Open Lobby" action button; battle phase transitioning to `READY`.
  - **Demo Data State:** Wheel landed on Side A.

### 09. Mission Control Room During Live Presentation
- **`09-control-room-presenting.png`**
  - **Page / Route:** `/e/[code]/control`
  - **Viewport:** Desktop `1440 × 900`
  - **State / Visible Elements:** Phase = `presenting_a`. Active speaker indicator card illuminated; synchronized digital countdown timer (e.g., `03:42`); "+30s / +60s / -30s" time adjustment buttons; "Pause Clock" and "Trigger Understanding Check" buttons; live connected audience counter (e.g., "42 Students Connected"); upcoming auto-question schedule markers.
  - **Demo Data State:** Battle active, Presenter A speaking, clock ticking down.

### 10. Stage Projector Lobby Display
- **`10-stage-lobby.png`**
  - **Page / Route:** `/e/[code]/live`
  - **Viewport:** Stage `1920 × 1080`
  - **State / Visible Elements:** Phase = `lobby`. Fight-night arena aesthetic; large battle topic title header; Side A vs Side B challenger cards with presenter names and presentation titles; giant high-resolution QR code centered for easy auditorium scanning; prominent 6-character join code; live counter: "38 joined".
  - **Demo Data State:** Lobby phase with simulated audience connected.

### 11. Stage Projector Presentation Screen & Countdown Ring
- **`11-stage-presenting-timer.png`**
  - **Page / Route:** `/e/[code]/live`
  - **Viewport:** Stage `1920 × 1080`
  - **State / Visible Elements:** Phase = `presenting_a`. Large circular SVG countdown ring with smooth stroke dashoffset animation, glowing active speaker badge (Presenter A), presentation title, digital clock (`02:15`), and subtle upcoming checkpoint indicators.
  - **Demo Data State:** Mid-presentation of Side A.

### 12. Student Phone Live Question Answering
- **`12-phone-question-screen.png`**
  - **Page / Route:** `/e/[code]` (Participant View)
  - **Viewport:** Phone `390 × 844`
  - **State / Visible Elements:** Question break active. Top Vote Power Tracker showing current progress; clear question prompt text ("What primary symptom indicates model overfitting?"); 4 touch-friendly option cards (A, B, C, D); ticking circular countdown timer ring (e.g., 18s remaining); tap ripple feedback.
  - **Demo Data State:** Checkpoint 1 open during Presenter A talk.

### 13. Student Phone Answer Reveal & Explanations
- **`13a-phone-reveal-correct.png`**
  - **Page / Route:** `/e/[code]` (Participant View)
  - **Viewport:** Phone `390 × 844`
  - **State / Visible Elements:** Checkpoint in `revealed` status. Student's selected option highlighted in emerald green with a checkmark icon; "Correct Answer!" badge; horizontal percentage distribution bars showing audience selections across all 4 options; clear pedagogical explanation block below options.
  - **Demo Data State:** Correct answer submitted.
- **`13b-phone-reveal-wrong.png`**
  - **Page / Route:** `/e/[code]` (Participant View)
  - **Viewport:** Phone `390 × 844`
  - **State / Visible Elements:** Checkpoint in `revealed` status. Student's selected option highlighted in soft red with an "X" icon; correct option outlined in green; audience distribution bars; explanatory notes.
  - **Demo Data State:** Incorrect answer submitted.

### 14. Student Vote Power Tracker with Matched Pairs
- **`14-phone-vote-tracker-pairs.png`**
  - **Page / Route:** `/e/[code]` (Participant View)
  - **Viewport:** Phone `390 × 844`
  - **State / Visible Elements:** Between presentations or changeover. Detailed Vote Power Tracker card showing:
    - Presenter A: 2 correct out of 3 checks (green badge icons).
    - Presenter B: 2 correct out of 3 checks (green badge icons).
    - Matched Pairs banner: "2 Matched Pairs Earned = 1.2x Vote Power".
    - Informative note: "Stay engaged with both presentations to maximize your vote power."
  - **Demo Data State:** After Presenter B presentation concludes, prior to voting.

### 15. Information & Fairness Transparency Modal
- **`15-information-panel-phone.png`**
  - **Page / Route:** `/e/[code]` (Modal)
  - **Viewport:** Phone `390 × 844`
  - **State / Visible Elements:** Bottom-sheet modal explaining the core voting rule in clear visual language:
    - Formula breakdown: `1.0 + 0.1 × min(correct_A, correct_B)`.
    - Why both sides matter: Explaining that answering only for one presenter grants zero boost.
    - Privacy & fairness guarantee: Missed answers never reduce vote below 1.0x.
  - **Demo Data State:** Info modal open on mobile.

### 16. Split-Screen Mobile Voting Ballot
- **`16-phone-voting-ballot.png`**
  - **Page / Route:** `/e/[code]` (Participant View)
  - **Viewport:** Phone `390 × 844`
  - **State / Visible Elements:** Phase = `voting`. Prominent voter power callout ("Your Vote Power: 1.2x Multiplier"); split-screen selection cards for Presenter A and Presenter B with topics and photos/avatars; tap selection state with blue highlight; "Submit Official Vote" button.
  - **Demo Data State:** Voting phase open, ballot ready for submission.

### 17. Post-Vote Waiting Screen & Live Turnout
- **`17-phone-waiting-turnout.png`**
  - **Page / Route:** `/e/[code]` (Participant View)
  - **Viewport:** Phone `390 × 844`
  - **State / Visible Elements:** Ballot confirmed message ("Ballot Cast for Presenter A1!"); live turnout progress bar displaying real-time audience participation (e.g. "86% of audience has voted"); animated waiting pulse for organizer to reveal results.
  - **Demo Data State:** Ballot submitted, waiting for admin reveal.

### 18. Stage Projector Voting & Reveal Celebration
- **`18a-stage-voting-turnout.png`**
  - **Page / Route:** `/e/[code]/live`
  - **Viewport:** Stage `1920 × 1080`
  - **State / Visible Elements:** Phase = `voting`. Dramatic arena stage screen with "VOTING IN PROGRESS"; giant animated circular progress ring tracking percentage of audience ballots submitted without revealing who is winning.
  - **Demo Data State:** Voting phase with ~35 ballots submitted.
- **`18b-stage-reveal-results.png`**
  - **Page / Route:** `/e/[code]/live`
  - **Viewport:** Stage `1920 × 1080`
  - **State / Visible Elements:** Phase = `revealed`. Multi-colored canvas confetti explosion across screen; illuminated winner card (Presenter A1) crowned champion; official comparison table showing:
    - Weighted Points (e.g., 26.4 pts vs 19.8 pts).
    - Raw Votes (e.g., 22 votes vs 18 votes).
    - Engagement turnout metric (92%).
  - **Demo Data State:** Battle concluded, results officially revealed.

### 19. Analytics & Audit Log Export Workbooks
- **`19a-analytics-excel-summary.png`**
  - **Page / Route:** Excel Viewer / Spreadsheet App
  - **Viewport:** Desktop `1440 × 900`
  - **State / Visible Elements:** Open workbook view of `battleclass_all_battles.xlsx` Sheet 1 ("Tournament Summary"): professional dark slate headers, columns showing Battle Code, Topic Title, Winner, Challenger names, Allocated Times, Actual Spoken Times, Overtime, Questions Asked, Audience Size, Weighted Points A/B, and Audience Question Accuracy (%).
  - **Demo Data State:** Populated demo tournament dataset.
- **`19b-analytics-excel-attendees.png`**
  - **Page / Route:** Excel Viewer / Spreadsheet App
  - **Viewport:** Desktop `1440 × 900`
  - **State / Visible Elements:** Open workbook view of Sheet 2 ("All Attendees & Scores") and Sheet 4 ("Full Answer Log"): student names, questions answered, accuracy %, earned multipliers (`1.2x`, `1.0x`), vote timestamps, and individual question answer response times in seconds with green/red status cells.
  - **Demo Data State:** Populated demo tournament dataset.

---

## 3. Automation Feasibility Assessment

### Current Environment Status
1. **Node.js:** `v24.22.1` installed.
2. **Python:** `3.15.0` installed.
3. **Local Browsers:** Both **Google Chrome** (`C:\Program Files\Google\Chrome\Application\chrome.exe`) and **Microsoft Edge** (`C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe`) are installed locally on this machine.
4. **Automation Framework:** Neither `playwright` nor `puppeteer` is currently pre-installed in the Python virtual environment or globally in npm.

### Automated Capture Approach
We can automate all 19 screenshots end-to-end without downloading external browser binaries by using the locally installed Google Chrome or Microsoft Edge via Python Playwright:

```bash
pip install playwright
```

By specifying `channel="chrome"` or `channel="msedge"` in Playwright (`chromium.launch(channel="chrome", headless=True)`), Playwright will immediately drive the native browser binary without needing any large browser downloads.

### Automation Prerequisites
1. Local database running with the demo seed data (from Phase 3).
2. Backend running on `http://127.0.0.1:8000`.
3. Frontend running on `http://127.0.0.1:3000`.
4. A Python capture script (`scripts/capture_showcase.py`) that uses Playwright to navigate, set viewports (desktop, mobile, stage), drive transitions, and capture clean PNGs directly to `../battleclass-showcase/assets/screenshots/`.
