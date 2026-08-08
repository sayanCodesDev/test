# 🧠 IRT Adaptive Learning Engine API

A production-ready, mathematically rigorous **Item Response Theory (IRT)** adaptive learning backend built with **TypeScript**, **Express**, **Prisma 7**, and **Supabase (PostgreSQL)**.

This backend dynamically estimates a student's ability ($\theta$) per topic and serves the most informative, appropriate questions in real time using the **1-Parameter Logistic (1PL) Rasch IRT Model**.

---

## 📐 How It Works (IRT Mathematical Foundation)

### 1. Ability ($\theta$) & Question Difficulty ($b$) Scales
* Both **Student Ability** ($\theta$) and **Question Difficulty** ($b$) exist on a continuous logit scale centered at `0`:
  * $\theta = 0 \to$ Average ability / default start for new topic
  * $\theta < 0 \to$ Weaker ability (e.g., $-1.5$ = beginner)
  * $\theta > 0 \to$ Stronger ability (e.g., $+1.5$ = advanced)
  * $b \in [-3, +3] \to$ Question difficulty ($-2$ easy, $0$ medium, $+2$ hard)
* **Topic Isolation**: Every student maintains an independent $\theta$ for *each* topic (e.g., Algebra $\theta = +0.8$, Geometry $\theta = -0.4$). Updating Algebra ability does **not** affect Geometry.

---

### 2. Probability of Correct Answer — 1PL Rasch Model
The probability $P$ that a student with ability $\theta$ answers a question of difficulty $b$ correctly is:

$$P(\text{correct}) = \frac{1}{1 + e^{-(\theta - b)}} = \text{sigmoid}(\theta - b)$$

* Implemented using a **numerically stable sigmoid** function clamped to prevent overflow or NaN/Infinity values.
* When $\theta = b \implies P = 0.5$ ($50\%$ chance of success).

---

### 3. Online Ability Update Rule
After every submitted answer, the student's ability is updated immediately on the server:

$$\theta_{\text{new}} = \text{clamp}\left(\theta_{\text{old}} + \alpha \cdot (\text{response} - P),\, \theta_{\text{min}},\, \theta_{\text{max}}\right)$$

* $\text{response} \in \{0, 1\}$ ($1$ = correct, $0$ = incorrect).
* $\alpha = 0.3$ (configurable learning rate — responsive yet stable).
* $\theta$ is strictly clamped within $[-4.0, +4.0]$.
* **Intuition**:
  * Correct answer on a hard question ($P$ low) $\to$ Larger increase in $\theta$.
  * Correct answer on an easy question ($P$ high) $\to$ Minor increase in $\theta$.
  * Incorrect answer on an easy question ($P$ high) $\to$ Larger decrease in $\theta$.

---

### 4. Maximum Item Information Question Selection
To maximize measurement precision, the engine selects the available candidate question that provides the highest **Item Information** $I(\theta)$:

$$I(\theta) = P \cdot (1 - P)$$

* Information peak occurs at $P = 0.5$ (when $b \approx \theta$).
* The system naturally selects questions matching the student's current skill level.
* **Already answered questions in the session are strictly excluded** so questions are never repeated.

---

### 5. Pedagogical Safeguards
To make the learning experience natural and supportive:
* **Struggle Safeguard**: If the student answers $3$ consecutive questions incorrectly in a session, selection uses an effective target $\theta_{\text{eff}} = \theta - 0.5$ (serving slightly easier questions to build confidence).
* **Streak Safeguard**: If the student answers $3$ consecutive questions correctly, selection uses $\theta_{\text{eff}} = \theta + 0.3$ (gradually challenging strong performance).
* *Note: Safeguards only affect selection scoring and do not corrupt the true stored $\theta$.*

---

## 🔄 End-to-End Server Answer Flow

The backend is the **single source of truth**. The client/frontend NEVER determines correctness, $\theta$, or the next question.

```
POST /api/sessions/:sessionId/answers  { questionId, selectedOption }
  │
  ├── 1. Server validates request parameters
  ├── 2. Loads Session & Question from DB
  ├── 3. Idempotency Check: UNIQUE(studentId, questionId, sessionId)
  │      └── If already answered → returns cached result (no double θ update)
  ├── 4. Server determines correctness (selectedOption === question.correctOption)
  ├── 5. Server fetches current student θ for the topic
  ├── 6. Calculates probability P = sigmoid(θ - b) and information I = P*(1-P)
  ├── 7. Computes new ability: θ_new = clamp(θ + α*(response - P))
  ├── 8. Persists QuestionResponse log (stores thetaBefore, thetaAfter, expectedP)
  ├── 9. Upserts new θ into StudentAbility
  ├── 10. Computes effective theta with recent performance safeguards
  ├── 11. Ranks all unanswered topic questions by maximum item information
  └── 12. Returns response with correctness, updated θ, and next sanitized question
```

---

## 🛠️ Architecture & Tech Stack

```
temp_api/
├── prisma/
│   └── schema.prisma         # Database models (Topic, Question, Student, Ability, Session, Response)
├── prisma.config.ts          # Prisma 7 datasourceUrl configuration
├── src/
│   ├── config/
│   │   └── irt.ts            # Centralized IRT constants & parameters
│   ├── db/
│   │   └── prisma.ts         # Singleton Prisma 7 database client
│   ├── middleware/
│   │   └── errorHandler.ts   # Express error handling middleware & ApiError
│   ├── routes/
│   │   ├── students.ts       # Student management & ability profile endpoints
│   │   ├── topics.ts         # Topic creation & listing endpoints
│   │   ├── questions.ts      # Question management endpoints (scoped under topic)
│   │   ├── sessions.ts       # Quiz session endpoints & next-question fetch
│   │   └── answers.ts        # Answer submission & adaptive loop entry point
│   ├── services/
│   │   ├── irt.ts            # Pure IRT math (sigmoid, P, I, theta update)
│   │   ├── irtSelector.ts    # Pure adaptive scoring & ranking algorithms
│   │   ├── abilityService.ts # Topic ability database persistence
│   │   ├── questionSelector.ts # DB candidate question fetching & filtering
│   │   └── quizService.ts    # Full quiz flow orchestrator
│   └── index.ts              # Express application entry point
└── tests/
    ├── unit/
    │   ├── irt.test.ts       # Pure IRT math unit tests (38 tests)
    │   └── questionSelector.test.ts # Adaptive selector unit tests (14 tests)
    └── simulation/
        └── simulation.test.ts # 3-student probabilistic adaptive simulation (11 tests)
```

---

## 📊 Database Schema (Prisma Models)

* **`Topic`**: Subject area (e.g. "Algebra").
* **`Question`**: Belongs to a Topic. Contains `text`, `options` (JSON), `correctOption` (0-indexed integer), and `difficulty` ($b \in [-3, +3]$).
* **`Student`**: User entity (`name`, `email`).
* **`StudentAbility`**: Unique `(studentId, topicId)` composite key storing `theta` ($\theta$). Default = `0`.
* **`QuizSession`**: Tracks a learning session (`studentId`, `topicId`, `startedAt`, `endedAt`).
* **`QuestionResponse`**: Full audit log for every submitted answer:
  * `(studentId, questionId, sessionId)` has a **UNIQUE constraint** guaranteeing idempotency (duplicate POSTs will not double-update ability).
  * Stores `correct`, `thetaBefore`, `thetaAfter`, `questionDifficulty`, and `expectedProbability`.

---

## 🚀 Getting Started

### 1. Prerequisites
* **Node.js**: v18+
* **npm**: v9+
* **PostgreSQL Database** (e.g., [Supabase](https://supabase.com))

### 2. Environment Setup
Copy `.env.example` to `.env` and set your Supabase database connection string:

```bash
cp .env.example .env
```

Set your `DATABASE_URL` in `.env`:
```env
DATABASE_URL="postgresql://postgres:[PASSWORD]@[HOST]:5432/postgres"
PORT=3000
NODE_ENV=development
```

### 3. Database Migration
Push the Prisma schema to your PostgreSQL database:

```bash
npx prisma migrate dev --name init
```

Generate the Prisma Client:
```bash
npx prisma generate
```

### 4. Running the Application

* **Development mode** (with auto-reload):
  ```bash
  npm run dev
  ```

* **Build for production**:
  ```bash
  npm run build
  npm start
  ```

* **Type Check**:
  ```bash
  npm run typecheck
  ```

---

## 🧪 Testing & Simulation

The test suite includes **63 automated tests** verifying IRT calculations, safeguards, edge cases, and adaptive simulation.

Run all tests:
```bash
npm test
```

### Simulation Test
`tests/simulation/simulation.test.ts` runs 3 simulated students through 25 adaptive questions:
1. **Beginner** ($\theta_{\text{true}} = -1.0$)
2. **Average** ($\theta_{\text{true}} = 0.0$)
3. **Advanced** ($\theta_{\text{true}} = +1.0$)

The simulation probabilistically generates answer correctness using $P(\text{correct})$ and verifies that:
* Estimated $\theta$ converges toward true ability.
* Advanced students receive harder questions on average than Beginner students.
* Question difficulty adapts smoothly without single-answer wild jumps.
* No `NaN` or `Infinity` reaches state or database.

---

## 📡 API Reference

### 1. Students

#### Create Student
`POST /api/students`
```json
// Request
{
  "name": "Riya Sharma",
  "email": "riya@example.com"
}

// Response (201 Created)
{
  "student": {
    "id": 1,
    "name": "Riya Sharma",
    "email": "riya@example.com",
    "createdAt": "2026-08-08T10:00:00.000Z"
  }
}
```

#### Get Student Profile & Topic Abilities
`GET /api/students/:id`
```json
// Response (200 OK)
{
  "student": { "id": 1, "name": "Riya Sharma", "email": "riya@example.com" },
  "abilities": [
    { "topicId": 1, "topicName": "Algebra", "theta": 0.45 },
    { "topicId": 2, "topicName": "Geometry", "theta": -0.30 }
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

// Response (201 Created) — correctOption is NEVER returned to client
{
  "question": {
    "id": 10,
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
{
  "studentId": 1,
  "topicId": 1
}

// Response (201 Created)
{
  "session": { "id": 101, "studentId": 1, "topicId": 1, "startedAt": "..." },
  "student": { "id": 1, "name": "Riya Sharma" },
  "topic": { "id": 1, "name": "Algebra" }
}
```

#### Get Next Adaptive Question
`GET /api/sessions/:sessionId/next-question`
```json
// Response (200 OK)
{
  "question": {
    "id": 10,
    "topicId": 1,
    "text": "Solve for x: 2x + 4 = 10",
    "options": ["x = 2", "x = 3", "x = 4", "x = 5"],
    "difficulty": 0.0
  },
  "theta": 0.0,
  "effectiveTheta": 0.0,
  "reason": "Selected by maximum item information (Rasch 1PL)"
}
```

#### Submit Answer (Main Adaptive Endpoint)
`POST /api/sessions/:sessionId/answers`
```json
// Request
{
  "questionId": 10,
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
    "id": 14,
    "topicId": 1,
    "text": "Solve for x: x^2 - 9 = 0",
    "options": ["x = ±3", "x = 3", "x = 9", "x = 0"],
    "difficulty": 0.2
  },
  "nextQuestionReason": "Selected by maximum item information (Rasch 1PL)",
  "duplicate": false
}
```

---

## 🔒 Security & Reliability Guarantees

1. **Idempotency**: Duplicate HTTP calls (e.g. user double-clicking submit or network retry) match the `@@unique([studentId, questionId, sessionId])` constraint and safely return the cached result without double-updating ability.
2. **Server Authority**: Answer correctness is evaluated purely on the server. `correctOption` is never sent in question payloads.
3. **Numerical Safety**: All IRT inputs/outputs pass through finite validation and clamping to prevent `NaN` or `Infinity` from reaching the database.
