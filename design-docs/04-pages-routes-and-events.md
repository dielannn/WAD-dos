# CodeAlong — Pages, API Routes & Socket Events

## Page Map

```
PUBLIC                         ANY LOGGED-IN USER                 STAFF (TA / PROF)
──────                         ──────────────────                 ─────────────────
Landing  /                     My courses        /courses         Teach view        /s/:id/teach   (prof)
Login    /login                Course page       /courses/:id     Help queue        /s/:id/help
Register /register             Join session      /join[/:code]    Help review       /help/:requestId
404                            Student workspace /s/:id/code      Lesson editor     /lessons/:id/edit (prof)
                               Replays           /replays         New lesson        /courses/:id/lessons/new (prof)
                               Replay viewer     /replays/:id     Insights report   /sessions/:id/report
                               Profile           /profile
```

---

## Client Pages

Each page lists its owner, who can open it, and how its layout responds. All pages share `AppNavbar` + `AppToast` from Andric's UI kit.

| Path | View | Owner | Who | Layout: xs (375–575 px) → lg/xl (≥ 992 px) |
|---|---|---|---|---|
| `/` | `LandingView` | Andric | Public | Hero + 3 feature cards stacked → hero left, cards in a row; "Join a class" CTA always visible |
| `/login`, `/register` | `LoginView`, `RegisterView` | Andric | Public | Full-width card → centred card max 420 px |
| `/courses` | `CoursesView` | Andric | Logged in | Course cards 1 col → 3 cols; prof sees "New course"; student sees "Enrol with code" |
| `/courses/:id` | `CourseView` | Andric (shell) + Ray Yien (lesson library) | Staff / enrolled | Tabs (Lessons · Sessions · People) → same tabs, lessons as a table with search |
| `/courses/:id/lessons/new` | `LessonNewView` | Evan (upload) + Ray Yien (GitHub import tab) | Prof | Stepper: 1 Source → 2 Review files → 3 Review steps |
| `/lessons/:id/edit` | `LessonEditView` | Evan | Prof | Step list only, editor opens as a sheet → file editor left, draggable step list right |
| `/join`, `/join/:code` | `JoinView` | Ying Qi | Logged in (redirects to login with `returnTo`) | Big 6-box code input, auto-submits when complete; `/join/:code` pre-fills from QR |
| `/s/:id/teach` | `TeachView` | Evan (editor, plan, dashboard) + Ying Qi (session bar) | Prof | **xs:** dashboard-first (summary bar, filters, tiles); editor behind a tab. **xl:** editor 8 cols + terminal below; plan + error groups 4 cols; tile grid full width below; join code card collapsible |
| `/s/:id/code` | `WorkspaceView` | Dylan (+ Ray Yien's "What you missed" panel) | Enrolled student | **xs:** tabs *Code · Preview · Console · Steps*, sticky bottom bar with **Done**, **I'm lost**, **Help**. **lg:** three panes (editor · preview · console) with step list in an offcanvas |
| `/s/:id/help` | `HelpQueueView` | Karin | Staff | Queue cards 1 col → queue list left, selected request right |
| `/help/:requestId` | `HelpReviewView` | Karin | Staff | Stacked: student code, then checkpoint, then reply → side-by-side diff with reply box under |
| `/replays` | `ReplayListView` | Ray Yien | Logged in | List grouped by course, search box on top |
| `/replays/:sessionId` | `ReplayView` | Ray Yien | Staff / enrolled | Timeline as horizontal scroller, code below, preview behind a tab → timeline left, code + preview right, notes drawer |
| `/sessions/:id/report` | `ReportView` | Andric | Staff | Cards stacked; funnel chart scrolls horizontally → 2-col grid: funnel, top errors, recurring steps, digest panel |
| `/profile` | `ProfileView` | Andric (profile) + Karin (Telegram link card) | Logged in | Single column |
| `*` | `NotFoundView` | Andric | Public | Friendly 404 with links back |

**Route guards (`router/index.js`):** `meta: { auth: true, roles: ['prof'] }`. Unauthenticated → `/login?returnTo=<path>`. Wrong role → toast "You don't have access to that page" and redirect to `/courses`. Course-level checks happen on the server (the client cannot know membership reliably).

### UI rules that apply to every page (IDP / 10 rules of thumb)

| Rule | How we apply it |
|---|---|
| Visibility of system status | Connection pill in the navbar (Live · Reconnecting… · Offline); saving indicator in the workspace |
| Match the real world | Labels say "Done", "I'm lost", "Load checkpoint", "Check my work" — no jargon like "sync state" |
| User control & freedom | Load checkpoint always keeps your attempt (restorable); help requests can be cancelled |
| Consistency | Same tile colours, icons and labels everywhere (`StatusPill`); one button style per action type |
| Error prevention | Confirm modal before End session / Delete lesson; join code input only accepts valid characters |
| Recognition over recall | Join code and QR stay visible on the teach view; current step pinned at the top of the workspace |
| Colour is never the only signal | Every tile state has an icon + text label as well as a colour |

---

## REST API Catalog

All endpoints are under `/api`, return `{ data }` / `{ error }` (see [09-integration-contracts.md](./09-integration-contracts.md#3-api-response-envelope)), and use JSON. "Staff" / "Enrolled" checks happen in the controller after loading the course.

### Auth & profile — `auth.routes.js`, `users.routes.js` (Andric), `telegram.routes.js` (Karin)

| Method | Path | What happens | Auth | Who |
|---|---|---|:---:|---|
| POST | `/api/auth/register` | Validate → hash → create `student` → set cookie → 201 user | No | Any |
| POST | `/api/auth/login` | Validate → compare bcrypt → set cookie → 200 user | No | Any (rate-limited) |
| POST | `/api/auth/logout` | Clear cookie → 204 | Yes | Any |
| GET | `/api/auth/me` | Current user (used on app load) | Yes | Any |
| PATCH | `/api/users/me` | Update name / password (current password required) | Yes | Any |
| POST | `/api/users/me/telegram/link` | Issue one-time link code → returns `https://t.me/<bot>?start=<code>` | Yes | Any (Karin) |
| DELETE | `/api/users/me/telegram` | Unlink Telegram | Yes | Any (Karin) |

### Courses — `courses.routes.js` (Andric)

| Method | Path | What happens | Who |
|---|---|---|---|
| GET | `/api/courses` | Courses where I am staff or enrolled | Any |
| POST | `/api/courses` | Create course; owner added to `staff` as prof; generates `enrolCode` | Prof |
| POST | `/api/courses/enrol` | `{ enrolCode }` → add me to `students` | Student |
| GET | `/api/courses/:courseId` | Course + my membership role | Staff / enrolled |
| PATCH | `/api/courses/:courseId` | Edit title/term/section/archived | Owner |
| DELETE | `/api/courses/:courseId` | Delete with cascade (confirm in UI) | Owner |
| POST | `/api/courses/:courseId/staff` | `{ email }` → add as TA (promotes account to `ta` if student) | Owner |
| GET | `/api/courses/:courseId/people` | Staff + students (names only) | Staff |
| DELETE | `/api/courses/:courseId/students/:userId` | Remove student | Owner |

### Lessons & steps — `lessons.routes.js` (Evan) + GitHub import (Ray Yien)

| Method | Path | What happens | Who |
|---|---|---|---|
| GET | `/api/courses/:courseId/lessons?search=` | Lesson library (title/description search) | Staff / enrolled |
| POST | `/api/courses/:courseId/lessons` | Create from uploaded files → runs TODO parser → steps → `draft` | Prof |
| POST | `/api/courses/:courseId/lessons/import` | `{ repoUrl }` → GitHub service fetches files → TODO parser → `draft` | Prof (Ray Yien) |
| GET | `/api/lessons/:lessonId` | Lesson with files + steps | Staff (students get steps only, no files until a session starts) |
| PATCH | `/api/lessons/:lessonId` | Edit title/description/week/files/entryFile/mockApi/status | Prof |
| POST | `/api/lessons/:lessonId/reparse` | Re-run TODO parser on current files; returns proposed steps (not saved) | Prof |
| PUT | `/api/lessons/:lessonId/steps` | Replace full ordered step list (create/reorder/delete in one go) | Prof |
| PATCH | `/api/lessons/:lessonId/steps/:stepId` | Edit one step | Prof |
| DELETE | `/api/lessons/:lessonId/steps/:stepId` | Delete one step (re-numbers the rest) | Prof |
| DELETE | `/api/lessons/:lessonId` | Delete if never taught, else 409 → archive | Prof |

### Sessions & checkpoints — `sessions.routes.js` (Ying Qi), `checkpoints.routes.js` (Ying Qi)

| Method | Path | What happens | Who |
|---|---|---|---|
| POST | `/api/lessons/:lessonId/sessions` | Start session → join code + QR data URL | Prof |
| POST | `/api/sessions/join` | `{ joinCode }` → `{ sessionId, role }` (then the client opens the socket) | Enrolled / staff |
| GET | `/api/sessions/:sessionId` | Session, lesson steps, latest checkpoint, my role | Member |
| PATCH | `/api/sessions/:sessionId/settings` | Toggle `allowCatchUp`, `hintsEnabled`, `idleAfterSec` | Prof |
| POST | `/api/sessions/:sessionId/end` | End → build report → broadcast `session:ended` | Prof |
| GET | `/api/courses/:courseId/sessions` | Past + live sessions | Staff / enrolled |
| POST | `/api/sessions/:sessionId/checkpoints` | Save checkpoint → broadcast `checkpoint:new` | Prof |
| GET | `/api/sessions/:sessionId/checkpoints` | All checkpoints (files included) | Member |

### Dashboard & Spotlight — `dashboard.routes.js` (Evan)

| Method | Path | What happens | Who |
|---|---|---|---|
| GET | `/api/sessions/:sessionId/dashboard` | Initial snapshot: tiles (no code), counts, lost count, error groups | Staff |
| POST | `/api/sessions/:sessionId/spotlight` | `{ errorKey, helpRequestId }` → build anonymised excerpt → broadcast `spotlight:show` | Prof |
| DELETE | `/api/sessions/:sessionId/spotlight` | Clear → broadcast `spotlight:hide` | Prof |

### Student workspace — `workspace.routes.js` (Dylan)

| Method | Path | What happens | Who |
|---|---|---|---|
| GET | `/api/sessions/:sessionId/workspace` | My autosave (or starter files if none) + my status | Enrolled |
| PUT | `/api/sessions/:sessionId/workspace` | Autosave (debounced 2 s on the client; ≤ 200 KB) | Enrolled |
| POST | `/api/sessions/:sessionId/catch-up` | Save `before_catchup` snapshot of my files → return latest checkpoint files + snapshot id | Enrolled (if `allowCatchUp`) |
| GET | `/api/sessions/:sessionId/my-attempts` | My `before_catchup` snapshots (restore an attempt) | Enrolled |

### Help — `help.routes.js` (Karin)

| Method | Path | What happens | Who |
|---|---|---|---|
| POST | `/api/sessions/:sessionId/help-requests` | `{ type, stepId, files, message?, inPerson?, seat? }` → snapshot + request → `help:new` to staff → Telegram ping if in-person | Enrolled |
| GET | `/api/sessions/:sessionId/help-requests` | Staff: open/claimed queue · Student: my requests | Member |
| GET | `/api/help-requests/:requestId` | Request + snapshot files + checkpoint for that step | Staff, or the requesting student |
| POST | `/api/help-requests/:requestId/claim` | `open → claimed` (409 if already claimed) | Staff |
| POST | `/api/help-requests/:requestId/release` | `claimed → open` | Claiming staff |
| POST | `/api/help-requests/:requestId/pass` | `claimed → passed` (check only) → student gets a tick | Claiming staff |
| POST | `/api/help-requests/:requestId/reply` | `{ reply }` → `claimed → replied` | Claiming staff |
| POST | `/api/help-requests/:requestId/resolve` | `claimed → resolved` (in-person done) | Claiming staff |
| POST | `/api/help-requests/:requestId/cancel` | `open → cancelled` | Requesting student |

### Replay, notes, search, export — `replays.routes.js`, `notes.routes.js` (Ray Yien)

| Method | Path | What happens | Who |
|---|---|---|---|
| GET | `/api/replays` | Ended sessions in my courses (+ whether I attended) | Any |
| GET | `/api/sessions/:sessionId/replay` | Checkpoints in order + step titles + per-checkpoint diff stats | Member |
| GET | `/api/sessions/:sessionId/notes` | My notes/bookmarks | Member |
| POST | `/api/sessions/:sessionId/notes` | Create note/bookmark | Member |
| PATCH | `/api/notes/:noteId` | Edit own note | Author |
| DELETE | `/api/notes/:noteId` | Delete own note | Author |
| GET | `/api/search?q=&courseId=` | Search step titles, checkpoint notes and my notes across lessons | Any |
| GET | `/api/github/oauth/start?sessionId=` | Redirect to GitHub OAuth (scope `public_repo`) | Any |
| GET | `/api/github/oauth/callback` | Exchange code → token held server-side for 10 min (not in MongoDB) → redirect back | Any |
| POST | `/api/sessions/:sessionId/export` | `{ repoName, private? }` → create repo + one commit per checkpoint → `{ repoUrl }` | Member with a fresh GitHub token |

### Reports & digests — `reports.routes.js` (Andric; sending via Karin's Telegram service)

| Method | Path | What happens | Who |
|---|---|---|---|
| GET | `/api/sessions/:sessionId/report` | Insights for one session | Staff |
| GET | `/api/lessons/:lessonId/trends` | Same lesson across sessions/terms: steps that trip students every time | Staff |
| POST | `/api/sessions/:sessionId/report/digest` | `{ audience: ['tas', 'absent'] }` → Telegram digest | Prof |

### Hints (COULD) — `hints.routes.js` (Karin)

| Method | Path | What happens | Who |
|---|---|---|---|
| POST | `/api/sessions/:sessionId/hints` | `{ stepId, errorKey }` → cached one-line hint (rule table first, LLM if enabled) | Enrolled (if `hintsEnabled`) |

### Test-only — `test.routes.js` (Andric; mounted only when `NODE_ENV=test`)

| Method | Path | What happens |
|---|---|---|
| POST | `/api/__test__/reset` | Drop DB + re-seed |
| GET | `/api/__test__/outbox` | Messages the fake Telegram/GitHub services recorded |

> **Route ordering:** static paths must be declared before parameter paths that could swallow them — e.g. `POST /api/courses/enrol` before `/api/courses/:courseId`, and `POST /api/sessions/join` before `/api/sessions/:sessionId`. Invalid ObjectIds are rejected by a `validateObjectId('courseId')` param middleware → 404, never a crash.

---

## Key Controller Logic

```
Start session  POST /api/lessons/:lessonId/sessions
  1. Load lesson; 404 if missing; 409 unless status === 'ready' and steps.length ≥ 1
  2. Load course; 403 unless caller is staff with role 'prof'
  3. Generate join code; retry on unique-index clash (max 5)
  4. Create Session { status: 'live' }
  5. Return { session, joinUrl: `${APP_URL}/join/${code}`, qrDataUrl }

Join session  POST /api/sessions/join
  1. findLiveByCode(joinCode) → 404 "No live class with that code"
  2. Caller enrolled → role 'student'; staff → role 'staff'; else 403 "You're not in this course"
  3. Return { sessionId, role }; client navigates to /s/:id/code or /s/:id/teach

Save checkpoint  POST /api/sessions/:sessionId/checkpoints
  1. Session live (409) + caller prof staff (403)
  2. stepId must be in lesson.steps (400)
  3. seq = nextSeq(); create (retry once on duplicate seq)
  4. setCurrentStep; dashboardCache.setTarget(sessionId, step.order) → re-classify all
  5. broadcast checkpoint:new to students + staff; 201

Catch up  POST /api/sessions/:sessionId/catch-up
  1. Session live + allowCatchUp (403 "Catch-up is turned off for this class")
  2. Latest checkpoint; 409 if none yet
  3. Create Snapshot { reason: 'before_catchup', files: req.body.files }
  4. incrementCatchUp; return { checkpoint, attemptSnapshotId }
  (Client then shows "What you missed" = diff(attempt, checkpoint) and loads checkpoint files)

Create help request  POST /api/sessions/:sessionId/help-requests
  1. Enrolled + session live
  2. Active request of same type exists → 409 "You already have a request waiting"
  3. Create Snapshot { reason: 'help_request' }, then HelpRequest { snapshotId }
  4. broadcast help:new to staff (name, step, type — no code)
  5. If inPerson: telegramService.pingOnDutyStaff(session, request) → set telegramNotifiedAt
     (Telegram failure is logged and shown to the student as "TA notified in app only" — never a 500)

End session  POST /api/sessions/:sessionId/end
  1. Prof staff; 409 if already ended
  2. endSession → broadcast session:ended → disconnect sockets from the room
  3. reportBuilder.build(sessionId) → SessionReport (async; report page polls until ready)
```

---

## Socket.IO Events

Owner: **Ying Qi** (transport, rooms, auth); each feature owner owns the payload of their events. Event names live in `shared/constants.js` as `SocketEvent.*`.

### Rooms

| Room | Members |
|---|---|
| `session:<id>:students` | Students who joined the session |
| `session:<id>:staff` | Prof + TAs viewing the session |
| `user:<id>` | Every socket of one user (multiple tabs/devices) |

### Client → server

| Event | Payload | Ack | Who | Notes |
|---|---|---|---|---|
| `session:join` | `{ sessionId }` | `{ ok, role, latestCheckpoint, myStatus?, settings }` | Member | Server re-checks membership; joins rooms |
| `session:leave` | `{ sessionId }` | — | Member | Also on disconnect |
| `status:update` | `{ sessionId, doneStepOrder, error: { name, message, file, line } \| null, checks? }` | `{ ok, state }` | Student | Sent **only when something changed**; server classifies |
| `heartbeat` | `{ sessionId, typing: boolean }` | — | Student | Every 30 s while the tab is visible |
| `lost:toggle` | `{ sessionId, lost: boolean }` | `{ ok }` | Student | Anonymous to staff |

### Server → client

| Event | Room | Payload | Owner |
|---|---|---|---|
| `checkpoint:new` | students + staff | `{ seq, stepId, stepOrder, title, createdAt }` (files fetched on demand) | Ying Qi |
| `session:settings` | students + staff | `{ allowCatchUp, hintsEnabled }` | Ying Qi |
| `session:ended` | students + staff | `{ sessionId, replayUrl }` | Ying Qi |
| `dashboard:tile` | staff | `{ studentId, name, state, stepOrder, errorKey, errorPreview, checks?, lastSeenAt }` — **no code** | Evan |
| `dashboard:summary` | staff (≤ 1 per second) | `{ counts: { done, in_progress, error, behind, idle }, lostCount, errorGroups: [{ key, preview, stepOrder, count }] }` | Evan |
| `spotlight:show` / `spotlight:hide` | students + staff | `{ excerpt, fixFiles?, errorPreview }` / `{}` | Evan |
| `help:new` / `help:updated` | staff | `{ requestId, studentName, stepOrder, type, status, inPerson, seat }` | Karin |
| `help:updated` | `user:<studentId>` | `{ requestId, status, reply? }` (a pass shows a tick on the student's step) | Karin |

### Connection lifecycle

```
connect (cookie JWT) → io.use(auth) → 401-style "unauthorized" error if invalid
  → client emits session:join → server validates → rooms joined → ack
disconnect → status unchanged; classifier marks idle after idleAfterSec without heartbeat
reconnect (Socket.IO automatic, backoff) → client re-emits session:join → re-sends its current status
```

---

## Component Breakdown (key screens)

| Component | Props | Emits | Notes |
|---|---|---|---|
| `CodeEditor` | `modelValue`, `language`, `errorLine`, `readOnly` | `update:modelValue`, `save` (Ctrl/⌘+S) | CodeMirror 6 wrapper; gutter marker on `errorLine` |
| `FileTabs` | `files`, `activePath` | `select` | Scrollable on xs |
| `PreviewFrame` | `files`, `entryFile`, `mockApi`, `runKey` | `console`, `runtime-error`, `network`, `ready` | Sandboxed iframe — [05-core-logic.md](./05-core-logic.md#1-sandboxed-preview) |
| `ConsolePanel` | `entries` | `clear` | Filters: All · Errors · Network |
| `DoneButton` | `stepOrder`, `disabled`, `passedByTa` | `done`, `undo` | Disabled while the last run has an error |
| `StudentTile` | `tile` | `select` | Colour + icon + label; shows step "4/8" and error preview |
| `TileGrid` | `tiles`, `filter` | `select` | 2 cols xs → 4 md → 6 xl; sorted error → behind → in progress → idle → done |
| `ClassSummaryBar` | `counts`, `lostCount`, `total` | `filter` | Clickable segments filter the grid |
| `ErrorGroupList` | `groups` | `spotlight` | Prof-only Spotlight button per group |
| `StepList` (prof) | `steps`, `currentStepId` | `reorder`, `edit`, `delete` | Drag handle on lg; up/down buttons on xs |
| `HelpRequestCard` | `request` | `claim`, `open` | Waiting time live-updates |
| `SideBySideView` | `studentFiles`, `checkpointFiles` | — | Uses Ray Yien's `DiffView` |
| `DiffView` | `before`, `after`, `mode: 'split' \| 'unified'` | — | Unified forced on xs |
| `CheckpointTimeline` | `checkpoints`, `activeSeq`, `bookmarks` | `select` | Keyboard ← → navigation |

---

## Error Handling (what the user sees)

| Situation | What the user sees | Handling |
|---|---|---|
| Form validation fails | Message under each field; values kept | 400 `error.fields` → form store |
| Wrong login | "Invalid email or password" (never says which) | 401 |
| Not logged in on a protected page | Redirect to `/login?returnTo=…` | Router guard + 401 interceptor |
| Not allowed (e.g. student opens `/s/:id/teach`) | Toast + redirect to `/courses` | Router guard (role) / 403 from API |
| Wrong join code | "No live class with that code" under the input | 404 |
| Session ended while coding | Banner "Class ended — your work is saved. Open replay" | `session:ended` event |
| Socket disconnected | Navbar pill turns amber "Reconnecting…"; edits continue locally; autosave retries | Socket.IO reconnect |
| Autosave fails | "Not saved — retrying" next to the file name; never blocks typing | Exponential retry; work kept in memory |
| Help already claimed by another TA | Toast "Ana is already on this one" and the card updates | 409 |
| Telegram / GitHub down | Feature-specific message ("Couldn't reach GitHub — try again in a minute"); rest of app works | Service catches, returns 502 with `code: 'UPSTREAM_UNAVAILABLE'` |
| Unknown route | Friendly 404 page | Vue `*` route; API `notFound` → 404 JSON |
| Unexpected server error | Toast "Something went wrong" + request id; no stack trace | `errorHandler` logs, returns 500 |
