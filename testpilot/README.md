# TestPilot

An AI-powered TDD assistant that generates, runs, and evaluates JUnit tests for Java code. Paste your source code, describe what you want tested, and a multi-agent pipeline writes tests, executes them with real Gradle, and produces a Red-Green-Refactor report.

## How it works

```
Planner → Generator → Executor → Evaluator → (retry loop) → Synthesizer → Report
```

1. **Planner** (Claude Haiku) — reads your goal and source code, produces a list of test cases to write
2. **Generator** (Claude Sonnet) — writes JUnit 5 test code for each test case
3. **Executor** — runs `gradle test` + JaCoCo coverage in an isolated temp directory
4. **Evaluator** (Claude Haiku) — checks pass/fail results and decides whether to retry with feedback or proceed
5. **Synthesizer** (Claude Sonnet) — after approval, generates the Red/Green/Refactor report with coverage metrics

Retries up to 2 times if tests fail, passing evaluator feedback back to the generator.

## Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React + Vite + Tailwind (zinc/emerald palette) |
| Worker API | Node.js + Express + BullMQ |
| AI | Anthropic Claude (Haiku + Sonnet) |
| Queue | Upstash Redis |
| Database | Supabase PostgreSQL via Prisma 7 |
| Test execution | Gradle 8 + JaCoCo (runs directly, no Docker at runtime) |
| Auth | Custom JWT (bcrypt + jsonwebtoken) |
| Deploy | Vercel (frontend) + Render Docker (worker) |

## Local development

### Prerequisites

- Node.js 22+
- Gradle 8.14+ on PATH (with JDK 17 or 21 via `JAVA_HOME`)
- Upstash Redis account (free tier)
- Supabase project
- Anthropic API key

### Setup

```bash
git clone https://github.com/jyarijarla/TestPilot.git
cd TestPilot/testpilot
npm install
```

Create `apps/worker/.env`:

```env
DATABASE_URL="postgresql://..."
DIRECT_URL="postgresql://..."
ANTHROPIC_API_KEY=sk-ant-...
REDIS_URL=rediss://default:...@....upstash.io:6379
JWT_SECRET=your-secret-here
PORT=3001
```

### Run

Terminal 1 — Worker:
```bash
npm run dev:worker
```

Terminal 2 — Web:
```bash
npm run dev:web
```

Open `http://localhost:5173`

## Usage

1. Sign up / sign in
2. Click **New Run** on the dashboard
3. Paste your Java source code and describe your testing goal
4. Watch the pipeline execute live — each agent's step appears as it completes
5. At the **Review** checkpoint, inspect the test plan and generated test file
6. Click **Approve & Generate Report** to produce the TDD report
7. View pass rate, line coverage, Red/Green/Refactor breakdown, and refactor suggestions

## Example

**Source code:**
```java
package com.example;

public class BankAccount {
    private double balance;

    public BankAccount(double initialBalance) { this.balance = initialBalance; }

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Deposit must be positive");
        this.balance += amount;
    }

    public void withdraw(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Withdrawal must be positive");
        if (amount > this.balance) throw new IllegalStateException("Insufficient funds");
        this.balance -= amount;
    }

    public double getBalance() { return this.balance; }
}
```

**Goal:** Write tests that verify a BankAccount class correctly handles deposits, withdrawals, and throws an exception when overdrawing

## Deployment

### Worker (Render)

- Language: **Docker**
- Root Directory: `testpilot`
- Dockerfile Path: `apps/worker/Dockerfile`
- Instance Type: Starter ($7/mo minimum — needs memory for Gradle)

Environment variables: same as `.env` above, with a strong `JWT_SECRET`.

### Frontend (Vercel)

- Root Directory: `testpilot`
- Build Command: `npm run build --workspace=packages/shared && npm run build --workspace=apps/web`
- Output Directory: `apps/web/dist`

Update `apps/web/vercel.json` with your Render worker URL:

```json
{
  "rewrites": [
    { "source": "/api/:path*", "destination": "https://your-worker.onrender.com/api/:path*" },
    { "source": "/(.*)", "destination": "/index.html" }
  ]
}
```

## Project structure

```
testpilot/
├── apps/
│   ├── web/          # React frontend
│   └── worker/       # Express API + BullMQ worker
│       └── src/
│           ├── agents/       # planner, generator, evaluator, synthesizer
│           ├── tools/        # gradle executor, coverage parser
│           ├── middleware/   # JWT auth
│           ├── app.ts        # Express routes
│           └── index.ts      # BullMQ worker + server entry
├── packages/
│   └── shared/       # Shared types and schemas
└── prisma/
    └── schema.prisma # User, Run, Step, Report models
```
