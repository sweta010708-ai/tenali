## 1. What is Tenali?

Tenali is an open-source, interactive, and adaptive learning platform designed to make mathematics practice engaging and effective for young learners. Named after the legendary Indian court wit Tenali Raman, celebrated for his sharp logic and intellect, the platform fosters mathematical reasoning and problem-solving intuition rather than rote memorization.

Traditional mathematics education relies heavily on textbooks and static question banks. These resources quickly run out of problems, lack variety, and fail to adapt to a learner's individual pace—either boring advanced students with repetitive drills or overwhelming struggling students with unmanageable difficulty. Tenali solves this by generating mathematics questions algorithmically on the server at runtime. Instead of pulling from a pre-written database, each problem is synthesized dynamically based on a difficulty parameter, making practice endless, fresh, and non-repeating.

Furthermore, Tenali keeps students engaged by integrating gamification mechanics: experience points (XP), coin rewards, streak tracking, interactive concept sandboxes (like the Balance Scale lab), multiplayer battle modes, and story-driven investigative cases.

---

## 2. What do I understand by Tenali (as a system)?

At an architectural level, Tenali operates as an algorithmic question-generation and validation engine paired with a rich, interactive Single Page Application (SPA) frontend.

### Core Users
1. **Learners:** Primary users who interact with the system either as authenticated users or guests. They choose topics from the curriculum dashboard, practice dynamically generated math problems, explore visual conceptual labs, and review step-by-step explanations.
2. **Proctors / Educators:** Administrators who oversee student activity and exam sessions through dedicated proctor endpoints (`/api/proctor/sessions`).

### Primary Components and Entities
- **Algorithmic Question Endpoints:** The system exposes consistent route pairs across dozens of puzzle types: `GET /<type>-api/question` generates a fresh problem with canonical answers, and `POST /<type>-api/check` verifies the learner's answer.
- **Adaptive Difficulty Engine:** Sessions adjust difficulty based on learner performance (tracking continuous adaptation scores and discrete difficulty bands such as easy, medium, hard, and extrahard).
- **Session Goals:** Quiz sessions support diverse goal modes:
  - *Standard Mode:* Focuses on accurate practice.
  - *Speed Run:* Implements strict per-question countdown timers where running out of time records an automatic timeout.
  - *Perfect Solve:* Tests precision where a single mistake or timeout concludes the run.
- **Visual Labs & Concept Playgrounds:** Specialized interactive modules—such as the Balance Scale lab, Geometry App, and Visual Math Redux—that allow students to visualize equations and geometric relationships physically.
- **Explanation Engine:** Centralized middleware (`server/explanations.js`) generates step-by-step mathematical derivations when a user requests a walkthrough using the "Solve" feature.
- **Multiplayer & Sandboxes:** A real-time Battle Arena powered by Socket.io, along with code sandboxes and multilingual vocabulary tools (Vachana).

### High-Level Data Flow
1. **Question Request:** The client triggers `GET /<type>-api/question?difficulty=N`. The server algorithmically computes the problem parameters and returns the problem structure to the client.
2. **Session Progression & Answering:** The student enters their response. In speed-run mode, a countdown timer ticks in the client. If time runs out, a client-side timeout handler immediately submits an empty answer, displays feedback, and updates the session result log.
3. **Validation & Adaptation:** The client submits answers to `POST /<type>-api/check`. The server validates correctness, computes scores, and updates adaptive metrics.
4. **State Persistence:** User progress, streaks, and XP are stored in MongoDB via Mongoose, with an in-memory fallback mechanism ensuring seamless local development even without a running database instance.

---

## 3. Current State of the Repository — What Has Been Done So Far

Tenali has grown into an extensive full-stack platform covering arithmetic, algebra, coordinate geometry, probability, matrices, and language logic:

### Tech Stack
- **Frontend:** React 19, Vite 8, Lucide icons, Framer Motion, HTML5 Canvas, and Three.js / Mafs for mathematical visualizations.
- **Backend:** Node.js, Express, Socket.io for multiplayer synchronization.
- **Persistence:** MongoDB with Mongoose, coupled with an in-memory fallback for local offline operation.
- **Testing & Tooling:** ESLint 9 (flat config), Vitest, Supertest, and GitHub Actions CI pipelines (BKT unit tests, Client lint, and API contract tests).

### Key Features Implemented
- **70+ Puzzle Families:** From fundamental addition and times tables to quadratic equations, polynomial factorization, limits, and simultaneous equations.
- **Interactive Labs:** The Balance Scale weight bank for solving linear equations, geometry construction tools, and probabilistic simulators.
- **Concept Modules & Vachana:** Dedicated concept learning progression ladders and language exploration suites.
- **Gamified Rewards:** Daily warmup streaks, badges, coins, and level progressions.

---

## 4. Gaps Observed in the Code

During my exploration and work on the codebase, I identified several concrete issues impacting maintainability, stability, and developer workflows:

### Gap 1: Inconsistent and Broken `handleTimeout` Implementations in Quiz Apps
- **Where:** `client/src/App.jsx` across multiple bespoke quiz components, including `SqrtApp`, `PolyMulApp`, `PolyFactorApp`, `PrimeFactorApp`, `QFormulaApp`, `SimulApp`, `FuncEvalApp`, and `LineEqApp`.
- **What:** When timeout handling was added for speed-run and perfect-solve goals, several apps referenced undefined state variables (`phase` and `setPhase('finished')`). However, these components manage their completion using `finished` and `setFinished(true)`. Furthermore, orphaned timeout functions existed in components like `RiyaApp` that lacked timer hooks.
- **Why it matters:** In "Perfect Solve" or "Speed Run" modes, letting the countdown timer expire failed to transition the quiz to the finished summary screen properly or threw runtime exceptions, preventing learners from viewing their performance breakdown.

### Gap 2: Missing Prop Destructuring in `QuizLayout` Causing CI ESLint Failures
- **Where:** `client/src/components/QuizLayout.jsx` (line 8 and line 75).
- **What:** During a mechanical cleanup, the `subtitle` prop was removed from the function parameter list (`export function QuizLayout({ title, onBack, children, ... })`), but line 75 still rendered `{subtitle && <p className="quiz-subtitle">{subtitle}</p>}`.
- **Why it matters:** Because ESLint enforces `no-undef: 'error'`, this caused blocking CI failures (`Exit code 2`) on pull requests across the repository, stalling continuous integration checks.

### Gap 3: Monolithic Structure of `client/src/App.jsx`
- **Where:** `client/src/App.jsx` (~71,000 lines of code).
- **What:** Tens of distinct quiz apps, concepts, and utilities are housed in a single massive file. Many quiz applications duplicate timer initialization, result recording, and submission logic rather than leveraging reusable hooks.
- **Why it matters:** Code reviews become difficult, merge conflicts are frequent when multiple contributors work on different puzzles, and editor performance suffers.

### Gap 4: Balance Scale Weight Bank State Reset Edge Cases
- **Where:** Balance Scale lab implementation.
- **What:** Interactive weight bank elements on the balance pans required more intuitive handling during dynamic re-balancing and state resets between practice rounds.
- **Why it matters:** When students experiment with different weights to balance algebraic equations, inconsistent weight bank resets can cause confusion and break interactive learning flow.

---

## 5. Ideas for the Project

### Idea 1: Standardized `useQuizSession` Custom Hook
- **What:** Extract timer logic, goal modes (`standard`, `speed`, `perfect`), timeout handling, and results recording into a unified custom hook.
- **Why:** Currently, boilerplate for `timer.start(...)`, `handleTimeout`, `results`, and `sessionGoal` is duplicated across dozens of quiz apps. Centralizing this logic ensures consistency and reduces bug surface area.
- **How:** Create `client/src/hooks/useQuizSession.js` that accepts `{ apiPath, sessionGoal, onTimeout, totalQ }` and provides standardized handlers (`submitAnswer`, `handleTimeout`, `results`, `isFinished`, `timer`).

### Idea 2: Component Extraction from `App.jsx`
- **What:** Continue modularizing the remaining inline quiz components from `client/src/App.jsx` into separate files in `client/src/apps/`.
- **Why:** Decreases the monolithic footprint of `App.jsx`, accelerates Vite hot-module replacement (HMR), and prevents merge conflicts between contributors.
- **How:** Move individual quiz components (`SqrtApp`, `PolyMulApp`, `PrimeFactorApp`, etc.) into standalone files, export them, and maintain the central lookup registry (`modeMap`).

### Idea 3: Enhanced Algebraic Balance Scale Visualizer
- **What:** Extend the Balance Scale lab to support two-sided variable terms ($ax + b = cx + d$) with drag-and-drop algebraic balancing.
- **Why:** The balance scale metaphor is one of the most effective ways for young students to develop an intuitive understanding of the equality axiom (performing the same operation on both sides of an equation).
- **How:** Add support for unknown mass blocks (representing $x$) alongside known gram weights, animating simultaneous removals from both pans.

---

## 6. My Contribution

As part of my onboarding, I made verified contributions addressing real defects in the platform's quiz infrastructure:

1. **Fixed Quiz Timers and Timeout Logic (PR #357):**
   - Corrected `handleTimeout` across 8 quiz applications in [`client/src/App.jsx`](client/src/App.jsx): `SqrtApp`, `PolyMulApp`, `PolyFactorApp`, `PrimeFactorApp`, `QFormulaApp`, `SimulApp`, `FuncEvalApp`, and `LineEqApp`.
   - Replaced errant references to undefined `phase` / `setPhase` with proper `finished` and `setFinished(true)` state handling.
   - Cleaned up orphaned, dead timeout declarations in `RiyaApp`.
   - Enhanced the Balance Scale interactive weight bank experience.

2. **Resolved Blocking CI Lint Errors:**
   - Diagnosed and fixed the blocking `no-undef` ESLint error in [`client/src/components/QuizLayout.jsx`](client/src/components/QuizLayout.jsx) by restoring the missing `subtitle` prop.
   - Addressed the mount-only navigation `react-hooks/exhaustive-deps` warning in [`client/src/vachana/index.jsx`](client/src/vachana/index.jsx).
   - Confirmed client builds cleanly (`npm run build` completed in ~8s) and reduced ESLint errors from 2 to 0.
