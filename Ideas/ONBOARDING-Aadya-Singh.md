# Contributor Onboarding — Aadya Singh

**Project:** Tenali — Adaptive Mathematics & Logic Learning Platform  
**Author:** Aadya Singh | **Branch:** `refactor/math-helpers` | **Date:** November 2026  

---

## 1. What is Tenali?

Tenali is a web-based adaptive mathematics learning platform designed for young learners, primarily supporting the Taittiriya and Tatsavit student cohorts. Instead of relying on static problem sheets or fixed question databases, Tenali generates practice questions algorithmically on the server at request time. This design ensures that practice is virtually infinite and never repeats, allowing learners to build fluency across arithmetic, algebra, geometry, calculus, trigonometry, probability, and foundational logic.

The system incorporates dynamic difficulty adaptation, adjusting the cognitive load in real time based on each student's ongoing accuracy and speed. To keep young students motivated, Tenali integrates pedagogical game mechanics including experience points (XP), collectable badges, story-based mystery investigations in the Math Detective Agency, and real-time multiplayer speed duels in the Battle Arena.

---

## 2. System Flow Walked Through the Code

### 2.1 One Question End-to-End: Linear Algebra Quiz
To understand the runtime lifecycle of a question, we can trace a Linear Algebra problem from initial client dispatch to permanent storage:

1. **Client Request:** When a learner begins a round, the `loadQuestion()` function in [`client/src/App.jsx#L43718-L43734`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/App.jsx#L43718-L43734) determines the active difficulty tier from `effectiveDifficulty()` and dispatches an HTTP GET request to `${API}/linearalgebra-api/question?difficulty=${diff}`.
2. **Server Route Dispatch:** The Express server mounts the advanced router at [`server/index.js#L657`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L657). In [`server/routes/advanced.js#L698-L704`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/routes/advanced.js#L698-L704), `router.get('/question')` extracts the topic name from the base URL and delegates execution to `generators.linearalgebra.question(req.query.difficulty)`.
3. **Algorithmic Generation:** The generator in [`server/routes/advanced.js#L165-L240`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/routes/advanced.js#L165-L240) selects an appropriate problem template for the chosen band, calculates operands using helper utilities, and returns a JSON payload containing the prompt, canonical answer, display string, and metadata.
4. **Adaptive Scoring Adjustment:** When the student submits an answer, [`client/src/App.jsx#L43769-L43771`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/App.jsx#L43769-L43771) updates `adaptScoreRef.current` by adding 0.25 points for correct answers or subtracting 0.35 points for errors within a bounded range of 0.0 to 3.0. In [`client/src/App.jsx#L54702`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/App.jsx#L54702), the helper `adaptiveLevel(score)` maps this continuous score to the discrete difficulty bands (`easy`, `medium`, `hard`, or `extrahard`) for subsequent questions.
5. **Answer Validation:** The client posts the user's input to `/linearalgebra-api/check` at [`client/src/App.jsx#L43763`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/App.jsx#L43763). In [`server/routes/advanced.js#L706-L712`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/routes/advanced.js#L706-L712), `router.post('/check')` passes the body to `generators.linearalgebra.check(body)` ([lines 241–273](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/routes/advanced.js#L241-L273)) to verify mathematical equality.
6. **Solve Middleware & Explanation:** If the client includes `{ solve: true }` in the request body (which occurs when the student clicks "Solve" or automatically upon a mistake at [`client/src/App.jsx#L43774`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/App.jsx#L43774)), the solve middleware in [`server/index.js#L361-L392`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L361-L392) intercepts `res.json`. It marks `data.solved = true` and invokes `generateExplanation(req, data)` from [`server/explanations.js#L12-L30`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/explanations.js#L12-L30) to attach a step-by-step derivation.
7. **Progress Persistence:** Once the question state is revealed, the `useProgressSubmit` hook in [`client/src/App.jsx#L52-L72`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/App.jsx#L52-L72) sends a POST request with the topic, correctness, and question ID to `/api/progress/update`. In [`server/progress.js#L22-L89`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/progress.js#L22-L89), MongoDB updates the user's progress atomically. Concurrently, the LIL interceptor in [`server/index.js#L408-L460`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L408-L460) updates the student's Bayesian Knowledge Tracing parameters asynchronously.

### 2.2 Additional System Traces
- **Authentication & Login (JWT):** The user submits credentials to `POST /api/auth/login`. In [`server/auth.js#L343-L370`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/auth.js#L343-L370), the server queries the database for the user record and verifies the password hash using `bcrypt.compare`. Upon success, `signToken(user)` in [`server/auth.js#L313-L319`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/auth.js#L313-L319) generates a signed JSON Web Token containing the user's ID, username, and role. Subsequent requests include this token in the `Authorization: Bearer` header, which the `requireAuth` middleware in [`server/auth.js#L321-L332`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/auth.js#L321-L332) validates via `jwt.verify`.
- **Battle Arena Round (Socket.IO):** A host creates a room by emitting `createRoom`, prompting the server in [`server/index.js#L2931-L2949`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L2931-L2949) to generate a four-letter room code in its active `rooms` map. When a second player emits `joinRoom` ([lines 2951–2965](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L2951-L2965)) and both players signal `ready` ([lines 2967–2986](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L2967-L2986)), the server starts the match. The function `startRound(room)` in [`server/index.js#L2811-L2827`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L2811-L2827) emits `roundStart` with a synchronized fifteen-second countdown. As players submit answers via `submitAnswer` ([lines 2988–3000](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L2988-L3000)), `endRound(room)` in [`server/index.js#L2829-L2893`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L2829-L2893) tallies base points and speed bonuses until `endMatch` concludes the duel.
- **Code Playground Request (Judge0 Proxy):** In [`client/src/PlaygroundApp.jsx#L882-L913`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/PlaygroundApp.jsx#L882-L913), the `handleRun` callback packages code and language identifiers, posting them to `/api/playground/run`. The Express route in [`server/index.js#L1439-L1472`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L1439-L1472) receives this payload and issues an upstream HTTPS request to the public Judge0 CE API at `https://ce.judge0.com/submissions?base64_encoded=false&wait=true`, returning compiler output and runtime metrics directly back to the client interface.

---

## 3. Implemented Features Overview

| Feature | Primary Files | How It Works (One-Line Summary) |
| :--- | :--- | :--- |
| **Puzzle Library** | [`client/src/App.jsx`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/App.jsx); [`server/routes/*.js`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/routes) | Procedurally generates algorithmic math problems across 88+ endpoints rendered in modular React quiz views. |
| **Adaptive Difficulty** | [`client/src/App.jsx#L41892`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/App.jsx#L41892); [`server/lib/bkt.js`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/lib/bkt.js); [`server/lil/masteryEngine.js`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/lil/masteryEngine.js) | Shifts difficulty bands based on real-time client score deltas combined with backend Bayesian Knowledge Tracing ($P(L_t)$). |
| **Battle Arena** | [`client/src/BattleApp.jsx`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/BattleApp.jsx); [`server/index.js#L2149-L2365`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L2149-L2365), [`L2811-L3065`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L2811-L3065) | Real-time 1v1 speed duel system over WebSockets where two players race to solve identical questions under 15s timers. |
| **Step-by-Step Explanations** | [`client/src/App.jsx#L43772`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/App.jsx#L43772); [`server/explanations.js`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/explanations.js); [`server/index.js#L361`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L361) | Intercepts `/check` requests bearing `{ solve: true }` and injects dynamically formatted derivation steps into response JSON. |
| **Code Playground** | [`client/src/PlaygroundApp.jsx`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/PlaygroundApp.jsx); [`server/index.js#L1437-L1501`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L1437-L1501); [`server/compiler.js`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/compiler.js) | Dual-execution environment running code in 50+ languages via external Judge0 CE proxy or local child processes. |

---

## 4. Verified Codebase Gaps & Vulnerabilities

### 1. `server/auth.js`: Fallback to Development Secret When Variable is Unset
- **File & Lines:** [`server/auth.js#L27-L40`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/auth.js#L27-L40) | **Status:** REAL GAP
```javascript
const DEFAULT_DEV_SECRET = 'tenali-dev-secret-change-me';
const JWT_SECRET = process.env.JWT_SECRET || DEFAULT_DEV_SECRET;
if (process.env.NODE_ENV === 'production' &&
  (!process.env.JWT_SECRET || process.env.JWT_SECRET === DEFAULT_DEV_SECRET)) {
  throw new Error('JWT_SECRET must be set to a strong, non-default value in production — refusing to start.');
}
```
In standard Node.js environments, `process.env.NODE_ENV` defaults to `undefined`. If a deployment accidentally omits setting `NODE_ENV=production`, the startup check evaluates to false and the server silently boots using the public development secret. Because this key is checked into version control in plaintext, an attacker can easily forge arbitrary tokens with administrator privileges.

### 2. `server/index.js`: Solve Middleware Monkey-Patches `res.json`
- **File & Lines:** [`server/index.js#L373-L389`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L373-L389) | **Status:** REAL GAP
```javascript
  const originalJson = res.json.bind(res);
  res.json = function (data) {
    data.solved = true;
    const explanation = generateExplanation(req, data);
    if (explanation) data.explanation = explanation;
    return originalJson(data);
  };
```
The middleware directly mutates the response payload by writing `data.solved = true`. If any route handler returns an error string or null instead of an object, JavaScript throws an unhandled `TypeError` that terminates the request unexpectedly. Furthermore, stacking three separate monkey patches across different middlewares creates fragile execution ordering and risks dropping intermediate response modifications.

### 3. `server/index.js` & `server/routes/financial.js`: Floating-Point Inaccuracies
- **File & Lines:** [`server/routes/financial.js#L330-L335`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/routes/financial.js#L330-L335) (historical line ~6201 in the monolith) and [`server/index.js#L1647`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L1647) | **Status:** REAL GAP
```javascript
const MV = P * n + (P * n * (n + 1) / (2 * 12)) * (r / 100);
return { id, difficulty: diff, prompt: `RD maturity: Monthly Rs ${P}, Rate ${r}%, Months ${n}`, answer: P, display: String(Math.round(P)) };
```
In the recurring deposit generator, the question displays a rounded integer string while storing the raw unrounded float in the answer field, causing discrepancies during student validation. In addition, the generic answer comparator relies on a fixed tolerance of `0.01`, which incorrectly accepts wrong answers for small fractional quantities such as probabilities and limits.

### 4. `server/lib/mathHelpers.js`: Zero Input Handling and Absence of Unit Tests
- **File & Lines:** [`server/lib/mathHelpers.js#L50-L66`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/lib/mathHelpers.js#L50-L66) | **Status:** REAL GAP
```javascript
function lcm(a, b) { return Math.abs(a * b) / gcd(a, b); }
function simplifyFraction(num, den) {
  if (den < 0) { num = -num; den = -den; }
  const g = gcd(Math.abs(num), den);
  return { num: num / g, den: den / g };
}
```
When invoked with zeros, `lcm(0, 0)` attempts to divide zero by zero and returns `NaN`. Similarly, `simplifyFraction(5, 0)` calculates a greatest common divisor of five and produces a fraction with a zero denominator instead of raising a descriptive error. A codebase search confirms that this utility file has zero automated unit test coverage across the repository.

### 5. `client/src/ContrastChallengeApp.jsx`: Hardcoded Concept Pairs
- **File & Lines:** [`client/src/ContrastChallengeApp.jsx#L10-L27`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/ContrastChallengeApp.jsx#L10-L27) | **Status:** REAL GAP
```javascript
// Hardcoded list of all contrast pairs (replaces server JSON + API)
const CONTRAST_PAIRS = [
  { id: 'area-perimeter', title: 'Area vs Perimeter' },
  { id: 'congruence-similarity', title: 'Congruence vs Similarity' }, ...
];
```
The client replaces dynamic server discovery with a static fifteen-item array embedded directly inside a 14,202-line file. If new challenge modules or topic mappings are added on the backend, the client cannot discover them without manual code changes and bundle redeployments, causing frontend and backend schemas to diverge.

### 6. `client/src/monsters/CureFlow.jsx`: Fallback Questions Defaulting to Bracketeer
- **File & Lines:** [`client/src/monsters/CureFlow.jsx#L87-L91`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/monsters/CureFlow.jsx#L87-L91) | **Status:** REAL GAP
```javascript
function getHardcodedFallbackQuestion(monsterId, index) {
  const list = HARDCODED_FALLBACKS[monsterId] || HARDCODED_FALLBACKS['bracketeer'];
  const q = list[index % list.length];
  return { id: `fallback-${monsterId}-${index}`, prompt: q.prompt, correctAnswer: q.correctAnswer };
}
```
When a student attempts to cure a misconception monster and has fewer than five recorded mistakes, the flow fills remaining slots with hardcoded questions. If the active monster is not one of four specific IDs, the logic falls back to `bracketeer`. Consequently, a student practicing geometry or calculus receives irrelevant algebraic bracket expansion problems.

### 7. `client/src/App.jsx`: State-Based Navigation Without a Dedicated Router
- **File & Lines:** [`client/src/App.jsx#L42448-L42455`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/App.jsx#L42448-L42455) and [`client/src/App.jsx#L42566`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/App.jsx#L42566) | **Status:** REAL GAP
```javascript
const [mode, setMode] = useState(() => {
  const params = new URLSearchParams(window.location.search);
  return params.get('mode') || null;
});
if (mode && currentMode !== mode) window.history.replaceState({}, '', `${BASE}/?mode=${mode}`);
```
The application does not include a declarative routing library, opting instead to manage global navigation using a React `mode` state variable. Because URL synchronization relies on `replaceState` rather than pushing new history entries, the browser's Back button exits the web application entirely rather than returning to the previous quiz view, preventing deep linking into nested lab states.

### 8. `client/src/proctor/`: Unauthenticated, Client-Reported Proctoring Events
- **File & Lines:** [`client/src/proctor/proctorEvents.js#L48`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/client/src/proctor/proctorEvents.js#L48) and [`server/index.js#L1250-L1274`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L1250-L1274) | **Status:** REAL GAP
```javascript
app.post('/api/proctor/event', async (req, res) => {
  const { sessionId, type, severity, evidence } = req.body;
  if (!sessionId || !type) return res.status(400).json({ error: 'sessionId and type required' });
  const event = await ProctorEvent.create({ sessionId, type, severity: severity || 1, ... });
  await ProctorSession.findByIdAndUpdate(sessionId, { $inc: { totalPenalty: severity || 1 } });
```
All anomaly detection algorithms run exclusively on the client machine, and the receiving backend endpoints omit authentication checks. A student taking an exam can bypass monitoring entirely by filtering or blocking outgoing network requests to `/api/proctor/event`. Furthermore, session overview endpoints remain completely open to public viewing.

### 9. Input Validation Deficiencies in `/check` Endpoints
- **File & Lines:** [`server/routes/darts.js#L53-L57`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/routes/darts.js#L53-L57) and [`server/index.js#L964-L979`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L964-L979) | **Status:** REAL GAP
```javascript
router.post('/check', express.json(), (req, res) => {
  const { userX, userY, x, y } = req.body;
  const correct = userX === x && userY === y;
  res.json({ correct, message: correct ? 'Bullseye!' : 'Missed!' });
});
```
In `darts.js`, sending an empty JSON object causes all four destructured variables to evaluate to `undefined`. Because the strict equality comparison `undefined === undefined` evaluates to true, an empty payload is graded as a correct bullseye. In the quadratic evaluation endpoint, the client supplies both the formula parameters and answer without server verification.

### 10. Audit of Files with Missing Automated Test Coverage
- **Server Libraries:** The utilities in `server/lib/`—including `mathHelpers.js`, `conceptSkills.js`, `dailyWarmup.js`, `displayedMastery.js`, `logger.js`, `question-banks.js`, `spacingLadder.js`, and `sudoku.js`—have no dedicated unit test files.
- **Router Modules:** Router files including `contrast.js`, `darts.js`, `la-mission-quiz.js`, `matrixmystics.js`, `review.js`, `riddle.js`, `sudoku.js`, `transfer.js`, `visual-math.js`, and `wordCreator.js` lack contract test coverage.
- **Core Subsystems:** Critical application services like `server/compiler.js`, `server/progress.js`, `server/proctorSchema.js`, and the student tracking logic in `server/lil/` operate without automated verification suites.

### 11. Additional Gap: Unauthenticated Remote Code Execution in Local Compiler
- **File & Lines:** [`server/index.js#L1489-L1501`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/index.js#L1489-L1501) and [`server/compiler.js#L243-L288`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/compiler.js#L243-L288) | **Status:** REAL GAP
```javascript
app.post('/api/playground2/run', async (req, res) => {
  const { language, code, stdin, timeout } = req.body;
  const result = await compiler.executeCode(language, code, stdin || '', timeout);
  res.json(result);
});
```
The `/api/playground2/run` endpoint is publicly exposed without authentication. It takes arbitrary source code across nine programming languages and executes it directly on the host operating system using Node's `child_process.spawn`. Because the execution environment lacks containerized sandboxing, an unauthorized user can read server files, consume system resources, or spawn reverse shell connections.

---

## 5. Author's Contribution Context (`refactor/math-helpers`)

My contribution on the `refactor/math-helpers` branch addresses technical debt identified in issue #117 by modularizing math utilities across the backend:
- **Centralizing Mathematical Logic:** I extracted duplicate inline helper functions—such as `randomInt`, `gcd`, `lcm`, `simplifyFraction`, and array randomizers—out of the legacy `server/index.js` monolith and individual route files into a single reusable module at [`server/lib/mathHelpers.js`](file:///c:/Users/hp/OneDrive/Desktop/projects/tenali/server/lib/mathHelpers.js).
- **Standardizing Route Handlers:** I refactored over ten separate router files across algebra, arithmetic, geometry, financial math, and calculus to import shared helpers from this common library, eliminating code duplication while ensuring that all 124 API contract tests continue to pass.
