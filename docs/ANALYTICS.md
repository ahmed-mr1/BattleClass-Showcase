# Analytics & Reporting Specifications

This document defines every reporting endpoint, export format, data column, and metric calculation implemented in BattleClass.

---

## 1. Reporting Endpoints

| Endpoint | Method | Auth Required | Output Format | Description |
| :--- | :--- | :--- | :--- | :--- |
| `/api/reports/battles/{code}/export` | `GET` | Admin Session | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | Comprehensive multi-sheet Excel workbook for a single battle. |
| `/api/reports/tournament/summary` | `GET` | Admin Session | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | Rollup Excel report covering all battles across the tournament roster. |

---

## 2. Single-Battle Report Structure (`battle_{code}_report.xlsx`)

The single-battle export contains 5 dedicated worksheets detailing every dimension of the session:

### Sheet 1: Battle Summary
Summarizes high-level battle parameters, presentation timings, aggregate voting, and winner determinations.

| Column Header | Data Source / Calculation | Definition |
| :--- | :--- | :--- |
| `Battle Code` | `battles.code` | Unique battle identifier (e.g., `DEMO01`). |
| `Topic Title` | `battles.title` | Full subject matter or debate prompt. |
| `Status` | `battles.state` | Final lifecycle phase (`REVEALED` / `CLOSED`). |
| `Winner` | Computed | Challenger side (`A` or `B`) achieving higher total weighted points. |
| `Challenger 1 (Side A)` | `presenters[A].name` | Presenter name for Side A. |
| `Challenger 2 (Side B)` | `presenters[B].name` | Presenter name for Side B. |
| `Allocated Time A` | `presenters[A].duration_ms` | Scheduled speaking budget in `m:ss` format. |
| `Allocated Time B` | `presenters[B].duration_ms` | Scheduled speaking budget in `m:ss` format. |
| `Actual Spoken A` | `presenters[A].spoken_ms` | Total active speaking time elapsed in `m:ss` format. |
| `Actual Spoken B` | `presenters[B].spoken_ms` | Total active speaking time elapsed in `m:ss` format. |
| `Overtime A` | `presenters[A].overtime_ms` | Additional speaking time consumed beyond budget (0s if under budget). |
| `Overtime B` | `presenters[B].overtime_ms` | Additional speaking time consumed beyond budget (0s if under budget). |
| `Total Attendees` | `COUNT(participants.id)` | Total students registered in the battle lobby. |
| `Total Ballots Cast` | `COUNT(ballots.id)` | Total distinct voting ballots submitted. |
| `Turnout Rate (%)` | `Total Ballots / Total Attendees` | Audience participation percentage. |
| `Raw Votes A` | `SUM(ballot votes for A)` | Unweighted vote count across all criteria. |
| `Raw Votes B` | `SUM(ballot votes for B)` | Unweighted vote count across all criteria. |
| `Weighted Points A` | `SUM(ballot votes A * voter_weight)` | Final weighted points allocated to Side A. |
| `Weighted Points B` | `SUM(ballot votes B * voter_weight)` | Final weighted points allocated to Side B. |
| `Overall Accuracy (%)` | `SUM(correct) / SUM(answers)` | Percentage of correct understanding check responses across all questions. |

### Sheet 2: Attendee Turnout & Weights
Provides per-student participation, accuracy by presenter, earned voting weight, and ballot submission status.

| Column Header | Definition |
| :--- | :--- |
| `Participant ID` | Internal unique attendee identifier. |
| `Student Name` | Name entered by attendee upon joining. |
| `Joined At (UTC)` | Timestamp of lobby entry. |
| `Total Answered` | Total comprehension check questions attempted. |
| `Total Correct` | Total comprehension check questions answered correctly. |
| `Answered (Side A)` | Questions attempted during Presenter A's talk. |
| `Correct (Side A)` | Correct answers scored during Presenter A's talk. |
| `Answered (Side B)` | Questions attempted during Presenter B's talk. |
| `Correct (Side B)` | Correct answers scored during Presenter B's talk. |
| `Matched Pairs` | `min(Correct_A, Correct_B)`: balanced comprehension demonstrated across both presenters. |
| `Earned Vote Weight` | Calculated vote weight applied to the student's ballot (e.g., 0.5x, 1.0x, 1.3x). |
| `Ballot Cast?` | Boolean (`YES` / `NO`) indicating whether the attendee submitted a final ballot. |
| `Ballot Cast At` | Timestamp when ballot was received. |

### Sheet 3: Question Performance
Item analysis for each comprehension check delivered during the session.

| Column Header | Definition |
| :--- | :--- |
| `Question ID` | Unique question identifier from question bank. |
| `Side` | Presenter affiliation (`A` or `B`). |
| `Presenter` | Presenter name who delivered the question. |
| `Question Prompt` | Verbatim text of the multiple-choice question. |
| `Correct Option` | Designated correct choice (`A`, `B`, `C`, or `D`). |
| `Count A / B / C / D` | Number of student selections for each respective option. |
| `Total Responses` | Total number of students who submitted an answer for this checkpoint. |
| `Correct Responses` | Number of students who selected the correct option. |
| `Accuracy (%)` | `Correct Responses / Total Responses * 100`. |
| `Avg Response Time (s)` | Mean submission latency from question open timestamp. |

### Sheet 4: Ballot Registry
Individual ballot audit record showing multi-criteria preferences and applied vote weighting.

| Column Header | Definition |
| :--- | :--- |
| `Ballot ID` | Unique submission identifier. |
| `Submitted At` | UTC timestamp of ballot submission. |
| `Student Name` | Name of submitting participant. |
| `Applied Weight` | Student's earned vote multiplier at ballot time. |
| `Clarity Vote` | Selected challenger for Criterion 1 (`A` or `B`). |
| `Technical Depth Vote` | Selected challenger for Criterion 2 (`A` or `B`). |
| `Argument Strength Vote` | Selected challenger for Criterion 3 (`A` or `B`). |

### Sheet 5: Raw Answer Submissions
Detailed transaction log of every single question response submitted during the session.

| Column Header | Definition |
| :--- | :--- |
| `Submission ID` | Unique response log identifier. |
| `Timestamp (UTC)` | Exact time the answer was received by the server. |
| `Question ID` | Checkpoint question identifier. |
| `Presenter` | Presenter whose talk was being checked. |
| `Student Name` | Submitting attendee name. |
| `Selected Option` | Option selected by student. |
| `Result` | `CORRECT` or `INCORRECT`. |
| `Latency (s)` | Elapsed seconds between question launch and answer submission. |

---

## 3. Metric Calculations (Code Reality)

### 3.1 Attendance & Participation
- **Student Registered:** An attendee who loaded the battle URL, submitted a display name, and received a `participant_token`.
- **Active Participant:** A registered attendee who attempted at least one comprehension check during the session.
- **Voter:** A registered attendee who submitted a final ballot during the `VOTING` phase.

### 3.2 Time Budget Compliance
- **Overtime Calculation:**
  $$\text{Overtime (ms)} = \max(0, \text{Actual Spoken Time} - \text{Allocated Duration})$$
- **Code Reality:** Overtime is tracked in milliseconds and recorded in the summary report. The system does not automatically deduct numerical points from a presenter's score for overtime; the data is provided in the export for instructor evaluation and manual course penalties.

### 3.3 Topic & Question Difficulty Analysis
- **Per-Question Accuracy:**
  $$\text{Accuracy} = \frac{\text{Correct Submissions}}{\text{Total Submissions for Question}} \times 100$$
- **Code Reality:** The system does not automatically flag or categorize topics as "struggled with" using machine learning or automated classification rules. Instead, instructors sort the `Question Performance` sheet by `Accuracy (%)` to identify specific conceptual topics where audience comprehension was low.

### 3.4 Vote Weighting Rule
To prevent partisan voting where students pay attention only to their friend or preferred topic:
- **Baseline Weight:** If a student answers zero questions, their vote weight defaults to 1.0 (or is excluded based on tournament configuration).
- **Single-Side Penalty:** If a student answers questions correctly for only one presenter (e.g., 3 correct for A, 0 correct for B), their vote weight is capped at **0.5x**.
- **Balanced Attention Reward:** When a student demonstrates comprehension for both speakers ($\text{Correct}_A \ge 1 \text{ and } \text{Correct}_B \ge 1$), their vote receives full **1.0x** weight plus a bonus for additional matched pairs:
  $$\text{Weight} = 1.0 + (\min(\text{Correct}_A, \text{Correct}_B) - 1) \times 0.1$$
  *(Capped at maximum configurable tournament weight, default 1.3x)*.
