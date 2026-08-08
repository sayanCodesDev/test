# 🧠 IRT Adaptive Learning Engine API

> **A production-ready, mathematically sound, server-driven adaptive learning backend.**  
> Built with **TypeScript**, **Express**, **Prisma 7**, and **PostgreSQL** (Neon / Supabase).

---

## 🎯 What is This Project?

In traditional EdTech platforms, every student gets the exact same set of questions in the same order. This has a major flaw:
* **Easy questions bore advanced students.**
* **Hard questions frustrate beginners.**

This backend solves that problem using **Item Response Theory (IRT)** — the same mathematical framework used by standardized tests like Duolingo, GRE, and SAT. 

The engine dynamically measures a student's hidden ability score ($\theta$) for each topic in real-time, and selects questions that are **perfectly matched to their current skill level**.

---

## 🏗️ Tech Stack

* **Language**: TypeScript (Strict Mode)
* **Framework**: Express.js
* **Database ORM**: Prisma 7 (using `@prisma/adapter-pg`)
* **Database**: PostgreSQL (Compatible with Neon & Supabase)
* **Test Runner**: Jest & `ts-jest`
* **Development Server**: `tsx` (TypeScript Execute & Watch)

---

## 💡 Core Concepts & Mathematics

### 1. Ability ($\theta$) & Difficulty ($b$) Scales

Both student ability ($\theta$) and question difficulty ($b$) live on the exact same continuous numerical scale, centered at `0`:

| Value | Student Ability ($\theta$) | Question Difficulty ($b$) |
|---|---|---|
| **$-2.0$** | Beginner / Needs help | Very Easy question |
| **$-1.0$** | Below average | Easy question |
| **`0.0`** | **Average (Default start)** | **Medium difficulty question** |
| **$+1.0$** | Above average | Hard question |
| **$+2.0$** | Advanced / Expert | Very Hard question |

> 📌 **Topic Isolation**: Every student has an independent $\theta$ for *each* topic. For example, a student can have Algebra $\theta = +1.2$ (Advanced) and Geometry $\theta = -0.5$ (Beginner). Updating Algebra ability does **not** alter Geometry ability.

---

### 2. Probability of Correct Answer — 1PL Rasch Model

The probability $P$ that a student with ability $\theta$ answers a question with difficulty $b$ correctly is:

$$P(\text{correct}) = \frac{1}{1 + e^{-(\theta - b)}} = \text{sigmoid}(\theta - b)$$

* **If $\theta > b$** (Student is stronger than the question) $\to P > 0.50$ (High chance of getting it right).
* **If $\theta < b$** (Question is harder than student's skill) $\to P < 0.50$ (Low chance of getting it right).
* **If $\theta = b$** (Perfect match) $\to P = 0.50$ ($50\%$ chance of success).

---

### 3. Server-Side Online Ability Update Rule

Immediately after a student submits an answer, the server updates their stored ability:

$$\theta_{\text{new}} = \text{clamp}\left(\theta_{\text{old}} + \alpha \cdot (\text{response} - P),\, -4.0,\, +4.0\right)$$

* $\text{response} \in \{0, 1\}$ ($1$ for correct, $0$ for incorrect).
* $\alpha = 0.3$ (learning rate parameter — responsive, yet smooth).
* $\theta$ is strictly clamped between $[-4.0, +4.0]$ to guarantee numerical stability.

#### 💡 How Updates Work in Practice:
* Getting a **hard question correct** ($P$ was low) $\to$ **Large increase** in $\theta$.
* Getting an **easy question correct** ($P$ was high) $\to$ **Small increase** in $\theta$.
* Getting an **easy question wrong** ($P$ was high) $\to$ **Large decrease** in $\theta$.

---

### 4. Maximum Item Information ($I$) & Question Selection

To measure student ability as accurately as possible in the shortest time, the engine selects the candidate question that yields the highest **Item Information** $I$:

$$I(\theta, b) = P \cdot (1 - P)$$

* Information $I$ reaches its absolute peak ($0.25$) when $P = 0.50$, which occurs when **Question Difficulty $b \approx \text{Student Ability } \theta$**.
* Therefore, the system naturally seeks out questions at the student's exact frontier of learning.

---

## ⚡ Question Selection Pipeline & Optimizations

To handle large question banks efficiently without slowing down or making jarring difficulty jumps, question selection follows a **Two-Stage Architecture**:

```
                          All Topic Questions in DB
                                     │
                                     ▼
      ┌─────────────────────────────────────────────────────────────┐
      │  STAGE 1: DB-Side Candidate Generation                       │
      │  - Exclude questions already answered in current session    │
      │  - Progressive Window Search around effectiveTheta:         │
      │    Try ±0.25 → ±0.50 → ±1.00 → ±2.00 → ∞                    │
      │  - Stop as soon as MIN_CANDIDATES (5) are found             │
      └──────────────────────────────┬──────────────────────────────┘
                                     │ (only ~5-15 candidates loaded)
                                     ▼
      ┌─────────────────────────────────────────────────────────────┐
      │  STAGE 2: Composite Ranking & Soft Constraints              │
      │  - Compute IRT Item Information: I = P * (1 - P)            │
      │  - Compute Difficulty Proximity: e^(-|b - effectiveTheta|)  │
      │  - Compute Pedagogy Alignment Score (recency weighted)      │
      │  - Composite Score: 70% Info + 20% Diff + 10% Pedagogy      │
      │  - Soft Max-Difficulty-Jump Limit (|b - θ| ≤ 2.0)           │
      └──────────────────────────────┬──────────────────────────────┘
                                     │
                                     ▼
                        Winning Question Served
```

### Key Optimizations:
1. **DB-Side Candidate Filtering (Stage 1)**: Queries filtering by `topicId` and `difficulty` happen directly in PostgreSQL using indexes. Avoids loading thousands of questions into memory.
2. **Progressive Difficulty Window**: Search window expands incrementally ($\pm 0.25 \to \pm 0.50 \to \pm 1.00 \to \pm 2.00 \to \infty$) and stops widening as soon as 5 candidate questions are found.
3. **Normalized Composite Score**:
   $$\text{Final Score} = 0.70 \cdot \text{InfoScore} + 0.20 \cdot \text{DiffScore} + 0.10 \cdot \text{PedagogyScore}$$
4. **Soft Difficulty Jump Constraint**: Prevents sudden jumps where $b$ is more than $2.0$ logits away from the student's current score.
5. **Continuous Pedagogical Safeguards**: Analyzes the last 5 responses using a recency-weighted average. If a student is struggling (consecutive wrong answers), the target $\theta_{\text{effective}}$ temporarily shifts downward to serve easier questions. *This only affects question selection and never corrupts the true stored $\theta$.*

---

## 🗄️ Database Schema & Models

The database consists of 6 core models:

```mermaid
erDiagram
    Topic ||--o{ Question : contains
    Topic ||--o{ StudentAbility : tracks
    Topic ||--o{ QuizSession : "session topic"
    Student ||--o{ StudentAbility : possesses
    Student ||--o{ QuizSession : starts
    QuizSession ||--o{ QuestionResponse : logs
    Question ||--o{ QuestionResponse : answered
```

### Models & Key Fields:
* **`Topic`**: Unique topic name (e.g., "Algebra", "Geometry").
* **`Question`**: Belongs to a Topic. Stores `text`, `options` (JSON array), `correctOption` (0-indexed integer), and `difficulty` ($b \in [-3, +3]$).
  * Has composite index `@@index([topicId, difficulty])`.
* **`Student`**: User entity (`name`, `email`).
* **`StudentAbility`**: Unique per `(studentId, topicId)`. Stores current `theta` ($\theta$). Defaults to `0`.
* **`QuizSession`**: Session container (`studentId`, `topicId`, `startedAt`, `endedAt`).
* **`QuestionResponse`**: Full audit log for every submitted answer.
  * Stores `correct`, `thetaBefore`, `thetaAfter`, `questionDifficulty`, and `expectedProbability`.
  * Has unique constraint `@@unique([studentId, questionId, sessionId])` for idempotency.
  * Has composite index `@@index([sessionId, studentId])`.

---

## 🔄 End-to-End Answer Flow

The backend acts as the **single source of truth**. The frontend never calculates correctness, $\theta$, or the next question.

```
POST /api/sessions/:sessionId/answers  { questionId, selectedOption }
  │
  ├── 1. Validate session & question existence
  ├── 2. Idempotency Check: UNIQUE(studentId, questionId, sessionId)
  │      └── If duplicate → returns cached result without re-updating θ
  ├── 3. Server determines correctness (selectedOption === question.correctOption)
  ├── 4. Load current student θ for topic
  ├── 5. Compute P(correct) & update ability: θ_new = clamp(θ + α*(response - P))
  ├── 6. Create QuestionResponse audit log (thetaBefore, thetaAfter, expectedP)
  ├── 7. Persist updated θ into StudentAbility table
  ├── 8. Calculate recency-weighted continuous pedagogy score
  ├── 9. Run Stage 1 DB progressive window query for candidate questions
  ├── 10. Run Stage 2 composite ranking (70% Info + 20% Diff + 10% Pedagogy)
  └── 11. Return response with correctness, updated θ, and sanitized next question
```

---

## 🚀 Installation & Local Setup

### 1. Prerequisites
* **Node.js**: v18 or higher
* **PostgreSQL Database**: Neon DB or Supabase instance

### 2. Clone & Install Dependencies
```bash
cd temp_api
npm install
```

### 3. Environment Configuration
Create a `.env` file in the root directory:
```env
DATABASE_URL="postgresql://user:password@host:5432/neondb?sslmode=require"
PORT=3000
NODE_ENV=development
```

### 4. Run Database Migrations
Push the Prisma schema and create database indexes:
```bash
npx prisma migrate dev
```

### 5. Seed Demo Data
Populate demo topics (Algebra, Geometry, Physics), 19 difficulty-calibrated questions ($b \in [-2.0, +2.3]$), and 3 demo students:
```bash
npm run db:seed
```

### 6. Start Server
* **Development mode** (with auto-reload):
  ```bash
  npm run dev
  ```
* **Build for Production**:
  ```bash
  npm run build
  npm start
  ```

---

## 🧪 Testing & Verification

The project includes **115 automated tests** covering math, selector logic, edge cases, and multi-student adaptive simulations.

Run all tests:
```bash
npm test
```

### Test Suite Summary:
* `tests/unit/irt.test.ts` (38 tests): Validates sigmoid stability, $P(\theta, b)$, Item Information $I$, clamping, and $\theta$ update math.
* `tests/unit/questionSelector.test.ts` (62 tests): Validates two-stage window widening, composite scoring, continuous pedagogy, max-jump limits, and 20 edge cases.
* `tests/simulation/simulation.test.ts` (15 tests): Runs 3 simulated students (Beginner $\theta=-1$, Average $\theta=0$, Advanced $\theta=+1$) through 25 adaptive questions to verify convergence, zero question repetitions, and smooth difficulty progression.

---

## 📡 API Reference

### 1. Students & Profile

#### Create Student
`POST /api/students`
```json
// Request
{ "name": "Aarav Patel", "email": "aarav@example.com" }

// Response (201 Created)
{
  "student": {
    "id": 1,
    "name": "Aarav Patel",
    "email": "aarav@example.com",
    "createdAt": "2026-08-08T10:00:00.000Z"
  }
}
```

#### Get Student Profile & Abilities
`GET /api/students/:id`
```json
// Response (200 OK)
{
  "student": { "id": 1, "name": "Aarav Patel", "email": "aarav@example.com" },
  "abilities": [
    { "topicId": 1, "topicName": "Algebra", "theta": 0.35 },
    { "topicId": 2, "topicName": "Geometry", "theta": -0.20 }
  ]
}
```

---

### 2. Topics & Questions

#### Create Topic
`POST /api/topics`
```json
{ "name": "Algebra" }
```

#### Add Question to Topic
`POST /api/topics/:topicId/questions`
```json
// Request
{
  "text": "Solve for x: 2x + 4 = 10",
  "options": ["x = 2", "x = 3", "x = 4", "x = 5"],
  "correctOption": 1,
  "difficulty": 0.0
}

// Response (201 Created) — correctOption is NEVER exposed to the client
{
  "question": {
    "id": 4,
    "topicId": 1,
    "text": "Solve for x: 2x + 4 = 10",
    "options": ["x = 2", "x = 3", "x = 4", "x = 5"],
    "difficulty": 0.0,
    "createdAt": "2026-08-08T10:00:00.000Z"
  }
}
```

---

### 3. Quiz Sessions & Adaptive Flow

#### Start Quiz Session
`POST /api/sessions`
```json
// Request
{ "studentId": 1, "topicId": 1 }

// Response (201 Created)
{
  "session": { "id": 101, "studentId": 1, "topicId": 1, "startedAt": "..." },
  "student": { "id": 1, "name": "Aarav Patel" },
  "topic": { "id": 1, "name": "Algebra" }
}
```

#### Get Next Adaptive Question
`GET /api/sessions/:sessionId/next-question`
```json
// Response (200 OK)
{
  "question": {
    "id": 4,
    "topicId": 1,
    "text": "Find the roots of x² - 5x + 6 = 0",
    "options": ["x = 1, 6", "x = 2, 3", "x = -2, -3", "x = 0, 5"],
    "difficulty": 0.0
  },
  "theta": 0.0,
  "effectiveTheta": 0.0,
  "reason": "Selected by composite IRT score [info=1.000, difficulty=1.000, pedagogy=0.500, final=0.950]. Effective θ=0.000 (neutral recent performance). Question difficulty=0.00."
}
```

#### Submit Answer (Main Adaptive Endpoint)
`POST /api/sessions/:sessionId/answers`
```json
// Request
{
  "questionId": 4,
  "selectedOption": 1
}

// Response (201 Created)
{
  "correct": true,
  "thetaBefore": 0.0,
  "thetaAfter": 0.15,
  "expectedProbability": 0.50,
  "information": 0.25,
  "effectiveTheta": 0.15,
  "nextQuestion": {
    "id": 5,
    "topicId": 1,
    "text": "What is the slope of the line passing through (2,3) and (6,11)?",
    "options": ["1", "2", "3", "4"],
    "difficulty": 0.7
  },
  "nextQuestionReason": "Selected by composite IRT score [info=0.865, difficulty=0.577, pedagogy=0.500, final=0.771]. Effective θ=0.150 (neutral recent performance). Question difficulty=0.70.",
  "duplicate": false
}
```

---

## 🔒 Security & Reliability

1. **Information Leak Prevention**: `correctOption` is stripped from all API payloads via `sanitizeQuestion()`.
2. **Idempotent Submissions**: Duplicate answer POST requests return the cached result without double-updating ability scores or corrupting session logs.
3. **Zero-Crash Numerical Guards**: All math functions check `Number.isFinite()`, clamp extreme values, and handle edge cases gracefully to ensure no `NaN` or `Infinity` reaches the database.
