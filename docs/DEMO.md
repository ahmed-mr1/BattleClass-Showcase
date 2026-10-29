# Generating Safe Demo Data (Local Sandbox)

This guide explains how to populate a local development instance with synthetic demo data for testing, taking screenshots, and verifying analytics reports without exposing any real student or presenter information.

---

## 1. Safety Guarantees

The demo seed script (`scripts/seed_demo.py`) implements strict privacy and safety guardrails:

- **Production Safeguard:** The script checks environment configuration before touching the database and immediately halts execution if `ENV=production`.
- **Zero Real Personal Data:** All entities use obviously synthetic names (e.g. `Presenter A1`, `Student 01`, `Student 40`).
- **Academic Topics:** All battle topics and quiz questions are synthetic pedagogical examples from machine learning and numerical optimization.
- **Local-Only Admin Account:** Generates a secure, temporary local admin credential (`demo@example.com`) and outputs the password once to standard output.
- **Idempotent Reset:** Safely purges and replaces previous demo battle codes (`DEMO01`, `DEMO02`, `DEMO03`) on each run.

---

## 2. Seeded Dataset Structure

The script populates **3 representative battles** in distinct lifecycle states:

### Battle 1: Concluded & Revealed Battle (`DEMO01`)
- **Topic:** *Gradient Descent vs Newton-Raphson*
- **Presenters:**
  - **Presenter A1:** *First-Order Optimization: Scalability of SGD* (Allocated: 5:00, Actual: 4:55, Overtime: 0:00).
  - **Presenter B1:** *Second-Order Curvature: Newton-Raphson Precision* (Allocated: 5:00, Extra Time: +30s, Actual: 6:15, Overtime: 0:45).
- **Checkpoints & Questions:**
  - 10 total approved questions (5 per side).
  - 6 checkpoints executed during the battle (3 per side).
  - **Difficulty Outlier:** Question 2A (*"Under what specific condition is the Hessian matrix guaranteed to be positive semi-definite everywhere?"*) was answered correctly by only ~18% of students, providing authentic accuracy variation for analytics.
- **Audience Attendance & Timing:**
  - **40 simulated participants** (`Student 01` to `Student 40`).
  - **8 Late Joiners** (`Student 33` to `Student 40`): Joined after Presenter A1 completed, illustrating the fairness rule (zero matched pairs earned on Side A, baseline 1.0x voting power).
- **Voting & Scoring:**
  - 37 ballots cast (~92.5% turnout).
  - Individual vote multipliers range from `1.0x` (0 pairs) to `1.3x` (3 matched pairs).
  - Full results calculated with weighted scores, raw votes, and turnout metrics.

### Battle 2: Setup Complete & Ready for Presentation (`DEMO02`)
- **Topic:** *CNNs vs Vision Transformers*
- **Presenters:** `Presenter A2` vs `Presenter B2`.
- **State:** `READY` (Readiness checklist 100% green; 5 approved questions per side; ready to spin the randomizer wheel or open the lobby).

### Battle 3: Initial Draft Setup (`DEMO03`)
- **Topic:** *Supervised Fine-Tuning vs RLHF*
- **Presenters:** `Presenter A3` vs `Presenter B3`.
- **State:** `PENDING` (Draft mode; questions in draft state; ideal for demonstrating question authoring and JSON import flows).

---

## 3. How to Run Locally

### Prerequisites
1. Local PostgreSQL running (or containerized via Docker).
2. Local backend virtual environment active with dependencies installed.

### Execution Steps

1. **Apply Latest Database Migrations:**
   ```bash
   cd backend
   alembic upgrade head
   cd ..
   ```

2. **Execute the Demo Seeder:**
   *(Ensure `ENV` is explicitly set to `development`)*

   - **Linux / macOS:**
     ```bash
     ENV=development python scripts/seed_demo.py
     ```

   - **Windows (PowerShell):**
     ```powershell
     $env:ENV="development"; python scripts/seed_demo.py; $env:ENV=""
     ```

3. **Console Output:**
   The script outputs a summary and displays the generated admin credentials:

   ```text
   ============================================================
   BattleClass: Seeding Local Database with Synthetic Demo Data
   ============================================================
   Created new demo admin: demo@example.com
   Purged previous demo battle records: ['DEMO01', 'DEMO02', 'DEMO03']
   ------------------------------------------------------------
   DEMO SEED COMPLETED SUCCESSFULLY:
     Battle 1: DEMO01 [REVEALED] - 'Gradient Descent vs Newton-Raphson'
               40 Students, Overtime on Presenter B1, 1 Difficult Question
     Battle 2: DEMO02 [READY]    - 'CNNs vs Vision Transformers'
               5 Approved Questions per side, Ready for Lobby
     Battle 3: DEMO03 [PENDING]  - 'Supervised Fine-Tuning vs RLHF'
               Draft questions for setup & import demonstrations
   ------------------------------------------------------------
   LOCAL ADMIN CREDENTIALS (Printed Once):
     Email:    demo@example.com
     Password: DemoPass!xxxx
   ============================================================
   ```

4. **Verify in the Web Application:**
   - Start the backend: `uvicorn app.main:app --port 8000`
   - Start the frontend: `npm run dev` (in `frontend/`)
   - Navigate to `http://localhost:3000/login` and log in with the printed credentials.
   - Access `/admin` to review the 3 demo battles, or open `/e/DEMO01/live` to view the revealed stage presentation.
