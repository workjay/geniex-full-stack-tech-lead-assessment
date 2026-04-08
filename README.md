# Tech Lead Assessment: WorkLog Settlement System

## Overview

You have been handed a **fullstack WorkLog settlement system** built by a junior developer. The system is composed of **10 files** in the `candidate/` directory — a TypeScript backend (Express + PostgreSQL) and a React frontend.

The system's goal is to:

- Track freelancer work via **WorkLogs** containing **TimeSegments** (independently recorded periods of work, each with hours and a rate)
- Apply **Adjustments** (deductions or bonuses) to WorkLogs
- Run a **settlement** that calculates how much each freelancer is owed and creates a **Remittance** (a single payout per user)
- Provide an admin-facing **Settlement Review** screen where an admin can inspect open worklogs and their amounts before confirming a settlement
- Handle **post-settlement changes**: new segments can be added to previously settled WorkLogs, reopening them for the next settlement cycle

Your task is to answer the questions stated below in the deliverables section.

**You do not need to run the code to complete this assessment.** All questions can be answered by reading the provided files and tracing the logic. The seed data in `seed.ts` shows you exactly what is in the database.

---

## Files

### Backend

| File | Purpose |
|------|---------|
| `candidate/types.ts` | Type definitions for all domain entities |
| `candidate/config.ts` | Application configuration |
| `candidate/db.ts` | Database query functions (PostgreSQL) |
| `candidate/calculations.ts` | Amount calculation logic and a `filterNewSegments` utility |
| `candidate/settlement.ts` | Settlement engine — core business logic |
| `candidate/api.ts` | Express route handlers |
| `candidate/seed.ts` | Seed data — **read the summary at the bottom of this file** |

### Frontend

| File | Purpose |
|------|---------|
| `candidate/api-client.ts` | API client functions used by the frontend |
| `candidate/use-settlement.ts` | React hook — fetches worklogs, manages settlement state, polls for updates |
| `candidate/SettlementReview.tsx` | React component — the admin-facing settlement review and confirmation screen |

---

## Deliverables

Create a file named **`solution.md`** and provide your responses to the 5 questions below.

- [Q1: Design Choices — The Settlement Contract](Q1-architecture.md)
- [Q2: Trace the State](Q2-trace-the-state.md)
- [Q3: Predict the Failure Mode](Q3-predict-the-failure.md)
- [Q4: What Happens With THIS Input?](Q4-trace-the-input.md)
- [Q5: Fix Evaluation](Q5-evaluate-the-fix.md)

**Expected Time: 60 Minutes**

---

## Submission Process

To submit your assessment, please follow these steps:

1. **Fork this repository**

   Create a fork of this repository to your personal GitHub account. Add a new `solution.md` file with your answers to all 5 questions.

2. **Raise a Pull Request**

   Create a Pull Request from your personal fork back to this repository.

3. **MANDATORY: Include your name in the PR**

   Add your Upwork name, Upwork profile link, and the role you applied for to the PR description so we can identify you.
