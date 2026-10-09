# CodeAlong — Stage Gates & Test Plan

## Stage Gates

This is the progress tracker. Each stage has a one-line gate: **don't move on until the gate passes** (ideally as an automated test). Week numbers follow the IS216 schedule — progress pitch materials are due **Week 9 Monday 23:59**; final submission is **Week 12 Friday 9:00 am** (we aim for Thursday).

| # | Stage | Target | Gate (must be true before moving on) |
|---|---|---|---|
| 0 | Design lock | Week 8 | All nine design docs reviewed by all six members; open questions in [08-workload-split.md](./08-workload-split.md#open-questions) answered |
| 1 | Skeleton | Week 8 | `npm run dev` boots; `npm start` serves the built Vue app at `/`; MongoDB connects; one Playwright smoke test passes |
| 2 | Platform | Week 8 | Register/login/logout, course create + enrol, role guards work; seed loads |
| 3 | **Live session core (pitch demo)** | Week 9 Mon | Prof starts session → 3 students join by code → checkpoint reaches all 3 → a typo turns a tile red → student loads checkpoint → tile turns green |
| 4 | Lesson plan | Week 9 | Upload starter → TODO parser proposes steps → prof edits/reorders/deletes → lesson `ready` |
| 5 | Status & dashboard | Week 10 | Classifier states correct for all 5 states; idle sweep; filters; anonymous lost count |
| 6 | Catch-up | Week 10 | Load checkpoint keeps attempt; "What you missed" diff correct; restore attempt works |
| 7 | Help flow | Week 10 | Check my work → pass/reply; Ask for help → claim/resolve; Telegram ping (fake in tests) |
| 8 | Replay & GitHub | Week 11 | Replay steps through all checkpoints; notes/bookmarks; import + export (one commit per step) |
| 9 | Error groups & Spotlight | Week 11 | Grouping matches unit-test expectations; Spotlight shows an anonymised excerpt to everyone |
| 10 | Insights | Week 11 | Report generated on end; trouble score + trends; digest sent (fake in tests) |
| 11 | Hardening | Week 11–12 | Validation + error states everywhere; responsive pass 375 → 1440 px; load test 40 students passes |
| 12 | Demo ready | Week 12 | Fresh clone → README steps → seed → all E2E green → deployed URL works → video recorded |

### Pattern checks per stage (reviewer ticks these in the PR)

| Stage | Required pattern |
|---|---|
| 2 | Passwords bcrypt-hashed; JWT only in httpOnly cookie; `requireAuth`/`requireRole` on every protected route |
| 3 | REST for writes, sockets for broadcast; socket handshake re-checks membership; reconnect re-joins the room |
| 4 | Parser is a pure function with unit tests; prof reviews before saving |
| 5 | Client never sends `state` — server classifies; staff payloads contain no code |
| 6 | `before_catchup` snapshot is written **before** the checkpoint loads |
| 7 | Claim/pass/reply use conditional updates (409 on conflict); Telegram failure never fails the request |
| 8 | GitHub token never stored in MongoDB; import limited to safe file types and size |
| 9 | Spotlight only uses help-request snapshots; names removed |
| 10 | Report contains counts only |
| 11 | Every page usable at 375 px; every interactive element has a `data-testid` |

---

## Testing Strategy

| Layer | Tool | Scope | Why it matters for the rubric |
|---|---|---|---|
| **E2E** (main) | Playwright, Chromium | Every main user journey, with **multiple browser contexts in one test** (prof + TA + students) | "Tests cover the app's main user journeys" |
| Unit | Vitest | Pure logic: classifier, normaliser, TODO parser, diff summary, srcdoc builder, report builder, GitHub export (fake HTTP) | Shows understanding of testing concepts; fast feedback |
| Load | `scripts/simulate-class.js` | 40 socket clients in one session | Proves the live dashboard holds up in a real class |

### Making E2E repeatable and stable

| Rubric ask | How |
|---|---|
| Repeatable | Server runs on **mongodb-memory-server**; each spec file calls `POST /api/__test__/reset` (re-seed) in `beforeAll`; E2E runs with `workers: 1` |
| No external flakiness | `EXTERNAL_APIS=mock` swaps Telegram/GitHub/LLM for fakes; tests read the fake outbox via `GET /api/__test__/outbox` |
| Stable selectors | Only `getByTestId()` and `getByRole()` — never CSS classes or text that may change. Convention in [09-integration-contracts.md](./09-integration-contracts.md#9-data-testid-convention) |
| No arbitrary waits | `await expect(locator).toHaveText(...)` auto-waits; no `waitForTimeout` except the idle-state test, which shortens `idleAfterSec` via settings |
| Clear structure | One spec file per feature; `test.describe` per journey; Arrange → Act → Assert comments; shared fixtures |
| Responsive | Two Playwright projects: `desktop-chrome` (1280×800) and `mobile-chrome` (375×667, `isMobile`, `hasTouch`). Specs tagged `@responsive` run on both |

### Fixture sketch — `tests/e2e/fixtures.js`

```js
import { test as base, expect } from '@playwright/test'

async function loginContext(browser, email, password = 'pass1234') {
  const context = await browser.newContext()
  const res = await context.request.post('/api/auth/login', { data: { email, password } })
  expect(res.ok()).toBeTruthy()                 // login via API — it's plumbing, not under test here
  return { context, page: await context.newPage() }
}

export const test = base.extend({
  prof:     async ({ browser }, use) => { const s = await loginContext(browser, 'prof@codealong.test'); await use(s.page); await s.context.close() },
  ta:       async ({ browser }, use) => { const s = await loginContext(browser, 'ta.ana@codealong.test'); await use(s.page); await s.context.close() },
  students: async ({ browser }, use) => {
    const emails = ['alex@codealong.test', 'bea@codealong.test', 'chen@codealong.test']
    const sessions = await Promise.all(emails.map(e => loginContext(browser, e)))
    await use(sessions.map(s => s.page))
    await Promise.all(sessions.map(s => s.context.close()))
  },
})
export { expect }
```

### Example journey — `tests/e2e/live-session.spec.js`

```js
test('a typo turns the tile red and catching up turns it green', async ({ prof, students }) => {
  const [alex] = students
  // Arrange: prof starts the seeded Mini Cart lesson
  await prof.goto('/courses')
  await prof.getByTestId('course-card-IS216').click()
  await prof.getByTestId('lesson-start-mini-cart').click()
  const code = await prof.getByTestId('join-code').innerText()

  // Act: Alex joins, prof saves checkpoint for step 1, Alex introduces a typo
  await alex.goto(`/join/${code}`)
  await expect(alex.getByTestId('workspace-step-current')).toContainText('Step 1')
  await prof.getByTestId('btn-save-checkpoint').click()
  await alex.getByTestId('editor').click()
  await alex.keyboard.type('itmes.push(1)')
  await alex.keyboard.press('Control+S')

  // Assert: red on the prof's dashboard, with the grouped error
  const tile = prof.getByTestId(/tile-.*/).filter({ hasText: 'Alex' })
  await expect(tile).toHaveAttribute('data-state', 'error')

  // Act + Assert: catch up → green after Done
  await alex.getByTestId('btn-load-checkpoint').click()
  await expect(alex.getByTestId('what-you-missed')).toBeVisible()
  await alex.getByTestId('btn-done').click()
  await expect(tile).toHaveAttribute('data-state', 'done')
})
```

---

## Test Plan

**Layer:** E2E = Playwright · Unit = Vitest · Int = API-level test with Playwright `request` · Load = simulator. Each ID maps to the spec file in brackets.

### PLAT — Accounts, courses, roles (platform) · `platform.spec.js`

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-PLAT-01 | Register with valid data | E2E | Account created as `student`; lands on `/courses` |
| T-PLAT-02 | Register duplicate email / short password | E2E | 400 field errors shown; values kept |
| T-PLAT-03 | Login wrong password vs unknown email | Int | Both 401 with the same generic message |
| T-PLAT-04 | Protected page while logged out | E2E | Redirect to `/login?returnTo=…`; after login, back to that page |
| T-PLAT-05 | Prof creates course | E2E | Course card shows enrol code |
| T-PLAT-06 | Student enrols with code / wrong code | E2E | Course appears / "No course with that code" |
| T-PLAT-07 | Prof adds TA by email | E2E | TA sees the course; account role becomes `ta` |
| T-PLAT-08 | Student opens prof-only route | E2E | Toast + redirect; API returns 403 for the matching call |
| T-PLAT-09 | Password stored as bcrypt hash | Unit | `passwordHash` matches `/^\$2[aby]\$/`; plain text absent |
| T-PLAT-10 | Login rate limit | Int | 11th attempt in a minute → 429 |

### LESSON — Lessons & plan · `lessons.spec.js`, `todo-parser.test.js`

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-LES-01 | Upload starter files → steps proposed | E2E | Step list shows the TODOs in order |
| T-LES-02 | Edit step title/description | E2E | Saved; persists after reload |
| T-LES-03 | Reorder steps (buttons on mobile, drag on desktop) | E2E `@responsive` | Orders contiguous 1…n |
| T-LES-04 | Delete step | E2E | Remaining steps renumbered |
| T-LES-05 | Unsafe path / > 20 files / > 200 KB | Int | 400 with field error |
| T-LES-06 | Delete a lesson that was taught | Int | 409 → UI offers Archive |
| T-LES-07 | Student fetches lesson files outside a session | Int | Steps only, no files |
| T-TODO-01 | JS, HTML and CSS comment forms parsed | Unit | 3 steps with correct files/lines |
| T-TODO-02 | Numbered TODOs sorted by number across files | Unit | Order follows numbers, not file order |
| T-TODO-03 | Same number in two files → one step, two refs | Unit | `todoRefs.length === 2` |
| T-TODO-04 | Unnumbered TODOs appended in scan order | Unit | After numbered ones |
| T-TODO-05 | Gaps (1, 2, 5) → orders 1, 2, 3 | Unit | Original numbers noted |
| T-TODO-06 | No TODOs → single default step | Unit | "Complete the exercise" |
| T-TODO-07 | "TODO" inside a string literal | Unit | Ignored |

### SESS — Live session engine · `live-session.spec.js`

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-SESS-01 | Prof starts session | E2E | Join code (6 chars, no 0/O/1/I/L) + QR visible |
| T-SESS-02 | Start on a draft lesson | Int | 409 |
| T-SESS-03 | Student joins via code input and via `/join/:code` | E2E | Lands in workspace on Step 1 |
| T-SESS-04 | Non-enrolled user joins | E2E | "You're not in this course" |
| T-SESS-05 | Wrong / ended code | E2E | "No live class with that code" |
| T-SESS-06 | Checkpoint broadcast to 3 students | E2E | All 3 see the toast + new step within 2 s |
| T-SESS-07 | Student reconnects (go offline → online) | E2E | Pill shows Reconnecting… then Live; status re-sent; checkpoint missed while offline is shown |
| T-SESS-08 | Two quick checkpoint saves | Int | Distinct `seq` 1, 2 (no duplicate) |
| T-SESS-09 | Socket handshake without cookie | Int | Connection refused |
| T-SESS-10 | Student emits `session:join` for another course's session | Int | Ack `{ ok: false }`; not added to room |
| T-SESS-11 | Prof ends session | E2E | Students see "Class ended" banner with replay link |
| T-SESS-12 | Settings toggle (catch-up off) | E2E | Students' Load checkpoint button disabled with tooltip |

### SBX — Workspace & sandbox · `workspace.spec.js`, `srcdoc-builder.test.js`, `mock-api.test.js`

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-SBX-01 | Local CSS/JS inlined; CDN tags kept | Unit | Output contains inline code + original CDN URLs |
| T-SBX-02 | Probe injected first in `<head>` | Unit | Probe precedes all user scripts |
| T-SBX-03 | Runtime error shown in console, gutter and status pill | E2E | File + line match the typo |
| T-SBX-04 | `console.log` appears in console panel | E2E | Entry visible |
| T-SBX-05 | Preview cannot read parent cookies / DOM | E2E | `document.cookie` in preview is empty; `parent.document` access throws (logged) |
| T-SBX-06 | Message with wrong `runId` ignored | Unit | No state change |
| T-SBX-07 | Infinite loop | E2E | Iframe replaced after 5 s with the hint message |
| T-SBX-08 | axios to mocked route | E2E | Data renders; console shows `mocked` |
| T-SBX-09 | Mock filters array by query param | Unit | `?category=fruit` returns fruit only |
| T-SBX-10 | `localStorage` works across runs | E2E | Cart persists after Ctrl+S re-run |
| T-SBX-11 | Autosave + refresh | E2E | Edits restored after reload |
| T-SBX-12 | Done disabled while the last run has an error | E2E | Button disabled with reason |

### STAT — Status & classifier · `status-classifier.test.js`, `dashboard.spec.js`

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-STAT-01 | Before first checkpoint | Unit | `in_progress` (or `done` if step 1 Done) |
| T-STAT-02 | done ≥ target | Unit | `done` |
| T-STAT-03 | done = target − 1 | Unit | `in_progress` |
| T-STAT-04 | done ≤ target − 2 | Unit | `behind` |
| T-STAT-05 | Error overrides behind/done | Unit | `error` |
| T-STAT-06 | Idle overrides everything | Unit | `idle` |
| T-STAT-07 | New checkpoint re-classifies all | E2E | Previously green tiles turn amber |
| T-STAT-08 | Idle after `idleAfterSec` (set to 5 s in test) | E2E | Tile grey; returns on activity |
| T-STAT-09 | Client cannot force a state | Int | Sending `state: 'done'` in payload is ignored |

### DASH — Dashboard · `dashboard.spec.js`

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-DASH-01 | Initial snapshot loads for a running session | E2E | Tiles for all joined students |
| T-DASH-02 | Summary bar counts match tiles | E2E | Counts equal per state |
| T-DASH-03 | Filter by clicking a summary segment | E2E `@responsive` | Only that state's tiles shown |
| T-DASH-04 | "I'm lost" from 2 students | E2E | Lost count = 2; no tile shows who |
| T-DASH-05 | Student un-taps "I'm lost" | E2E | Count decrements |
| T-DASH-06 | TA (staff) can view dashboard; student cannot | E2E | TA sees tiles; student redirected / 403 |
| T-DASH-07 | Tiles show icon + label, not only colour | E2E | `data-state` + visible label text |

### ERRG — Error groups & Spotlight · `error-normalizer.test.js`, `spotlight.test.js`, `spotlight.spec.js`

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-ERRG-01 | Line/col and URLs stripped | Unit | Same `key` for two stacks differing only in line |
| T-ERRG-02 | Numbers masked | Unit | `status code <n>` |
| T-ERRG-03 | `items` vs `itmes is not defined` | Unit | Different `key`, same `shapeKey` |
| T-ERRG-04 | Group threshold (≥ 2 or ≥ 10%) | Unit | Singletons excluded |
| T-ERRG-05 | Same typo by 2 students | E2E | One group "×2" on the dashboard |
| T-ERRG-06 | Spotlight with an eligible help snapshot | E2E | All students see excerpt + fix |
| T-ERRG-07 | Excerpt anonymised | Unit | No student names, no comments, ≤ 30 lines |
| T-ERRG-08 | Spotlight with no help snapshot | E2E | Error + fix only, with explanation to prof |

### CATCH — Catch-up · `catch-up.spec.js`, `diff-summary.test.js`

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-CATCH-01 | Load checkpoint | E2E | Editor shows checkpoint code; "What you missed" open |
| T-CATCH-02 | Attempt preserved | Int | `before_catchup` snapshot exists **before** files are returned |
| T-CATCH-03 | Restore attempt | E2E | Original code back in the editor |
| T-CATCH-04 | First-difference line correct | Unit | Points at the typo line |
| T-CATCH-05 | Whitespace-only changes ignored | Unit | No hunks |
| T-CATCH-06 | Catch-up before any checkpoint | Int | 409 with message |
| T-CATCH-07 | "What you missed" readable at 375 px | E2E `@responsive` | Unified diff mode, no horizontal page scroll |

### HELP — Help flow · `help.spec.js`

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-HELP-01 | Student sends Check my work | E2E | TA queue shows card instantly (name, step, type — no code) |
| T-HELP-02 | TA claims → side-by-side view | E2E | Student code vs checkpoint for that step |
| T-HELP-03 | Pass | E2E | Student sees tick on step; tile shows passed |
| T-HELP-04 | Reply | E2E | Student sees reply in their help panel |
| T-HELP-05 | Two TAs claim the same request | E2E | Second gets "already on this one" |
| T-HELP-06 | Duplicate active request of same type | Int | 409 |
| T-HELP-07 | Student cancels open request | E2E | Card disappears from queue |
| T-HELP-08 | In-person help → Telegram ping | E2E | Fake outbox has message with name, seat, step, link |
| T-HELP-09 | Telegram fake set to fail | E2E | Request still created; student sees "notified in app only" |
| T-HELP-10 | Another student opens `/help/:id` | Int | 403 |
| T-HELP-11 | Pass on a `help` (not `check`) request | Int | 400 |

### TG — Telegram linking & digests · `telegram.test.js`, `insights.spec.js`

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-TG-01 | Link code → `/start <code>` update binds chatId | Unit | `telegram.chatId` set; code cleared |
| T-TG-02 | Expired code | Unit | Not bound; bot replies "code expired" |
| T-TG-03 | User text HTML-escaped | Unit | `<script>` sent as `&lt;script&gt;` |
| T-TG-04 | 429 handling | Unit | Waits `retry_after`, retries once |

### REPLAY — Replay, notes, search · `replay.spec.js`

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-REP-01 | Replay lists checkpoints in order | E2E | Step titles 1…n |
| T-REP-02 | Changed lines highlighted vs previous checkpoint | E2E | Highlight count matches diff |
| T-REP-03 | Read-only preview runs | E2E | Preview renders the app at that step |
| T-REP-04 | Add / edit / delete note and bookmark | E2E | Persist after reload; private to author |
| T-REP-05 | Search finds a step title and a note | E2E | Both in results with links |
| T-REP-06 | Search with regex characters | Int | No crash; literal match |
| T-REP-07 | Non-member opens replay | Int | 403 |

### GH — GitHub import/export · `github-export.test.js`, `replay.spec.js`

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-GH-01 | Import public repo (fake) | E2E | Draft lesson with filtered files + steps |
| T-GH-02 | Import filters unsafe types / > 20 files | Unit | Only allowed files kept |
| T-GH-03 | Export creates one commit per checkpoint | Unit | Fake records 1 repo + n trees + n commits + 1 ref update, in order |
| T-GH-04 | Commit messages `Step k: <title>` | Unit | Match |
| T-GH-05 | Repo name taken (422) | E2E | UI asks for another name |
| T-GH-06 | Token not persisted | Int | No GitHub token in any MongoDB document after export |

### INS — Insights · `report-builder.test.js`, `insights.spec.js`

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-INS-01 | `reachedDone` and `dropOff` from fixture histories | Unit | Match hand-computed values |
| T-INS-02 | Trouble score formula | Unit | Matches expected ranking |
| T-INS-03 | Recurring trouble across 2 seeded past sessions | Unit | Step flagged |
| T-INS-04 | Report generated on end | E2E | Report page shows funnel + top errors |
| T-INS-05 | Report has no per-student data | Int | No `studentId` fields in payload |
| T-INS-06 | Digest to TAs and absent students | E2E | Fake outbox: TA summary + absent-student messages only to linked, absent students |

### PRIV — Privacy by design · `privacy.spec.js`

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-PRIV-01 | `dashboard:tile` / `dashboard:summary` payloads | Int | No `files` / code fields |
| T-PRIV-02 | Staff fetches a student's autosave | Int | No route exists → 404 |
| T-PRIV-03 | Staff fetches a snapshot through a help request | Int | Allowed (student sent it) |
| T-PRIV-04 | Lost identity never sent | Int | No payload contains which student is lost |
| T-PRIV-05 | Notes private | Int | Another user's note → 404 |

### RESP — Responsive (Chrome) · tagged `@responsive`

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-RESP-01 | Workspace at 375 px | E2E | Tabs Code/Preview/Console/Steps; sticky bottom bar; no horizontal scroll |
| T-RESP-02 | Workspace at 1280 px | E2E | Three panes visible |
| T-RESP-03 | Teach view at 375 px | E2E | Dashboard-first layout; editor behind tab |
| T-RESP-04 | Teach view at 1280 px | E2E | Editor + plan + tile grid visible together |
| T-RESP-05 | No page has horizontal scroll at 375 px | E2E | `scrollWidth <= innerWidth` on every route |
| T-RESP-06 | Landscape 667×375 workspace | E2E | Usable; bottom bar doesn't cover editor |

### VALID — Validation & error handling · `errors.spec.js`

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-VAL-01 | Invalid ObjectId in any route | Int | 404 JSON, no crash |
| T-VAL-02 | Unknown API route | Int | 404 JSON envelope |
| T-VAL-03 | Unknown client route | E2E | Friendly 404 page |
| T-VAL-04 | Forced server error | Int | 500 envelope with request id, no stack trace |
| T-VAL-05 | XSS in step title / note / reply | E2E | Rendered as text (Vue escaping); never executed |
| T-VAL-06 | Oversized payload | Int | 413 / 400 |
| T-VAL-07 | Upstream (GitHub) down | E2E | Feature message; rest of app works |

### LOAD & SEED

| ID | Test | Layer | Expected |
|---|---|:---:|---|
| T-LOAD-01 | 40 simulated students, update every 3 s, 5 minutes | Load | p95 tile latency < 500 ms; no dropped sockets |
| T-LOAD-02 | 40 students receive a checkpoint | Load | All acked within 2 s |
| T-SEED-01 | `npm run seed` twice | Int | Same counts both times (idempotent) |
| T-SEED-02 | Seeded accounts log in with documented passwords | Int | All roles correct |
| T-SEED-03 | Seeded past session has replay + report | E2E | Both pages render |
| T-SMOKE-01 | `/` loads landing in production build | E2E | Title + "Join a class" visible |

---

## Test Summary

| Area | Count | Main stage |
|---|---:|---|
| PLAT | 10 | 2 |
| LESSON + TODO | 14 | 4 |
| SESS | 12 | 3 |
| SBX | 12 | 3 |
| STAT | 9 | 5 |
| DASH | 7 | 5 |
| ERRG | 8 | 9 |
| CATCH | 7 | 6 |
| HELP | 11 | 7 |
| TG | 4 | 7 |
| REPLAY | 7 | 8 |
| GH | 6 | 8 |
| INS | 6 | 10 |
| PRIV | 5 | 5–9 |
| RESP | 6 | 11 |
| VALID | 7 | 11 |
| LOAD + SEED + SMOKE | 6 | 11–12 |
| **Total** | **137** | |

**Must be green for the video:** every E2E test tagged on a MUST feature (PLAT, LESSON, SESS, SBX, STAT, DASH, CATCH, HELP, REPLAY, PRIV, RESP, VALID).

### How to run (goes in the README)

```bash
npm install
npx playwright install chromium
npm run test:unit                    # Vitest — ~5 s
npm run test:e2e                     # builds client, starts test server (in-memory DB, mocked APIs), runs Playwright
npx playwright show-report           # HTML report with traces for any failure
npm run simulate -- --session ABC234 --students 40   # load test against a running session
```
