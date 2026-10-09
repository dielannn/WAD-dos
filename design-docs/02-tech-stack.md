# CodeAlong — Tech Stack & Project Structure

## Technology Stack

| Layer | Technology | Why | Course link |
|---|---|---|---|
| Markup / styling | HTML5, CSS3, **Bootstrap 5** + Bootstrap Icons | Responsive grid and components out of the box; our own CSS tokens on top for a consistent look | WAD2 (Bootstrap, responsive design), IDP (UI patterns) |
| Frontend framework | **Vue 3** (Composition API, single-file components) | Reactive dashboard and workspace; covered in WAD2 and the final exam | WAD2 |
| Routing | **Vue Router 4** | Client-side pages with route guards by role | WAD2 |
| Client state | **Pinia** | One store per domain; shared between components without prop drilling | WAD2 (state management) |
| Build tool | **Vite** | Dev server with hot reload; builds the static bundle Express serves | New (tooling only) |
| In-browser editor | **CodeMirror 6** | Lightweight, mobile-friendly editor with gutter markers for error lines | New — credited |
| HTTP client | **axios** | Promise-based async HTTP from Vue to our API | WAD2 (async, axios) |
| Realtime | **Socket.IO** (server + client) | Rooms, acknowledgements and automatic reconnection over WebSockets | New — credited |
| Backend | **Node.js + Express 4** | REST API; same stack as IS113 | IS113 |
| Database | **MongoDB Atlas** + **Mongoose 8** | Document model fits lessons/files/checkpoints; free M0 tier | IS113 |
| Auth | **bcrypt** + **jsonwebtoken** in an httpOnly cookie | Stateless auth that also works for the Socket.IO handshake | IS113 (bcrypt), new (JWT) |
| Validation | **zod** | One schema per request body; readable error messages per field | New — credited |
| Diffing | **diff** (jsdiff) | Line diffs for "What you missed" and replay | New — credited |
| QR codes | **qrcode** | Join-code QR on the projector | New — credited |
| External APIs | **Telegram Bot API**, **GitHub REST API**, LLM API (KIV) | See [05-core-logic.md](./05-core-logic.md) | Briefing requirement |
| Testing | **Playwright** (E2E), **Vitest** (unit), **mongodb-memory-server** | Multi-browser-context E2E; repeatable tests without Atlas | Briefing requirement |
| Dev tooling | nodemon, concurrently, ESLint + Prettier | Auto-restart, run client + server together, consistent style | — |

### Why not React / plain HTML pages?

Vue is covered in the syllabus (so the teaching team can help us) and in the final exam (FAQ 3). Plain multi-page HTML would make the live dashboard and workspace far harder to keep reactive.

---

## Project Folder Structure (npm workspaces monorepo)

```
codealong/
├── package.json                ← workspaces: client, server; root scripts (dev, build, start, test)
├── README.md
├── .env.example                ← copy to server/.env (never commit .env)
├── playwright.config.js        ← projects: desktop-chrome (1280px), mobile-chrome (375px)
├── vitest.config.js
│
├── shared/
│   └── constants.js            ← Roles, tile states, help statuses, socket event names, limits
│
├── client/                     ← Vue 3 + Vite
│   ├── index.html
│   ├── vite.config.js          ← dev proxy: /api and /socket.io → http://localhost:3000
│   └── src/
│       ├── main.js
│       ├── App.vue
│       ├── router/index.js     ← routes + beforeEach role guards
│       ├── api/
│       │   ├── http.js         ← axios instance (withCredentials, error unwrapping)
│       │   ├── socket.js       ← single Socket.IO client, typed emit/on helpers
│       │   └── *.api.js        ← auth, courses, lessons, sessions, help, replays, reports
│       ├── stores/             ← Pinia: auth, courses, session, workspace, dashboard, help, replay
│       ├── views/              ← one file per page (see 04-pages-routes-and-events.md)
│       ├── components/
│       │   ├── ui/             ← AppNavbar, AppToast, ConfirmModal, EmptyState, StatusPill (Andric)
│       │   ├── workspace/      ← CodeEditor, FileTabs, PreviewFrame, ConsolePanel, DoneButton (Dylan)
│       │   ├── dashboard/      ← StudentTile, TileGrid, ErrorGroupList, ClassSummaryBar (Evan)
│       │   ├── lesson/         ← StepList, StepEditor, StarterUpload (Evan)
│       │   ├── session/        ← JoinCodeCard, QrDisplay, CheckpointToast (Ying Qi)
│       │   ├── help/           ← HelpQueue, HelpRequestCard, SideBySideView (Karin)
│       │   ├── replay/         ← CheckpointTimeline, DiffView, NoteEditor (Ray Yien)
│       │   └── insights/       ← StepFunnelChart, TopErrorsTable (Andric)
│       ├── composables/        ← useSandbox, useAutosave, useBreakpoint, useCountdown
│       ├── utils/              ← diff-summary.js (Ray Yien), format helpers
│       ├── sandbox/
│       │   └── probe.js        ← injected into the preview iframe (console/error/network capture)
│       └── styles/
│           ├── tokens.css      ← colours, spacing, radii, tile-state colours
│           └── main.css
│
├── server/                     ← Node + Express + Socket.IO
│   ├── server.js               ← creates HTTP server, attaches Socket.IO, connects Mongo, listens
│   ├── app.js                  ← Express app: middleware, /api routes, static client/dist, SPA fallback, errors
│   ├── config/env.js           ← reads + validates process.env once
│   ├── models/                 ← Mongoose schemas (Dylan owns; see 03-data-model.md)
│   ├── routes/                 ← auth, users, telegram, courses, lessons, sessions, checkpoints,
│   │                              dashboard, workspace, help, hints, replays, notes, github,
│   │                              reports, test (test env only)
│   ├── controllers/            ← one per route file
│   ├── middleware/
│   │   ├── auth.js             ← requireAuth, requireRole, loadCourseMembership
│   │   ├── validate.js         ← validate(zodSchema) → 400 with field errors
│   │   └── errors.js           ← notFound, errorHandler (AppError → JSON envelope)
│   ├── validators/             ← zod schemas per route group
│   ├── sockets/
│   │   ├── index.js            ← io auth middleware (JWT from cookie), connection handler
│   │   ├── session.handlers.js ← join/leave, status:update, lost:toggle, heartbeat (Ying Qi)
│   │   └── broadcast.js        ← helpers other controllers call: toStudents(), toStaff(), toUser()
│   ├── services/
│   │   ├── telegram.service.js ← sendMessage, link codes, long-polling loop (Karin)
│   │   ├── github.service.js   ← importRepo, exportReplay via Git Data API (Ray Yien)
│   │   ├── llm.service.js      ← hints (KIV) with cache + rule fallback
│   │   └── fakes/              ← in-memory fakes used when EXTERNAL_APIS=mock
│   ├── lib/                    ← pure functions, unit-tested
│   │   ├── todo-parser.js      ← starter files → ordered steps (Evan)
│   │   ├── error-normalizer.js ← raw error → group key (Evan)
│   │   ├── status-classifier.js← raw status + checkpoint → tile state (Ying Qi)
│   │   ├── dashboard-cache.js  ← in-memory per-session status cache + summary (Ying Qi)
│   │   ├── report-builder.js   ← statuses + help requests → insights (Andric)
│   │   └── join-code.js        ← unambiguous 6-char codes
│   └── seed/
│       ├── seed.js             ← clears + inserts demo data (07-seed-and-demo.md)
│       └── fixtures/mini-cart/ ← starter files with TODOs for the seed lesson
│
├── scripts/
│   └── simulate-class.js       ← N fake students over Socket.IO (load test + fuller demo dashboard)
│
└── tests/
    ├── e2e/                    ← Playwright specs (see 06-stages-and-tests.md)
    │   ├── fixtures.js         ← profPage, taPage, studentPages(n), seeded test DB
    │   └── *.spec.js
    └── unit/                   ← Vitest specs for server/lib and client/src/sandbox
```

---

## How Everything Connects

### REST request flow

```
Vue component
   │  calls store action
   ▼
Pinia store ──▶ client/src/api/*.api.js ──▶ axios (cookie sent automatically)
                                              │
                                              ▼
                                   Express app.js  /api/...
                                              │
                                   routes/*.routes.js
                                              │
                                   middleware: requireAuth → requireRole → validate(schema)
                                              │
                                   controllers/*.controller.js
                                     │            │             │
                                     ▼            ▼             ▼
                                 models/      services/     sockets/broadcast.js
                                (MongoDB)   (GitHub/Telegram) (notify rooms)
                                              │
                                   res.json({ data })  or  next(err) → errors.js
```

### Realtime flow

```
Student browser                         Server                              Prof / TA browser
───────────────                         ──────                              ─────────────────
socket connect (cookie JWT) ──────────▶ io auth middleware
emit session:join {sessionId} ────────▶ check enrolment + session live
                                        join room  session:<id>:students
                       ◀──── ack {latestCheckpoint, myStatus}
edits → probe reports error / Done
emit status:update (only on change) ──▶ status-classifier → dashboard-cache
                                        upsert StudentStatus (throttled)
                                        emit dashboard:tile ───────────────▶ tile re-renders
                                        emit dashboard:summary (≤1/s) ────▶ counts, error groups

                                        POST /api/sessions/:id/checkpoints ◀── "Save checkpoint"
                       ◀──── checkpoint:new  (room session:<id>:students)
```

### Concrete example: prof saves a checkpoint

1. **Prof** clicks **Save checkpoint** in `TeachView.vue` → `sessionStore.saveCheckpoint()` → `POST /api/sessions/:id/checkpoints` with the current files and step ID.
2. **`checkpoints.routes.js`** runs `requireAuth` → `validate(createCheckpointSchema)`.
3. **`checkpoints.controller.create`**:
   - loads the session; 404 if missing, 409 if `status !== 'live'`
   - checks the caller is staff of the session's course (403 otherwise)
   - assigns the next `seq`, creates the `Checkpoint`, sets `session.currentStepId`
   - calls `broadcast.toStudents(sessionId, 'checkpoint:new', payload)` and `broadcast.toStaff(...)`
   - asks `dashboard-cache` to re-classify every student against the new target step (some become `behind`)
   - responds `201 { data: checkpoint }`
4. **Students' browsers** receive `checkpoint:new`: a toast appears, the "Load checkpoint" button updates, the step list advances.
5. **MongoDB** persists the checkpoint; replay later reads these in `seq` order.

---

## Coding Conventions

| Concern | Convention |
|---|---|
| File names | kebab-case; server suffixes `.routes.js`, `.controller.js`, `.service.js`, `.model.js`; Vue views `PascalCaseView.vue`, components `PascalCase.vue` |
| Controller pattern | `async (req, res, next) => { try { … } catch (err) { next(err) } }`; throw `new AppError(status, code, message)` for expected failures |
| Validation | `validate(schema)` middleware before the controller; controller assumes `req.body` is valid |
| Authorization | Coarse role checks in middleware; course membership/ownership checks in the controller (see [09-integration-contracts.md](./09-integration-contracts.md#4-authorization-layering)) |
| Vue components | Props typed with `defineProps`; emitted events declared with `defineEmits`; no direct API calls |
| CSS | Use Bootstrap utilities first; custom styles only via variables from `tokens.css`; mobile-first media queries |
| Commits | Conventional prefixes: `feat:`, `fix:`, `test:`, `docs:`, `refactor:` |

---

## WAD2 Concepts Mapped to the Project

Useful for the "explain how you used JS/Vue/CSS" part of the video.

| Concept | Where we use it |
|---|---|
| Vue reactivity (`ref`, `reactive`) | Workspace file buffers, dashboard tiles |
| `computed` | Tile counts per state, filtered tiles, progress bar %, number of changed lines in "What you missed" |
| `watch` | Autosave debounce on editor changes; re-run preview when files change |
| Components, props, emits | `StudentTile` (props: tile) → emits `select`; `CodeEditor` emits `update:modelValue`, `save` |
| `v-for`, `v-if`, `v-model`, `:class` | Tile grid, filter chips, step editor, state colours |
| Lifecycle hooks | `onMounted` connects socket/joins session; `onBeforeUnmount` leaves room and clears timers |
| Vue Router + guards | Role-guarded routes (`meta.roles`), `/join/:code` deep link from QR |
| Pinia | Shared session state between editor, toolbar, console and step list |
| Async JS (`async/await`, Promises) | axios calls, Socket.IO acknowledgements, server `fetch` to Telegram/GitHub |
| JSON | REST payloads, socket messages, lesson mock-API data |
| DOM events / `postMessage` | Sandbox iframe ↔ parent communication |
| `localStorage` | Remembered editor font size and last-open file (per-device conveniences only) |
| Bootstrap grid + breakpoints | Workspace switches from 3 panes (lg+) to tabs (< md); tile grid columns per breakpoint |
| Express + MongoDB (from IS113) | REST API, Mongoose models, bcrypt |

---

## Responsive Breakpoints (Bootstrap 5)

| Name | Min width | Our test width | Must look right |
|---|---:|---:|---|
| xs | 0 | **375 px** (iPhone 6) and **575 px** (demo width) | Every page; workspace in tab mode |
| sm | 576 px | 600 px | Forms two-column where useful |
| md | 768 px | 800 px | Tile grid 4 columns |
| lg | 992 px | 1024 px | Workspace three panes; teach view editor + side panel |
| xl | 1200 px | 1280 px | Full teach view: editor, plan, dashboard |
| xxl | 1400 px | 1440 px | No stretched lines (max content width) |

Both portrait and landscape are checked at xs (FAQ 17 tests "different screen sizes and orientations").

---

## Third-Party Libraries (credit these)

FAQ 6 requires every public library to be named in the presentation and in code comments.

| Library | Licence | Used for |
|---|---|---|
| Vue, Vue Router, Pinia | MIT | Frontend framework, routing, state |
| Bootstrap, Bootstrap Icons | MIT | Styling, icons |
| CodeMirror 6 | MIT | In-browser editor |
| Socket.IO | MIT | Realtime |
| axios | MIT | HTTP client |
| Express, Mongoose, bcrypt, jsonwebtoken, cookie-parser, helmet, express-rate-limit | MIT / BSD | Backend |
| zod | MIT | Validation |
| diff (jsdiff) | BSD-3 | Line diffs |
| qrcode | MIT | QR join codes |
| Playwright, Vitest, mongodb-memory-server | Apache-2.0 / MIT | Testing |

---

## Environment Variables (`server/.env`)

```env
NODE_ENV=development
PORT=3000
APP_URL=http://localhost:3000            # used in QR codes and Telegram links
MONGODB_URI=mongodb+srv://<USER>:<PASS>@<CLUSTER>/codealong?retryWrites=true&w=majority
JWT_SECRET=<long-random-string>
JWT_TTL_HOURS=12

EXTERNAL_APIS=live                       # live | mock  (tests always use mock)
TELEGRAM_BOT_TOKEN=<from @BotFather>
TELEGRAM_BOT_USERNAME=CodeAlongBot
GITHUB_TOKEN=<fine-grained token, public-repo read>   # server-side imports: 5,000 req/h
GITHUB_OAUTH_CLIENT_ID=<oauth app>                    # student replay export
GITHUB_OAUTH_CLIENT_SECRET=<oauth app>
LLM_API_KEY=                             # optional (KIV); hints fall back to rules when empty
```

> `.env` is in `.gitignore`. Only `.env.example` (with placeholders) is committed. Credentials the markers need go in the README, not in the repo.

### npm scripts (root)

| Script | What it does |
|---|---|
| `npm install` | Installs all workspaces |
| `npm run dev` | Vite (5173) + nodemon server (3000) via `concurrently`; Vite proxies `/api` and `/socket.io` |
| `npm run build` | Builds `client/dist` |
| `npm start` | Production: Express serves the API, Socket.IO and `client/dist` on one port |
| `npm run seed` | Runs `server/seed/seed.js` against `MONGODB_URI` |
| `npm run test:unit` | Vitest |
| `npm run test:e2e` | Builds client, starts server on in-memory Mongo with `EXTERNAL_APIS=mock`, runs Playwright |
| `npm run simulate -- --session <joinCode> --students 40` | Load test / fills the dashboard for demos |

---

## Deployment

**Constraint:** Socket.IO needs a long-running Node process. Serverless platforms that only run short functions (e.g. Vercel functions) don't fit.

| Option | Notes |
|---|---|
| **AWS** (Elastic Beanstalk or Lightsail container) — current plan, TBC | One Node service; set env vars in the console; enable WebSocket support on the load balancer |
| Render / Railway (fallback) | One web service; free tiers may sleep when idle — check before submission, since markers test after the deadline |
| Database | MongoDB Atlas M0 (free); allow the host's outbound IP in Network Access |

Checklist (owner: Andric):
- [ ] `npm run build && npm start` works locally on a clean clone
- [ ] Deployed URL loads the landing page at `/`
- [ ] Socket.IO connects over `wss://` in production (check DevTools → Network → WS)
- [ ] Telegram bot runs in the deployed process only (avoid two pollers on one token)
- [ ] Seeded demo accounts work on the deployed DB
- [ ] Don't restart/redeploy after submission unless the teaching team asks; keep running until Week 18
