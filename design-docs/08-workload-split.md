# CodeAlong — Workload Split (6 members)

> Based on the "Who do what?" table in our proposal. The Q&A slides ask for **distinct** responsibilities (no "Alice & Bob are responsible for the database"), roughly one meaningful feature per person, and a visible contribution from everyone. Each card below has a single owner per file. Items marked **(proposed)** weren't in the proposal table — confirm them at the next meeting (see [Open questions](#open-questions)).

## Summary

| Member | Owns | Feature | Tier | Also owns |
|---|---|---|:---:|---|
| **Ying Qi** | Live session engine: rooms, join codes + QR, checkpoint broadcast, reconnection | F1 | 2 | Status classifier + dashboard cache, load simulator **(proposed)** |
| **Dylan** | Student workspace: editor, sandboxed preview, console, error capture, Done, load checkpoint | F2 | 2–3 | **All Mongoose models**; automatic step checks (COULD) |
| **Karin** | Help flow: Check my work / Ask for help, TA queue, side-by-side view, pass ticks, Telegram bot | F3 | 2 | Telegram service used by others; AI hints (COULD) **(proposed)** |
| **Evan** | Prof view: editor, lesson plan from TODOs, class dashboard | F4 | **3** | Error normaliser + grouping, Spotlight **(proposed)** |
| **Ray Yien** | Replay, lesson library, search, "What you missed" diff, GitHub import + export | F5 | 2 | `DiffView` + diff utility used by Dylan and Karin |
| **Andric** | Accounts, courses, roles, insights report, deployment, UI consistency | F6 + platform | 2–3 | Playwright config/fixtures, test-only routes, README assembly **(proposed)** |

Everyone writes the **E2E specs and unit tests for their own feature** (test IDs in [06-stages-and-tests.md](./06-stages-and-tests.md)). Andric owns the shared test setup.

---

## Non-Negotiable Implementation Pattern

Every task card follows the same pattern so the codebase reads like one person wrote it:

1. Server: route in `routes/*.routes.js` → `requireAuth` / `requireRole` → `validate(zodSchema)` → controller.
2. Controller: membership/ownership check → model method → `res.status(…).json({ data })`; expected failures `throw new AppError(…)`; everything else `next(err)`.
3. Realtime: persist via REST first, then call `broadcast.*` — never write to MongoDB from a socket handler except status updates (Ying Qi).
4. Client: component → Pinia store action → `api/*.api.js`. No `axios`/`socket` in components.
5. UI: Bootstrap + `tokens.css`; shared components from `components/ui`; works at 375 px before it's "done".
6. Every interactive element gets a `data-testid` (convention in [09-integration-contracts.md](./09-integration-contracts.md#9-data-testid-convention)).
7. PR includes its tests and a 375 px + 1280 px screenshot.

---

## Timeline

| Week | Goal | Everyone |
|---|---|---|
| **8** (now → pitch) | Stages 0–3: design lock, skeleton, platform, **live session core** | Day 1: review these docs together. Day 2: Andric + Dylan push skeleton + models; others build against stubs |
| **9** | Pitch (materials due **Mon 23:59**); stage 4 lesson plan; start MUST features | Pitch: Ying Qi drives the 4-laptop demo; each member presents their job scope |
| **10** | Stages 5–7: dashboard, catch-up, help flow | Mid-week integration day; first full E2E run |
| **11** | Stages 8–10: replay + GitHub, error groups + Spotlight, insights | Feature freeze for MUST + SHOULD at end of week |
| **12** | Stage 11–12: hardening, responsive pass, load test, README, slides, video | Record by Wednesday; **submit Thursday** (deadline Fri 9:00 am, no extensions) |

> Raise any teammate issue with the teaching team **at least 3 weeks before the deadline** (i.e. by Week 9) — later than that may be too late for them to act.

### Week 9 pitch: what each person must have running

The pitch rubric gives 25% for "a functioning task". Each member shows one working sub-feature:

| Member | Working at the pitch |
|---|---|
| Ying Qi | Start session → join by code → checkpoint broadcast to 3 laptops |
| Dylan | Editor + sandboxed preview + console capturing a real error; Load checkpoint |
| Karin | Telegram bot linked to a TA account; `sendMessage` from the server |
| Evan | Basic dashboard tiles updating live; TODO parser on the Mini Cart starter |
| Ray Yien | GitHub import of a public repo into a draft lesson |
| Andric | Register/login, courses, enrol by code, role guards; app deployed once (even if early) |

---

## Task Cards

### 🟣 Ying Qi — "Live Session Engine" (F1)

| What | File(s) | Done |
|---|---|:---:|
| Socket.IO server setup, cookie-JWT handshake auth | `server/sockets/index.js` | ⬜ |
| Rooms + join/leave + reconnection handling | `server/sockets/session.handlers.js` | ⬜ |
| Broadcast helpers used by others | `server/sockets/broadcast.js` | ⬜ |
| Start / join / end session, settings | `routes/sessions.routes.js`, `controllers/sessions.controller.js` | ⬜ |
| Save / list checkpoints | `routes/checkpoints.routes.js`, `controllers/checkpoints.controller.js` | ⬜ |
| Join codes | `server/lib/join-code.js` | ⬜ |
| Status classifier + dashboard cache + idle sweep **(proposed)** | `server/lib/status-classifier.js`, `server/lib/dashboard-cache.js` | ⬜ |
| Client socket wrapper + session store | `client/src/api/socket.js`, `client/src/stores/session.js` | ⬜ |
| Join page, join code card, QR, checkpoint toast, connection pill | `views/JoinView.vue`, `components/session/*` | ⬜ |
| Load simulator **(proposed)** | `scripts/simulate-class.js` | ⬜ |
| Tests | T-SESS-*, T-STAT-*, T-LOAD-* | ⬜ |

**Boundary:** owns transport and the status pipeline. Does not render tiles (Evan) or the workspace (Dylan).
**Hands to others:** `broadcast.toStudents/toStaff/toUser`, `sessionStore` with `latestCheckpoint`, `connectionState`.

---

### 🔵 Dylan — "Student Workspace" (F2) + Data Layer

| What | File(s) | Done |
|---|---|:---:|
| **All Mongoose models** + indexes + model methods | `server/models/*.model.js` | ⬜ |
| Workspace API: autosave, catch-up, my attempts | `routes/workspace.routes.js`, `controllers/workspace.controller.js` | ⬜ |
| Workspace page (tabs on xs, panes on lg) | `views/WorkspaceView.vue` | ⬜ |
| CodeMirror wrapper, file tabs | `components/workspace/CodeEditor.vue`, `FileTabs.vue` | ⬜ |
| Sandbox: srcdoc builder, probe, localStorage shim, mock API | `client/src/sandbox/*`, `components/workspace/PreviewFrame.vue` | ⬜ |
| Console panel, error gutter, status pill | `components/workspace/ConsolePanel.vue` | ⬜ |
| Done / I'm lost / Help bottom bar; emits `status:update` on change | `components/workspace/DoneButton.vue`, `stores/workspace.js` | ⬜ |
| Load checkpoint + restore attempt (uses Ray Yien's `DiffView`) | `stores/workspace.js` | ⬜ |
| Automatic step checks (COULD) | `client/src/sandbox/checks.js`, `server/lib/check-generator.js` | ⬜ |
| Tests | T-SBX-*, T-CATCH-01/02/03/06 | ⬜ |

**Boundary:** owns everything inside the student's workspace and every schema file. Others request schema changes by PR to Dylan.
**Spike first (Week 8):** `localStorage` shim + error file/line from `//# sourceURL` in Chrome.

---

### 🟠 Karin — "Help Flow + Telegram" (F3)

| What | File(s) | Done |
|---|---|:---:|
| Help request API + transitions (conditional updates) | `routes/help.routes.js`, `controllers/help.controller.js` | ⬜ |
| Telegram service: link codes, polling, send queue, 429 retry | `server/services/telegram.service.js`, `services/fakes/telegram.fake.js` | ⬜ |
| Telegram link/unlink endpoints + profile card | `routes/telegram.routes.js`, `controllers/telegram.controller.js`, `components/help/TelegramLinkCard.vue` | ⬜ |
| In-person ping to on-duty staff | `controllers/help.controller.js` | ⬜ |
| Student help panel (send, cancel, see reply/tick) | `components/help/StudentHelpPanel.vue` | ⬜ |
| TA queue page + review page (side-by-side via `DiffView`) | `views/HelpQueueView.vue`, `views/HelpReviewView.vue`, `components/help/*` | ⬜ |
| `sendDigest()` used by Andric's report | `server/services/telegram.service.js` | ⬜ |
| AI hints (COULD) **(proposed)** | `routes/hints.routes.js`, `services/llm.service.js`, `lib/hint-rules.js` | ⬜ |
| Tests | T-HELP-*, T-TG-* | ⬜ |

**Boundary:** owns the help lifecycle and every outgoing Telegram message. Andric decides digest *content*; Karin's service sends it.

---

### 🟢 Evan — "Prof View: Lesson Plan + Dashboard" (F4)

| What | File(s) | Done |
|---|---|:---:|
| Lessons + steps API (upload, edit, reorder, delete, reparse) | `routes/lessons.routes.js`, `controllers/lessons.controller.js` | ⬜ |
| TODO parser | `server/lib/todo-parser.js` | ⬜ |
| New lesson (upload tab) + lesson editor | `views/LessonNewView.vue`, `views/LessonEditView.vue`, `components/lesson/*` | ⬜ |
| Teach view: prof editor, plan, Save checkpoint, summary bar | `views/TeachView.vue` | ⬜ |
| Tiles, grid, filters, lost count | `components/dashboard/*`, `stores/dashboard.js` | ⬜ |
| Dashboard snapshot endpoint | `routes/dashboard.routes.js`, `controllers/dashboard.controller.js` | ⬜ |
| Error normaliser + grouping **(proposed)** | `server/lib/error-normalizer.js` (+ grouping in `dashboard-cache` summary, agreed with Ying Qi) | ⬜ |
| Spotlight (pick snapshot, anonymise, broadcast) **(proposed)** | `controllers/dashboard.controller.js`, `server/lib/spotlight.js` | ⬜ |
| Tests | T-LES-*, T-TODO-*, T-DASH-*, T-ERRG-* | ⬜ |

**Boundary:** owns what the prof sees during class. Uses Ying Qi's dashboard cache data; doesn't change the socket transport.

---

### 🟡 Ray Yien — "Replay, Library, Diff, GitHub" (F5)

| What | File(s) | Done |
|---|---|:---:|
| Diff utility + `DiffView` (split/unified) | `client/src/utils/diff-summary.js`, `components/replay/DiffView.vue` | ⬜ |
| "What you missed" panel (embedded in Dylan's workspace) | `components/replay/WhatYouMissed.vue` | ⬜ |
| Lesson library (search, list) in course page | `components/lesson/LessonLibrary.vue` | ⬜ |
| GitHub service: import, OAuth, export via Git Data API | `server/services/github.service.js`, `services/fakes/github.fake.js` | ⬜ |
| Import endpoint + import tab in New lesson | `controllers/lessons.controller.js` (`importFromGithub` only), `components/lesson/GithubImportTab.vue` | ⬜ |
| Replay API, notes API, search API | `routes/replays.routes.js`, `routes/notes.routes.js`, controllers | ⬜ |
| Replay list + viewer, timeline, notes drawer, export dialog | `views/ReplayListView.vue`, `views/ReplayView.vue`, `components/replay/*` | ⬜ |
| Tests | T-REP-*, T-GH-*, T-CATCH-04/05/07 | ⬜ |

**Boundary:** owns everything after class except insights. `importFromGithub` is the only function Ray Yien adds to Evan's lessons controller (agreed signature in [09-integration-contracts.md](./09-integration-contracts.md)).

---

### 🔴 Andric — "Platform, Insights, Deployment, UI Consistency" (F6)

| What | File(s) | Done |
|---|---|:---:|
| Express app, env config, error middleware, auth middleware | `server/app.js`, `server/server.js`, `config/env.js`, `middleware/*` | ⬜ |
| Auth + profile + courses API | `routes/auth.routes.js`, `routes/users.routes.js`, `routes/courses.routes.js`, controllers | ⬜ |
| Vue app shell, router + guards, auth/courses stores | `client/src/main.js`, `App.vue`, `router/index.js`, `stores/auth.js`, `stores/courses.js` | ⬜ |
| UI kit + tokens (navbar, toast, modal, empty state, status pill) | `components/ui/*`, `styles/tokens.css` | ⬜ |
| Landing, login, register, courses, course page shell, profile, 404 | `views/*` (those pages) | ⬜ |
| Report builder + report API + trends + digest content | `server/lib/report-builder.js`, `routes/reports.routes.js`, `views/ReportView.vue`, `components/insights/*` | ⬜ |
| Seed script | `server/seed/*` (lesson fixture files written with Evan) | ⬜ |
| Playwright config, fixtures, test routes **(proposed)** | `playwright.config.js`, `tests/e2e/fixtures.js`, `routes/test.routes.js` | ⬜ |
| Deployment + README assembly **(proposed)** | — | ⬜ |
| Tests | T-PLAT-*, T-INS-*, T-VAL-*, T-RESP-05, T-SEED-*, T-SMOKE-01 | ⬜ |

**Boundary:** owns the frame everyone builds inside and the final responsive/consistency pass (reviews every PR's screenshots).

---

## Dependencies & Unblocking

```
Andric: skeleton + auth middleware ─┐
Dylan:  models ─────────────────────┼──▶ everyone
                                    │
Ying Qi: sockets + broadcast ───────┼──▶ Evan (dashboard), Karin (help:new), Dylan (status:update)
Ray Yien: DiffView ─────────────────┼──▶ Dylan (What you missed), Karin (side-by-side)
Karin: telegram.service ────────────┴──▶ Andric (digest)
```

| Risk | Mitigation |
|---|---|
| Skeleton/models late → everyone blocked | Contracts in [09-integration-contracts.md](./09-integration-contracts.md) are fixed on Day 1, so others code against stubs and fake data |
| Sandbox browser quirks (localStorage, error lines) | Dylan spikes them in Week 8 before building the full workspace |
| Socket bugs only appear with many clients | Ying Qi's simulator available from Week 9; every dashboard PR tested with 25 fake students |
| Deployment surprises (WebSockets, sleeping hosts) | Andric deploys a "hello socket" build in Week 8 and redeploys weekly |
| Scope creep | COULD items only after the full MUST demo passes (stage gate) |

---

## Git Workflow

| Rule | Detail |
|---|---|
| Branches | `main` (protected, always deployable) · `feat/<name>-<topic>` e.g. `feat/karin-help-queue` |
| Pull requests | Small; one reviewer who isn't the author; CI (unit + E2E) must pass |
| Commits | `feat:`, `fix:`, `test:`, `docs:`, `refactor:` |
| Shared files | Changes to `shared/constants.js`, models or contracts need a ping in the team channel first |
| Weekly | Integration + redeploy every Friday; update the "Done" columns above |

---

## Open Questions

Answer these before Stage 0 is ticked:

1. Confirm the **(proposed)** items: classifier → Ying Qi; error groups + Spotlight → Evan; AI hints → Karin; test setup + README + deployment → Andric.
2. AWS or Render for deployment? (WebSocket support and "doesn't sleep" are the deciding factors.)
3. Do all six members appear in the video? (FAQ 9 — check with our section's instructional team.)
4. Will a prof or TA try CodeAlong in one lab before Week 12? (The proposal plans a classmate poll + TA chat to validate the problem.)
5. LLM provider for hints — or drop hints and keep the rule table only?
