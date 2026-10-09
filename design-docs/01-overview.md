# CodeAlong — Project Overview

> IS216 Web Application Development II · Section G2 · Team T4
> Andric Ang · Yew Ying Qi · Dylan Shuck · Evan Sim · Karin Tan · Tan Ray Yien

## What Are We Building?

**CodeAlong** is a live-coding classroom tool for courses like IS216 — "Wooclap for live coding". The prof's starter code becomes a step-by-step lesson plan, students code along in the browser, and the prof's dashboard shows who is keeping up, who is stuck and why. Stuck students load the latest checkpoint or ask a TA for help, and after class everyone gets a step-by-step replay.

The core loop is:

```
Prof uploads starter code → TODOs become steps → Prof saves a checkpoint per step
      → Students code along + tap Done → Dashboard shows the room
      → Stuck students catch up / ask the TA → After class: replay + insights
```

**One-sentence pitch** (the format the Q&A slides ask for):

> CodeAlong helps **IS216 profs and TAs** to **see who is falling behind during live coding, and get them back on track**, because **every step is a checkpoint the class can sync to, and every student's progress (not their code) shows up live on one dashboard**.

### The Problem (summary — full version in the proposal)

| Pain point | What happens today |
|---|---|
| Falling behind is silent | A typo at step 2 breaks a student's code. While they debug, the prof moves on and they stay lost for the rest of the lesson. |
| The prof can't see the room | With 40+ laptops, the prof can't tell whether 5 or 25 people are stuck, or whether they're stuck on the same thing. |
| The final file hides the steps | The file sent after class shows finished code, not how it was built — which is what a lost student needs. |

Why existing tools aren't enough: chat tools (Slack/Telegram) move messages but show no progress; VS Code Live Share lets students watch but gives the prof no class view; ChatGPT fixes one student's error but never tells the prof that 13 students made the same one.

### User Roles

| Role | Can do |
|---|---|
| **Student** | Enrol in courses, join a live session by code/QR, code along in the browser workspace, tap Done, tap "I'm lost" (anonymous), load the latest checkpoint, send code to the TA ("Check my work" / "Ask for help"), view replays, write notes/bookmarks, export a replay to GitHub |
| **TA** | Everything a student can do in courses they're staff of **+** view the class dashboard, work the help queue (claim, pass, reply), get Telegram pings for in-person help, receive after-class digests |
| **Prof** | Everything a TA can do **+** create/edit/delete courses and lessons, import starter code (upload or GitHub), edit the lesson plan, start/end live sessions, save checkpoints, Spotlight a common error, view insights reports, send Telegram digests |

> Course-level access is decided by **membership** (staff or enrolled student of that course), not only by account type. A TA account can only see dashboards for courses where they are listed as staff. See [03-data-model.md](./03-data-model.md#authorization-matrix).

---

## Mandatory Requirements Checklist

Every item below comes from the IS216 Group Project Briefing and the Project Q&A slides. The right-hand column shows where each one is handled.

| # | Requirement (source) | How We Address It |
|---|---|---|
| 1 | Identify and solve a problem (Briefing) | Live-coding drop-off in IS216 labs — see problem table above and the proposal |
| 2 | Use HTML, CSS and JavaScript (Briefing) | Vue 3 single-file components (HTML templates + JS), Bootstrap 5 + our own CSS tokens — see [02-tech-stack.md](./02-tech-stack.md) |
| 3 | *Suggested:* CSS framework + JS framework, JSON (Briefing) | Bootstrap 5, Vue 3 + Vue Router + Pinia, JSON REST API |
| 4 | Responsive design following IDP UI patterns (Briefing) | Every page has a defined layout from iPhone 6 (375 px) to Bootstrap XL (≥ 1200 px) — see [04-pages-routes-and-events.md](./04-pages-routes-and-events.md#client-pages) |
| 5 | Dynamic site with a backend data store (Briefing) | Express REST API + Socket.IO + MongoDB Atlas (Mongoose) — see [03-data-model.md](./03-data-model.md) |
| 6 | Meaningful use of ≥ 1 external public API via async HTTP (Briefing) | **Telegram Bot API** (TA pings, digests) and **GitHub REST API** (starter-code import, replay export, one commit per step). LLM API is Keep-In-View — see [05-core-logic.md](./05-core-logic.md) |
| 7 | At least end-to-end testing of core features (Briefing) | Playwright E2E with multi-browser-context tests (prof + students in one test), plus Vitest unit tests for core logic — see [06-stages-and-tests.md](./06-stages-and-tests.md) |
| 8 | Displays properly in Chrome; responsive checked in Chrome DevTools from iPhone 6 to Bootstrap XL (FAQ 5, 17) | Responsive pass is a stage gate; Playwright runs a 375 px and a 1280 px project |
| 9 | App packaged at root; first page via "index" (Briefing) | Express serves the built Vue app (`client/dist/index.html`) at `/` — `http://localhost:3000` loads the landing page |
| 10 | README: URL / repo link, setup, run, test steps, credentials (Briefing) | README skeleton in [07-seed-and-demo.md](./07-seed-and-demo.md#readme-skeleton) |
| 11 | Deployed app link; keep it running until Week 18 (Briefing) | Single Node service + Atlas — see [02-tech-stack.md](./02-tech-stack.md#deployment) |
| 12 | Slides (~10, PPTX/PDF, video link on slide 1) + YouTube video ≤ 12 min, ≥ 720p, unlisted/public (Briefing) | Slide outline + demo script in [07-seed-and-demo.md](./07-seed-and-demo.md) |
| 13 | Free public libraries must be credited in the presentation and code comments (FAQ 6) | Third-party credit table in [02-tech-stack.md](./02-tech-stack.md#third-party-libraries-credit-these) |
| 14 | AI only for lower-order tasks; declare AI-reliant parts in README (FAQ 20) | See [AI Usage Policy](#ai-usage-policy-faq-20) below |
| 15 | API free tiers must survive ~5–6 tests per feature per day (FAQ 19) | Telegram has no cost; GitHub uses an authenticated token (5,000 req/h); LLM responses are cached — see [05-core-logic.md](./05-core-logic.md#external-api-budget) |
| 16 | Don't demo login/registration; demo at 575 px in DevTools with console open; show GitHub page (Q&A slide 11) | Built into the demo script in [07-seed-and-demo.md](./07-seed-and-demo.md) |
| 17 | Login/registration is plumbing, not a feature (Q&A slide 5) | Accounts are listed under "platform", never counted in our feature list |

---

## Grading Breakdown (30 marks total)

Understanding the rubric helps us prioritise work.

### Progress Pitch — 5 marks (Week 9)

| Component | % | Marks | What we show |
|---|---:|---:|---|
| Clearly defined problems & features | 25% | 1.25 | Problem table + tiered feature list (this doc) |
| Clearly defined web design & tech stack | 25% | 1.25 | Prototype screens + [02-tech-stack.md](./02-tech-stack.md) |
| Clearly defined job scopes / task delegation | 25% | 1.25 | [08-workload-split.md](./08-workload-split.md) |
| A functioning task (backend service / API call / CRUD) | 25% | 1.25 | Live demo: prof starts session → students join → checkpoint broadcast → a typo turns a tile red → student loads checkpoint → tile turns green |

A+ needs a **challenging** task working well. A real-time multi-laptop demo qualifies.

### Final Presentation and Deliverables — 20 marks

| Component | % | Marks | What gets top marks |
|---|---:|---:|---|
| **Presentation** | **25%** | **5** | |
| The presentation | 20% | 4 | Crystal-clear, seamless demo; no clarification questions needed; at least one bonus point |
| Q&A | 5% | 1 | Every member can explain their own code and the overall design |
| **Working web application** | **75%** | **15** | |
| Problem solving & solution appropriateness | 20% | 4 | A highly challenging problem solved with an appropriate solution (business side, not code complexity) |
| Working app, correctly implemented, usable | 27% | 5.4 | No bugs; intuitive per the 10 UI design rules of thumb |
| Styling (including responsive design) | 18% | 3.6 | Beautiful and fully responsive across sizes and orientations |
| Testing | 10% | 2 | Tests cover main user journeys, are repeatable, use stable selectors, clear structure, clear README instructions |

### Peer Evaluation — 5 marks

Teamwork and project management are graded implicitly. Raise any conflict with the teaching team **at least 3 weeks before the deadline**.

---

## Features Mapped to the Grading Tiers

The Q&A slides grade on three tiers: **Tier 1** get data → show data (~B+), **Tier 2** get → merge/massage → show (~A-), **Tier 3** logic/algorithm X-factor (A/A+). A feature is something a user would describe as "this app lets me ___" end to end; sub-tasks don't count.

| # | Feature ("this app lets me…") | Tier | Why it's that tier | Owner |
|---|---|:---:|---|---|
| F1 | **Run a live coding session** — start a session, students join by code/QR, every checkpoint reaches every laptop instantly | 2 | Real-time fan-out, reconnection, room authorization; merges session + checkpoint + membership data | Ying Qi |
| F2 | **Code along and catch up** — edit files in the browser, see the live preview and console, have errors caught automatically, tap Done, load the prof's checkpoint without losing my attempt | 2–3 | Sandboxed execution + error capture + snapshot preservation; automatic step checks are the stretch | Dylan |
| F3 | **Get help quickly** — send my code to the TA ("Check my work" / "Ask for help"); TA reviews side-by-side with the checkpoint, passes or replies, or gets pinged on Telegram to come over | 2 | Queue workflow + side-by-side comparison + Telegram Bot API | Karin |
| F4 | **See the room and teach to it** — turn starter-code TODOs into a lesson plan, watch one tile per student, see grouped common errors, Spotlight one anonymised example | **3** | TODO parsing, status classification, error normalisation + grouping algorithm | Evan |
| F5 | **Review the lesson afterwards** — step-by-step replay with "What you missed" diffs, notes, bookmarks, search, import starter code from GitHub and export my replay with one commit per step | 2 | GitHub REST (Git Data API) + diffing + search across lessons | Ray Yien |
| F6 | **Learn which steps lose students** — after-class insights (drop-off per step, top errors, recurring trouble steps across terms) sent as a Telegram digest | 2–3 | Aggregation across statuses, help requests and past terms | Andric |

**Platform (not features):** accounts, login/registration, roles, courses, enrolment, deployment, shared UI kit — owned by Andric.

**Our signature (Tier 3):** F4's error grouping + status classifier and F2's catch-up flow. These are what we lead with in the pitch and the video.

---

## Scope Target

Priorities follow the latest proposal (self-marking + automatic error detection + TA checks are the baseline; automatic step checks are attempted on top).

| Level | What's included |
|---|---|
| **Week 9 pitch (must work live)** | Sessions + join by code, checkpoint broadcast, student editor + preview + console, basic dashboard tiles, load checkpoint |
| **MUST (baseline for submission)** | Accounts/courses/roles; lesson CRUD + plan from TODOs; live checkpoints; student workspace with error detection + Done; class dashboard with filters + anonymous "I'm lost" count; catch-up with "What you missed" diff; Check my work / Ask for help + TA queue + Telegram ping; replay with notes/bookmarks |
| **SHOULD** | Error groups + Spotlight; GitHub import + export; search across lessons; insights report + Telegram digest |
| **COULD** | Automatic step checks generated from checkpoints; AI one-line hints (hints only, cached, rule-based fallback); VS Code one-line reporter; `.vue` single-file-component lessons |

**Rule:** no COULD work starts until every MUST item passes the full end-to-end demo in [07-seed-and-demo.md](./07-seed-and-demo.md).

**Out of scope:** lessons that need an Express server, `npm install` or Vite (API calls in lessons go to CodeAlong's mock API instead); recording keystrokes; showing any student's code to staff unless the student sends it.

---

## Coding Patterns We Will Follow

| Area | Pattern |
|---|---|
| Module system | ES modules everywhere (`"type": "module"`), so `shared/constants.js` is imported by both client and server |
| Vue style | Composition API with `<script setup>`; props down / events up; `computed` for derived state, `watch` for side effects |
| Client state | One Pinia store per domain (`auth`, `courses`, `session`, `workspace`, `dashboard`, `help`, `replay`) |
| Client I/O | Components never call `axios`/`socket.io` directly — only through `client/src/api/*` modules |
| Server layering | `routes` → `middleware` → `controllers` → `services` / `lib` → `models` |
| Async style | `async/await` + `try/catch`; unexpected errors go to `next(err)` |
| Validation | Request bodies validated with `zod` schemas in `server/validators/`; Mongoose enforces types/enums as the second layer |
| Responses | Always `{ data }` on success, `{ error: { code, message, fields? } }` on failure — see [09-integration-contracts.md](./09-integration-contracts.md) |
| Realtime | REST for anything that writes to the DB; Socket.IO only for broadcasting and lightweight status |
| Testability | Every interactive element has a `data-testid` following the convention in [09-integration-contracts.md](./09-integration-contracts.md#9-data-testid-convention) |
| Credits | Any third-party snippet gets a `// Source: <url> (licence)` comment and a line in the slides |

---

## Design Decisions Summary

| Decision | Why |
|---|---|
| Student code runs in a sandboxed `<iframe>` in **their own browser** | Zero server cost per student, no remote code execution risk on our server, broken code can't affect anything else |
| Dashboard shows **status, not code** | Privacy by design (our unique angle); also keeps socket payloads tiny |
| "I'm lost" is stored per student but only ever **sent out as a count** | Dedup (one student can't spam it) without exposing identity |
| One `StudentStatus` document per student per session (unique index) | The dashboard needs the *latest* state only — an upsert per change, not an event log |
| Snapshots stored separately from status, with a `reason` | Keeps attempts after "Load checkpoint"; only snapshots attached to a help request are ever visible to staff |
| Steps embedded in `Lesson` | Reorder = replace one array atomically; steps are never queried on their own |
| REST writes + Socket.IO broadcasts | Writes get validation, auth and HTTP status codes; sockets stay a thin, replaceable layer |
| Express serves the built Vue app (single origin) | Satisfies "index at root", lets the httpOnly JWT cookie work for both REST and the Socket.IO handshake |
| Self-marking + auto error detection as baseline; auto step checks as stretch | Automatic checking is the riskiest part — the app is complete without it |
| In-memory MongoDB + mocked external APIs in E2E | Tests are repeatable and never burn API quota |
| Bootstrap 5 + our own design tokens | Fast responsive grid; tokens keep the look consistent across six people's pages |

---

## AI Usage Policy (FAQ 20)

| Allowed (lower-order) | Not allowed (higher-order) |
|---|---|
| Information search, explaining errors, debugging hints | Core business logic (status classifier, error grouping, TODO parser, catch-up, help workflow, export) |
| Layout/theme ideas, UI/UX inspiration | Backend endpoints and Socket.IO handlers |
| Boilerplate and starter snippets | Critical frontend interactivity (workspace, dashboard, replay) |
| Generating unit test cases, sample inputs, mock/seed data | Solving major implementation issues end-to-end |

- Each member keeps a short log of any AI-assisted part they rely on; it goes into the README's "Use of AI" section.
- These design docs and the proposal's throwaway prototypes were drafted with AI help (planning/boilerplate); the prototypes are not submitted.
- In Q&A, every member must be able to explain their own code without notes.
