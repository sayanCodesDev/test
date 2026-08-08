# 🧠 IRT-Based Adaptive Learning Engine API

A production-ready, mathematically rigorous **Item Response Theory (IRT)** adaptive question selection backend built with **TypeScript**, **Express**, **Prisma 7**, and **PostgreSQL (Neon / Supabase)**.

This system dynamically estimates a student's ability ($\theta$) per topic and serves the most informative, appropriate questions in real-time using an optimized **Two-Stage 1PL Rasch IRT Model** with **Pedagogical Safeguards**.

---

## 🌟 Key Highlights & Architectural Optimizations

| Feature / Optimization | Description | Why It Matters |
|---|---|---|
| **1PL Rasch IRT Model** | $P(\text{correct}) = \text{sigmoid}(\theta - b)$ | Mathematically rigorous ability estimation centered at $\theta = 0$. |
| **Two-Stage Selection** | Stage 1: DB filtering $\to$ Stage 2: Composite ranking | Prevents loading thousands of questions into memory ($O(k)$ DB candidate generation). |
| **Progressive Difficulty Window** | Expands window ($\pm 0.25 \to \pm 0.50 \to \pm 1.00 \to \pm 2.00 \to \infty$) | Stops expanding as soon as $\ge 5$ candidates are found. Prevents over-widening. |
| **Composite IRT Ranking** | $0.70 \cdot \text{Info} + 0.20 \cdot \text{DiffProx} + 0.10 \cdot \text{Pedagogy}$ | Information is dominant, while smoothing difficulty jumps and honoring recency. |
| **Soft Difficulty Jump Limit** | $|b - \theta| \le 2.0$ logits | Prevents extreme, frustrating difficulty swings (e.g. $b=-2 \to b=+3$). |
| **Continuous Pedagogy Signal** | Recency-weighted 5-answer window | Adjusts selection target ($\theta_{\text{effective}}$) without corrupting stored mathematical $\theta$. |
| **Indexed DB Queries** | `@@index([topicId, difficulty])` & `@@index([sessionId, studentId])` | High-performance PostgreSQL range queries for large item banks. |
| **Explainable Selection** | `SelectionDiagnostics` payload returned | Human-readable explanation of why every question was selected for debugging/insights. |
| **Idempotent Answers** | `@@unique([studentId, questionId, sessionId])` | Safe against double-clicks, network retries, or duplicate submissions. |
| **115 passing tests** | 38 Math + 62 Selector + 15 Adaptive Simulation tests | 100% verified correctness and numerical stability. |

---

## 📐 How the Adaptive Engine Works

### 1. Two-Stage Question Selection Pipeline

When a student requests the next question or submits an answer, the selection process follows a two-stage architecture:

```
                          All Topic Questions in DB
                                     │
                                     ▼
      ┌─────────────────────────────────────────────────────────────┐
      │  STAGE 1: DB-Side Candidate Generation                       │
      │  - Exclude already-answered questions in current session      │
      │  - Progressive Window Search around effectiveTheta:         │
      │    Try ±0.25 → ±0.50 → ±1.00 → ±2.00 → ∞                    │
      │  - Stop as soon as MIN_CANDIDATES (5) are found             │
      └──────────────────────────────┬──────────────────────────────┘
                                     │ (only ~5-15 candidates loaded)
                                     ▼
      ┌─────────────────────────────────────────────────────────────┐
      │  STAGE 2: Composite Ranking & Filtering                     │
      │  - Compute IRT Item Information: I = P * (1 - P)            │
      │  - Compute Difficulty Proximity: e^(-|b - effectiveTheta|)  │
      │  - Compute Pedagogy Alignment Score                         │
      │  - Compute Composite Score (70% Info + 20% Diff + 10% Ped)  │
      │  - Apply Soft Max-Difficulty-Jump limit (|b - θ| ≤ 2.0)     │
      └──────────────────────────────┬──────────────────────────────┘
                                     │
                                     ▼
                        Winning Question Served
```

---

### 2. IRT Mathematical Foundation

#### A. Probability of Correct Answer
$$P(\text{correct} \mid \theta, b) = \frac{1}{1 + e^{-(\theta - b)}} = \text{sigmoid}(\theta - b)$$
* $\theta$ = Student ability ($-4.0$ to $+4.0$, centered at $0$).
* $b$ = Question difficulty ($-3.0$ to $+3.0$, centered at $0$).
* When $\theta = b \implies P = 0.5$ ($50\%$ chance of success).

#### B. Online Ability Update Rule
After every submitted answer, the student's stored ability is updated:
$$\theta_{\text{new}} = \text{clamp}\left(\theta_{\text{old}} + \alpha \cdot (\text{response} - P),\, -4.0,\, +4.0\right)$$
* $\text{response} \in \{0, 1\}$ ($1$ = correct, $0$ = incorrect).
* $\alpha = 0.3$ (learning rate — responsive yet stable).

#### C. Item Information ($I$)
$$I(\theta, b) = P \cdot (1 - P)$$
* Maximized ($0.25$) when $P = 0.50$ (question difficulty matches student ability).

---

### 3. Pedagogical Context & Safeguards

The engine separates **mathematical ability** ($\theta$) from **pedagogical adaptation** ($\theta_{\text{effective}}$):

* **Recency-Weighted Pedagogy Score**: Analyzes the last 5 answers in the session. Recent answers carry higher weights ($5, 4, 3, 2, 1$).
* **Struggling Signal (Score $< 0$)**: Temporarily biases selection toward easier questions ($\theta_{\text{effective}} = \theta - 0.5 \cdot |\text{score}|$) to rebuild confidence.
* **Streak Signal (Score $> 0$)**: Temporarily biases selection toward harder questions ($\theta_{\text{effective}} = \theta + 0.3 \cdot \text{score}$) to challenge strong performance.
* **Crucial Rule**: Pedagogical adjustments only affect question selection. The stored mathematical ability $\theta$ is **never corrupted**.

---

## 🔄 End-to-End Answer Flow

```
POST /api/sessions/:sessionId/answers  { questionId, selectedOption }
  │
  ├── 1. Validate session, question, & option parameters
  ├── 2. Idempotency Check: UNIQUE(studentId, questionId, sessionId)
  │      └── If duplicate → returns cached result without re-updating θ
  ├── 3. Server determines correctness (selectedOption === question.correctOption)
  ├── 4. Fetch current student θ for topic
  ├── 5. Compute P(correct) & update ability: θ_new = clamp(θ + α*(response - P))
  ├── 6. Store QuestionResponse audit log (thetaBefore, thetaAfter, expectedP)
  ├── 7. Upsert new θ into StudentAbility table
  ├── 8. Calculate recent response history & continuous pedagogy score
  ├── 9. Run Stage 1 DB progressive window query for next candidate set
  ├── 10. Run Stage 2 composite ranking (70% Info + 20% Diff + 10% Pedagogy)
  └── 11. Return response with correctness, updated θ, and sanitized next question
```

---

## 📁 Project Structure

```
temp_api/
├── prisma/
│   ├── schema.prisma         # Models + Composite DB Indexes
│   └── seed.ts               # Seed script for topics, questions, & students
├── prisma.config.ts          # Prisma 7 datasourceUrl configuration
├── src/
│   ├── config/
│   │   └── irt.ts            # Centralized IRT config & window/weight parameters
│   ├── db/
│   │   └── prisma.ts         # Prisma 7 singleton client with @prisma/adapter-pg
│   ├── middleware/
│   │   └── errorHandler.ts   # Express error handling middleware & ApiError
│   ├── routes/
│   │   ├── students.ts       # Student management & ability profiles
│   │   ├── topics.ts         # Topic endpoints
│   │   ├── questions.ts      # Question endpoints (scoped under topic)
│   │   ├── sessions.ts       # Quiz session & next-question endpoints
│   │   └── answers.ts        # Answer submission entry point
│   ├── services/
│   │   ├── irt.ts            # Pure IRT math (sigmoid, P, I, theta update)
│   │   ├── irtSelector.ts    # Two-stage scoring, composite ranking, pedagogy
│   │   ├── abilityService.ts # Ability database persistence
│   │   ├── questionSelector.ts # DB candidate fetcher (Stage 1) & coordinator
│   │   └── quizService.ts    # Full answer orchestrator
│   └── index.ts              # Express application entry point
└── tests/
    ├── unit/
    │   ├── irt.test.ts       # Pure IRT math unit tests (38 tests)
    │   └── questionSelector.test.ts # Two-stage selector & edge-case tests (62 tests)
    └── simulation/
        └── simulation.test.ts # Adaptive simulation for 3 student profiles (15 tests)
```

---

## 🗄️ Database Schema & Indexes

```prisma
model Question {
  id            Int      @id @default(autoincrement())
  topicId       Int
  text          String
  options       Json     // Array of options
  correctOption Int      // 0-indexed integer (never sent to client)
  difficulty    Float    // IRT b-parameter (-3.0 to +3.0)

  // Composite index for Stage 1 candidate queries
  @@index([topicId, difficulty])
}

model QuestionResponse {
  id                  Int      @id @default(autoincrement())
  sessionId           Int
  studentId           Int
  questionId          Int
  topicId             Int
  selectedOption      Int
  correct             Boolean
  thetaBefore         Float
  thetaAfter          Float
  questionDifficulty  Float
  expectedProbability Float

  @@unique([studentId, questionId, sessionId])  // Idempotency
  @@index([sessionId, studentId])                // Recent history index
}
```

---

## 🚀 Getting Started

### 1. Prerequisites
* **Node.js**: v18+
* **PostgreSQL Database** (Neon or Supabase)

### 2. Setup Environment
Create `.env` file:
```env
DATABASE_URL="postgresql://user:password@host:5432/neondb?sslmode=require"
PORT=3000
NODE_ENV=development
```

### 3. Run Database Migrations
```bash
npx prisma migrate dev
```

### 4. Seed Demo Data
Populates 3 topics (Algebra, Geometry, Physics), 19 questions across difficulty spectrum ($b \in [-2.0, +2.3]$), and 3 demo students:
```bash
npm run db:seed
```

### 5. Start Development Server
```bash
npm run dev
```

---

## 🧪 Testing & Verification

Run the full test suite (**115 passing tests** across 3 test suites):
```bash
npm test
```

### Simulation Testing (`tests/simulation/simulation.test.ts`)
Simulates 3 student types over 25 adaptive questions:
1. **Beginner** ($\theta_{\text{true}} = -1.0$)
2. **Average** ($\theta_{\text{true}} = 0.0$)
3. **Advanced** ($\theta_{\text{true}} = +1.0$)

**Verified Behaviors**:
* ✅ $\theta$ estimate converges toward true student ability.
* ✅ Advanced students receive harder questions than Beginner students on average.
* ✅ No difficulty jumps exceed max jump limit.
* ✅ Zero question repetitions within a session.
* ✅ No `NaN` or `Infinity` under any condition.

---

## 📡 API Reference

### 1. Students & Topics

#### Create Student
`POST /api/students`
```json
{ "name": "Aarav Patel", "email": "aarav@example.com" }
```

#### Get Student Profile & Topic Abilities
`GET /api/students/1`
```json
{
  "student": { "id": 1, "name": "Aarav Patel", "email": "aarav@example.com" },
  "abilities": [
    { "topicId": 1, "topicName": "Algebra", "theta": 0.35 },
    { "topicId": 2, "topicName": "Geometry", "theta": -0.20 }
  ]
}
```

---

### 2. Quiz Sessions & Adaptive Flow

#### Start Quiz Session
`POST /api/sessions`
```json
{ "studentId": 1, "topicId": 1 }
```

#### Get Next Adaptive Question
`GET /api/sessions/:sessionId/next-question`
```json
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
    "text": "What is the slope of line passing through (2,3) and (6,11)?",
    "options": ["1", "2", "3", "4"],
    "difficulty": 0.7
  },
  "nextQuestionReason": "Selected by composite IRT score [info=0.865, difficulty=0.577, pedagogy=0.500, final=0.771]. Effective θ=0.150 (neutral recent performance). Question difficulty=0.70.",
  "duplicate": false
}
```

---

## 🔒 Security & Reliability

1. **Server-Side Correctness**: `correctOption` is stripped from all API outputs via `sanitizeQuestion()`.
2. **Idempotency**: Duplicate POST requests return cached results without double-updating ability scores.
3. **Finite Safeguards**: All inputs are checked with `Number.isFinite()`, clamping, and fallback values to ensure zero runtime crashes.
