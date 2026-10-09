# CodeAlong — Integration Contracts (Shared)

This is the **single source of truth** for anything two or more people depend on. If a task card or another doc disagrees with this file, **this file wins**. Changing anything here needs a message in the team channel and a PR reviewed by everyone it affects.

---

## 1. Shared Constants — `shared/constants.js`

Imported by both client and server. Never hard-code these strings elsewhere.

```js
export const Role = Object.freeze({ PROF: 'prof', TA: 'ta', STUDENT: 'student' })
export const CourseRole = Object.freeze({ PROF: 'prof', TA: 'ta' })

export const LessonStatus = Object.freeze({ DRAFT: 'draft', READY: 'ready', ARCHIVED: 'archived' })
export const SessionStatus = Object.freeze({ LIVE: 'live', ENDED: 'ended' })

export const TileState = Object.freeze({
  DONE: 'done', IN_PROGRESS: 'in_progress', ERROR: 'error', BEHIND: 'behind', IDLE: 'idle',
})
export const TileStateOrder = [TileState.ERROR, TileState.BEHIND, TileState.IN_PROGRESS, TileState.IDLE, TileState.DONE] // grid sort

export const SnapshotReason = Object.freeze({ AUTOSAVE: 'autosave', BEFORE_CATCHUP: 'before_catchup', HELP_REQUEST: 'help_request' })

export const HelpType = Object.freeze({ CHECK: 'check', HELP: 'help' })
export const HelpStatus = Object.freeze({
  OPEN: 'open', CLAIMED: 'claimed', PASSED: 'passed', REPLIED: 'replied', RESOLVED: 'resolved', CANCELLED: 'cancelled',
})

export const NoteKind = Object.freeze({ NOTE: 'note', BOOKMARK: 'bookmark' })

export const Limits = Object.freeze({
  MAX_FILES: 20,
  MAX_SNAPSHOT_BYTES: 200_000,
  JOIN_CODE_LENGTH: 6,
  JOIN_CODE_ALPHABET: 'ABCDEFGHJKMNPQRSTUVWXYZ23456789',
  HEARTBEAT_MS: 30_000,
  IDLE_AFTER_SEC_DEFAULT: 180,
  SUMMARY_THROTTLE_MS: 1_000,
  STATUS_WRITE_THROTTLE_MS: 2_000,
  AUTOSAVE_DEBOUNCE_MS: 2_000,
  RUN_DEBOUNCE_MS: 1_500,
  RUN_TIMEOUT_MS: 5_000,
  ERROR_MESSAGE_MAX: 300,
  ERROR_GROUP_MIN: 2,
})

export const SocketEvent = Object.freeze({
  // client → server
  SESSION_JOIN: 'session:join',
  SESSION_LEAVE: 'session:leave',
  STATUS_UPDATE: 'status:update',
  HEARTBEAT: 'heartbeat',
  LOST_TOGGLE: 'lost:toggle',
  // server → client
  CHECKPOINT_NEW: 'checkpoint:new',
  SESSION_SETTINGS: 'session:settings',
  SESSION_ENDED: 'session:ended',
  DASHBOARD_TILE: 'dashboard:tile',
  DASHBOARD_SUMMARY: 'dashboard:summary',
  SPOTLIGHT_SHOW: 'spotlight:show',
  SPOTLIGHT_HIDE: 'spotlight:hide',
  HELP_NEW: 'help:new',
  HELP_UPDATED: 'help:updated',
})
```

---

## 2. Auth

### Cookie + `req.user`

- Login/register set an httpOnly cookie `ca_token` (JWT `{ sub, role, name }`), `sameSite: 'lax'`, `secure` in production.
- `requireAuth` verifies it and sets:

```js
req.user = { _id: '6507…', role: 'prof' | 'ta' | 'student', name: 'Prof Lee' }
```

- The client never reads the token. `GET /api/auth/me` on app start fills `authStore.user`; a 401 anywhere clears it and redirects to `/login?returnTo=…`.

### Socket handshake

```js
// server/sockets/index.js
io.use((socket, next) => {
  const token = parseCookie(socket.handshake.headers.cookie || '').ca_token
  try { socket.user = verifyJwt(token); next() }
  catch { next(new Error('unauthorized')) }
})
```

Every socket event handler must check membership again for the `sessionId` in the payload — **never** trust that a socket is in a room just because it asked.

---

## 3. API Response Envelope

```js
// success
res.status(200).json({ data: <object | array> })

// list with paging (only where needed)
res.json({ data: [...], meta: { total, page, pageSize } })

// failure (from errorHandler)
{ "error": { "code": "VALIDATION_FAILED", "message": "Please fix the highlighted fields", "fields": { "email": "Enter a valid email" }, "requestId": "r_8f2c…" } }
```

| HTTP | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | zod failed; `fields` present |
| 400 | `BAD_REQUEST` | Valid shape, invalid action (e.g. pass on a `help` request) |
| 401 | `UNAUTHENTICATED` | No/expired cookie; wrong login (generic message) |
| 403 | `FORBIDDEN` | Not a member / wrong role / not the owner |
| 404 | `NOT_FOUND` | Unknown route, invalid or missing ObjectId |
| 409 | `CONFLICT` | State conflict: already claimed, session ended, duplicate active request, lesson already taught |
| 413 | `PAYLOAD_TOO_LARGE` | Files over `MAX_SNAPSHOT_BYTES` |
| 429 | `RATE_LIMITED` | Login rate limit |
| 502 | `UPSTREAM_UNAVAILABLE` | Telegram/GitHub/LLM failed |
| 500 | `INTERNAL` | Anything else (logged with `requestId`; no stack trace to the client) |

```js
// server/middleware/errors.js
export class AppError extends Error {
  constructor(status, code, message, fields) { super(message); Object.assign(this, { status, code, fields }) }
}
export const notFound = (req, res, next) => next(new AppError(404, 'NOT_FOUND', 'Not found'))
export const errorHandler = (err, req, res, next) => {
  const status = err.status ?? 500
  if (status === 500) console.error(req.id, err)
  res.status(status).json({ error: {
    code: err.code ?? 'INTERNAL',
    message: status === 500 ? 'Something went wrong' : err.message,
    ...(err.fields && { fields: err.fields }),
    requestId: req.id,
  } })
}
```

Client: `api/http.js` unwraps `data` and throws `{ status, code, message, fields }` so stores and forms handle errors one way.

---

## 4. Authorization Layering

Two layers, always both:

1. **Route middleware** — `requireAuth`, plus `requireRole(Role.PROF)` where only profs ever qualify (e.g. create course).
2. **Controller** — load the resource, then check **course membership**:

```js
// server/middleware/auth.js
export async function loadMembership(userId, courseId) {
  const course = await Course.findById(courseId).lean()
  if (!course) throw new AppError(404, 'NOT_FOUND', 'Course not found')
  const staff = course.staff.find(s => String(s.userId) === String(userId))
  const enrolled = course.students.some(id => String(id) === String(userId))
  return { course, isStaff: !!staff, staffRole: staff?.role ?? null, isEnrolled: enrolled,
           isOwner: String(course.ownerId) === String(userId) }
}

// in a controller
const m = await loadMembership(req.user._id, session.courseId)
if (!m.isStaff) throw new AppError(403, 'FORBIDDEN', 'Only course staff can view the dashboard')
```

| Check | Helper condition |
|---|---|
| Prof-only course actions | `m.isOwner` (course edit/delete/add TA) or `m.staffRole === 'prof'` (lessons, sessions, checkpoints, Spotlight) |
| Staff actions | `m.isStaff` |
| Student actions | `m.isEnrolled` |
| Viewer actions (replay, library) | `m.isStaff \|\| m.isEnrolled` |

---

## 5. Privacy Contract (our unique feature — every PR is checked against this)

| Rule | Enforced where |
|---|---|
| Staff payloads (`dashboard:*`, `help:new`, queue list, report) **never contain files or code** | Serializers in Evan's/Karin's/Andric's controllers; T-PRIV-01 |
| A student's code reaches staff **only** via `GET /api/help-requests/:id` (snapshot the student sent) or an anonymised Spotlight excerpt | Karin, Evan; T-PRIV-02/03 |
| There is **no** endpoint that returns another user's autosave or `before_catchup` snapshot | Dylan; T-PRIV-02 |
| "I'm lost" is only ever sent as a **count** | Ying Qi/Evan; T-PRIV-04 |
| Notes are private to the author | Ray Yien; T-PRIV-05 |
| Reports contain counts only, no `studentId` | Andric; T-INS-05 |
| No keystroke logging — only `status:update` on change + heartbeat | Dylan |

---

## 6. Socket Payloads

Shapes are fixed; add optional fields only, never rename.

```js
// client → server
'session:join'  { sessionId }                       → ack { ok, role, latestCheckpoint: { seq, stepId, stepOrder, title } | null, myStatus, settings }
'status:update' { sessionId, doneStepOrder, error: { name, message, file, line } | null, checks?: { passed, total } } → ack { ok, state }
'heartbeat'     { sessionId, typing }
'lost:toggle'   { sessionId, lost }                 → ack { ok }

// server → client
'checkpoint:new'    { sessionId, seq, stepId, stepOrder, title, createdAt }
'session:settings'  { sessionId, allowCatchUp, hintsEnabled }
'session:ended'     { sessionId, replayUrl }
'dashboard:tile'    { sessionId, studentId, name, state, stepOrder, errorKey, errorPreview, passedSteps: [stepOrder], checks?, lastSeenAt }
'dashboard:summary' { sessionId, counts: { done, in_progress, error, behind, idle }, total, lostCount,
                      errorGroups: [{ key, shapeKey, preview, stepOrder, count, variants: [{ key, count }] }] }
'spotlight:show'    { sessionId, errorPreview, excerpt: { path, startLine, code }, fix: { path, startLine, code } }
'spotlight:hide'    { sessionId }
'help:new'          { sessionId, requestId, studentName, stepOrder, type, inPerson, seat, createdAt }
'help:updated'      { sessionId, requestId, status, claimedByName?, reply? }   // to staff room and to user:<studentId>
```

Acks with `{ ok: false, code, message }` use the same `code` values as the REST envelope.

---

## 7. Cross-Owner Function Signatures

| Function | Owner | Called by | Signature |
|---|---|---|---|
| `broadcast.toStudents` / `toStaff` / `toUser` | Ying Qi | Evan, Karin, Dylan | `(sessionId \| userId, event, payload) => void` |
| `dashboardCache.setTarget` | Ying Qi | Checkpoints controller | `(sessionId, stepOrder) => void` |
| `dashboardCache.getSnapshot` | Ying Qi | Evan (dashboard endpoint) | `(sessionId) => { tiles, summary }` |
| `normalizeError` | Evan | Ying Qi (status pipeline) | `({ name, message }) => { key, shapeKey }` |
| `groupErrors` | Evan | Ying Qi (summary) | `(tiles, { min, presentCount }) => errorGroups[]` |
| `parseTodos` | Evan | Ray Yien (import), Andric (seed) | `(files, entryFile) => steps[]` |
| `diffSummary` | Ray Yien | Dylan, Karin | `(beforeFiles, afterFiles) => { files: [{ path, added, removed, firstDiffLine, hunks }] }` |
| `<DiffView>` | Ray Yien | Dylan, Karin | props `before`, `after`, `mode` |
| `githubService.importRepo` | Ray Yien | Lessons controller (`importFromGithub`) | `(repoUrl) => Promise<{ files, entryFile, source }>` |
| `telegramService.notify` | Karin | Help controller | `(chatIds[], html) => Promise<{ sent, failed }>` |
| `telegramService.sendDigest` | Karin | Andric (reports) | `(sessionReport, audience) => Promise<{ sent, failed }>` |
| `reportBuilder.build` | Andric | Sessions controller (end), seed | `(sessionId) => Promise<SessionReport>` |
| `loadMembership` | Andric | Every controller | see §4 |

---

## 8. Route Ordering & Params

- Static paths before parameter paths: `POST /api/courses/enrol` before `/api/courses/:courseId`; `POST /api/sessions/join` before `/api/sessions/:sessionId`.
- Every `:xxxId` param passes `validateObjectId` → 404 on bad format.
- Vue Router: `/join` and `/join/:code` are separate records; `*` 404 route last.
- Express: `/api/*` routes → `/api` 404 → `express.static(client/dist)` → SPA fallback (`index.html`) for any other GET → `errorHandler`.

---

## 9. `data-testid` Convention

Format: `<area>-<element>[-<id>]`, kebab-case, stable across text/style changes.

| Area | Examples |
|---|---|
| Navigation | `nav-courses`, `nav-replays`, `nav-profile`, `nav-connection-pill` |
| Courses | `course-card-<code>`, `btn-new-course`, `input-enrol-code` |
| Lessons | `lesson-row-<slug>`, `lesson-start-<slug>`, `step-item-<order>`, `btn-step-up-<order>` |
| Session | `join-code`, `join-qr`, `input-join-code`, `btn-save-checkpoint`, `btn-end-session`, `toast-checkpoint` |
| Workspace | `editor`, `file-tab-<path>`, `preview-frame`, `console-entry`, `status-pill`, `btn-done`, `btn-lost`, `btn-help`, `btn-load-checkpoint`, `what-you-missed`, `workspace-step-current`, `tab-code`, `tab-preview`, `tab-console`, `tab-steps` |
| Dashboard | `tile-<studentId>` (with `data-state="<TileState>"`), `summary-<state>`, `lost-count`, `error-group-<index>`, `btn-spotlight-<index>`, `spotlight-panel` |
| Help | `help-card-<requestId>`, `btn-claim`, `btn-pass`, `btn-reply`, `input-reply`, `help-status` |
| Replay | `timeline-step-<seq>`, `diff-view`, `btn-add-note`, `note-item`, `input-search`, `btn-export-github` |
| Reports | `report-funnel`, `report-top-errors`, `btn-send-digest` |

Tiles also expose state with `data-state`, so tests assert state without depending on colours or label text.

---

## 10. Environment & Ports

| Variable / port | Value | Notes |
|---|---|---|
| Server port | 3000 | API + Socket.IO + built client in production |
| Vite dev port | 5173 | Proxies `/api` and `/socket.io` (with `ws: true`) to 3000 |
| `EXTERNAL_APIS` | `live` \| `mock` | Tests and anyone without keys use `mock` |
| `TELEGRAM_POLLING` | `true` only on one running instance | Two pollers on one token conflict |
| Test DB | mongodb-memory-server | Started by `npm run test:e2e`; never Atlas |

Full list in [02-tech-stack.md](./02-tech-stack.md#environment-variables-serverenv).

---

## 11. Definition of Done (every PR)

- [ ] Follows the pattern in [08-workload-split.md](./08-workload-split.md#non-negotiable-implementation-pattern)
- [ ] Privacy contract (§5) respected
- [ ] Uses constants from `shared/constants.js`
- [ ] Validation + loading + empty + error states in the UI
- [ ] Works at 375 px and 1280 px (screenshots in the PR)
- [ ] `data-testid` on interactive elements
- [ ] Tests for the feature added and passing locally; CI green
- [ ] Third-party snippets credited in a code comment
- [ ] Any AI-assisted part noted for the README

---

## Integration Checklist (before Week 12 sign-off)

- [ ] Landing page loads at `/` on the production build and the deployed URL
- [ ] Register → enrol → join a live session works end to end
- [ ] Prof imports/uploads a lesson, edits steps, starts a session
- [ ] Checkpoint reaches every student; reconnect recovers missed checkpoints
- [ ] Runtime error turns a tile red with the right file/line; Done turns it green
- [ ] Idle and behind states appear at the right times
- [ ] "I'm lost" count correct and anonymous
- [ ] Load checkpoint keeps the attempt; "What you missed" points at the right line
- [ ] Check my work → pass/reply; Ask for help → Telegram ping → resolve
- [ ] Error groups merge the same mistake; Spotlight is anonymised
- [ ] Ending a session shows the replay link; replay, notes, search work
- [ ] GitHub export creates one commit per checkpoint
- [ ] Insights report + digest correct (counts only)
- [ ] No staff payload contains student code (T-PRIV-*)
- [ ] Invalid IDs → 404, unknown routes → 404 page, server errors → friendly 500
- [ ] Every page usable 375 → 1440 px, portrait and landscape, in Chrome
- [ ] Load test: 40 simulated students within latency target
- [ ] `npm run seed` repeatable; `npm run test:e2e` green from a fresh clone
