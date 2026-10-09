# CodeAlong — Core Logic & Algorithms

This is the "how it actually works" document: the parts that make CodeAlong a Tier 2/3 project rather than a CRUD app. Each section names its owner and the unit-test file that pins it down. All pure logic lives in `server/lib/` or `client/src/sandbox/` so it can be tested without a browser or database.

| # | Logic | Owner | Tier | Tested in |
|---|---|---|:---:|---|
| 1 | Sandboxed preview + probe | Dylan | 2–3 | `tests/unit/srcdoc-builder.test.js`, E2E `workspace.spec.js` |
| 2 | Mock API inside the sandbox | Dylan | 2 | `tests/unit/mock-api.test.js` |
| 3 | Status reporting + classifier | Ying Qi | 3 | `tests/unit/status-classifier.test.js` |
| 4 | Error normalisation + grouping | Evan | **3** | `tests/unit/error-normalizer.test.js` |
| 5 | TODO parser → lesson plan | Evan | 2–3 | `tests/unit/todo-parser.test.js` |
| 6 | Catch-up + "What you missed" | Dylan (load) · Ray Yien (diff) | 2–3 | `tests/unit/diff-summary.test.js`, E2E `catch-up.spec.js` |
| 7 | Help queue + Telegram ping | Karin | 2 | E2E `help.spec.js` |
| 8 | Spotlight anonymisation | Evan | 2 | `tests/unit/spotlight.test.js` |
| 9 | Replay + GitHub import/export | Ray Yien | 2 | `tests/unit/github-export.test.js` (fake API), E2E `replay.spec.js` |
| 10 | Insights report + trends | Andric | 2–3 | `tests/unit/report-builder.test.js` |
| 11 | Telegram bot integration | Karin | 2 | `tests/unit/telegram.test.js` (fake API) |
| 12 | Stretch: auto step checks, AI hints, VS Code reporter | Dylan · Karin | 3 | — |

---

## 1. Sandboxed Preview

**Goal:** run a student's HTML/CSS/JS (incl. Vue from CDN) in their own browser, capture every error and console line, and make sure their code can't touch CodeAlong itself.

### How a run works

```
files + entryFile + mockApi
        │
        ▼
buildSrcdoc()                       ← client/src/sandbox/srcdoc-builder.js
  1. Start from entryFile (index.html)
  2. Inline local <link href="x.css">  → <style>/* x.css */ …</style>
  3. Inline local <script src="x.js">  → <script>… //# sourceURL=x.js</script>
     (CDN URLs are left as-is)
  4. Inject at the top of <head>:
       <script>window.__CA__ = { runId, mockApi, storage }</script>
       <script> probe.js </script>
        │
        ▼
<iframe sandbox="allow-scripts allow-modals allow-forms" srcdoc="…">
        │   (no allow-same-origin → opaque origin: no access to our cookies,
        │    our DOM, or our API)
        ▼
probe.js → window.parent.postMessage({ source: 'ca-probe', runId, type, payload }, '*')
        │
        ▼
PreviewFrame.vue: accept only if event.source === iframe.contentWindow
                  AND data.runId === currentRunId
```

### What the probe captures

| Type | Hook | Sent as |
|---|---|---|
| `console` | Wraps `console.log/info/warn/error` (keeps original output) | `{ level, args: safeSerialize(args) }` |
| `runtime-error` | `window.addEventListener('error')`, `console.error(errorObject)` | `{ name, message, file, line }` — file/line parsed from `error.stack` (Chrome honours `//# sourceURL`) |
| `rejection` | `unhandledrejection` | Same shape as runtime-error |
| `vue-warn` | `console.warn` starting with `[Vue warn]` | Shown in console as a warning, **not** an error |
| `network` | Wrapped `fetch` + `XMLHttpRequest` (axios uses XHR) | `{ method, url, status, mocked }` |
| `ready` | `DOMContentLoaded` | `{}` |

`safeSerialize` handles circular objects, DOM nodes and truncates to 2 KB per entry.

### `localStorage` shim

A sandboxed iframe without `allow-same-origin` throws a `SecurityError` on `localStorage`, but lessons like Mini Cart use it. The probe installs an in-memory `Storage` replacement (`Object.defineProperty(window, 'localStorage', …)`) seeded from `__CA__.storage`; every write posts a `storage` message so the parent keeps the values for the next run of that session. **Spike this in Week 8** — it's the riskiest browser detail.

### Re-run policy

- A run is triggered by Ctrl/⌘+S, the Run button, or 1.5 s after typing stops (debounced).
- Each run gets a fresh `runId`; messages from older runs are ignored.
- Infinite loops: if no `ready` message arrives within 5 s, the parent replaces the iframe and shows "Your code didn't finish running — check for an infinite loop".

**Limitation (documented in the README):** ES-module `import` between local files isn't supported in the baseline; lessons use `<script>` tags and the Vue global build from a CDN, as IS216 lessons already do.

---

## 2. Mock API

Lessons that call an Express backend (e.g. `axios.get('http://localhost:3000/items?category=fruit')`) must work without each student running a server.

- The prof defines `lesson.mockApi`: `[{ method: 'GET', path: '/items', status: 200, body: [...] }]`.
- Paths may contain params (`/items/:id`) and the body may be filtered by query using a small rule: if the body is an array and the request has `?key=value`, return items where `item[key] == value`.
- The probe intercepts any request whose path matches a mock route **regardless of host** (`localhost:3000`, relative `/api/...`), responds after a 150 ms delay (so loading states are visible), and reports `{ mocked: true }`.
- Unmatched requests pass through to the real network (CDNs, public APIs). Unmatched requests to `localhost` get a mocked **404** with a helpful message instead of a CORS error.

---

## 3. Status Reporting & Classifier

### Client side (Dylan)

The workspace keeps `{ doneStepOrder, lastError }` and emits `status:update` **only when one of them changes**. Heartbeat every 30 s while the tab is visible (`document.visibilityState`). No keystrokes, no code.

### Server side (Ying Qi) — `server/lib/status-classifier.js`

`targetOrder` = step order of the **latest checkpoint** (0 before the first checkpoint).

```js
export function classify({ doneStepOrder, error, lastSeenAt, targetOrder, now, idleAfterMs }) {
  if (now - lastSeenAt > idleAfterMs) return TileState.IDLE          // grey
  if (error)                          return TileState.ERROR         // red
  if (targetOrder === 0)              return doneStepOrder >= 1 ? TileState.DONE : TileState.IN_PROGRESS
  if (doneStepOrder >= targetOrder)   return TileState.DONE          // green
  if (doneStepOrder === targetOrder - 1) return TileState.IN_PROGRESS // amber — on the current step
  return TileState.BEHIND                                            // blue — 2+ steps behind
}
```

| Precedence | State | Colour | Icon | Meaning |
|:---:|---|---|---|---|
| 1 | `idle` | grey | `bi-moon` | No heartbeat or event for `idleAfterSec` (default 180 s) |
| 2 | `error` | red | `bi-x-octagon` | Last run threw — the most actionable state |
| 3 | `behind` | blue | `bi-hourglass` | Two or more steps behind the latest checkpoint |
| 4 | `in_progress` | amber | `bi-pencil` | Working on the step the prof just finished |
| 5 | `done` | green | `bi-check-circle` | Marked Done on the latest checkpoint's step |

### When re-classification happens

| Trigger | Who is re-classified |
|---|---|
| `status:update` from a student | That student |
| New checkpoint (`targetOrder` changes) | Everyone in the session |
| Idle sweep (every 15 s) | Students whose `lastSeenAt` crossed the threshold |
| Settings change (`idleAfterSec`) | Everyone |

### Dashboard cache — `server/lib/dashboard-cache.js`

An in-memory `Map<sessionId, { targetOrder, students: Map<studentId, status> }>`, rebuilt from MongoDB on the first access after a restart.

- Tile changes are emitted immediately (`dashboard:tile`).
- Summary (`counts`, `lostCount`, `errorGroups`) is recomputed and emitted at most once per second (`dashboard:summary`).
- DB writes are buffered: at most one `StudentStatus` upsert per student every 2 s (latest wins), plus a history entry whenever the **state** changes.
- Assumes a single server instance (fine for one class of 40). Scaling out would need the Socket.IO Redis adapter — out of scope.

**Load target:** 40 simulated students sending an update every 3 s → p95 tile latency < 500 ms on the deployed server (checked with `scripts/simulate-class.js`).

---

## 4. Error Normalisation & Grouping (signature algorithm)

**Goal:** if 13 students hit "the same mistake", the prof sees **one** group of 13, not 13 different messages. Messages differ in volatile parts (line numbers, URLs, values), so we normalise before grouping.

### `normalizeError()` — `server/lib/error-normalizer.js`

```js
export function normalizeError({ name = 'Error', message = '' }) {
  let m = String(message).split('\n')[0].trim()
  m = m
    .replace(/(?:https?:\/\/|blob:|about:)\S+/g, '<url>')   // URLs
    .replace(/:\d+(?::\d+)?\b/g, '')                         // :line:col
    .replace(/\b\d+(?:\.\d+)?\b/g, '<n>')                     // numbers
    .replace(/\s+/g, ' ')
    .slice(0, 200)

  const key = `${name}: ${m}`                                 // level 1: exact mistake

  const shape = m
    .replace(/(['"`])(?:(?!\1).)*\1/g, '<str>')               // quoted identifiers/values
    .replace(/\b[\w$.]+(?= is not (?:defined|a function|iterable))/g, '<id>')
    .replace(/\(reading <str>\)/g, '(reading <prop>)')
  const shapeKey = `${name}: ${shape}`                        // level 2: same kind of mistake

  return { key, shapeKey }
}
```

### Examples

| Raw error (from different students) | `key` | `shapeKey` |
|---|---|---|
| `TypeError: Cannot read properties of undefined (reading 'items') at app.js:22:15` | `TypeError: Cannot read properties of undefined (reading 'items')` | `TypeError: Cannot read properties of undefined (reading <prop>)` |
| `ReferenceError: items is not defined` | `ReferenceError: items is not defined` | `ReferenceError: <id> is not defined` |
| `ReferenceError: itmes is not defined` | `ReferenceError: itmes is not defined` | `ReferenceError: <id> is not defined` |
| `AxiosError: Request failed with status code 404` | `AxiosError: Request failed with status code <n>` | same |

### Grouping rules

1. Group by `(stepOrder, key)`. Students follow the same code, so identifiers usually match — the exact key is the most useful label ("`items is not defined`" tells the prof exactly what to explain).
2. Groups on the same step that share a `shapeKey` are shown as **one group with variants** ("`<id> is not defined` — items ×9, itmes ×2").
3. Only groups with **≥ 2 students** (or ≥ 10% of those present, whichever is larger) appear in `errorGroups`.
4. Sort by student count desc, then most recent.
5. A student leaves a group as soon as their next run has no error.

---

## 5. TODO Parser → Lesson Plan

`server/lib/todo-parser.js` turns starter files into proposed steps. The prof always reviews them before saving.

### Recognised comment forms

| File | Pattern |
|---|---|
| `.js` | `// TODO 3: Emit add-to-cart` · `/* TODO: … */` |
| `.html` | `<!-- TODO 2 - Show the item list -->` |
| `.css` | `/* TODO 5: Highlight the selected category */` |

Regex core (case-insensitive): `TODO\s*(\d+)?\s*[:.)\-–]?\s*(.+?)\s*(?:\*\/|-->)?$`

### Ordering algorithm

1. Scan files in this order: `entryFile` first, then the rest in their stored order; within a file, top to bottom.
2. TODOs **with** numbers are sorted by number. Ties (the same number in two files, e.g. TODO 3 in `index.html` and `app.js`) become **one step** with two `todoRefs`.
3. TODOs **without** numbers follow, in scan order.
4. Titles: comment text, trimmed, first letter capitalised, max 120 chars.
5. Result: `[{ order: 1…n, title, todoRefs: [{ file, line }] }]`.

Edge cases covered by unit tests: no TODOs (→ one step "Complete the exercise"), duplicate numbers, gaps (1, 2, 5 → orders 1, 2, 3, original numbers kept in the description), TODO inside a string literal (ignored when not in a comment).

---

## 6. Catch-up & "What you missed"

**Promise to the student:** loading the checkpoint never loses your work.

```
Student clicks "Load checkpoint"
  1. Client POSTs current files → server saves Snapshot { reason: 'before_catchup' }
  2. Server returns the latest checkpoint
  3. Client computes diffSummary(attemptFiles, checkpointFiles)    ← Ray Yien's lib
  4. "What you missed" panel opens:
       • per file: lines you're missing (green) / lines that differ (red → green)
       • "First difference: app.js line 22 — you wrote `items`, checkpoint has `props.items`"
       • Optional: "What the class did since your last Done" = diff(checkpoint at your step, latest)
  5. Editor loads the checkpoint files; tile re-classifies on the next run
  6. "Restore my attempt" stays available (GET /my-attempts)
```

`client/src/utils/diff-summary.js` uses jsdiff `diffLines` per file and returns `{ files: [{ path, added, removed, firstDiffLine, hunks }] }`. Whitespace-only changes are ignored (`ignoreWhitespace: true`) so indentation doesn't drown out the real mistake.

---

## 7. Help Queue & Telegram Ping

- **Check my work** (`type: check`): TA opens the side-by-side view (student snapshot vs checkpoint for that step) and either **Pass** (student gets a tick on that step — shown on their tile too) or **Reply** with what's wrong.
- **Ask for help** (`type: help`): same queue; if the student ticks "Come to my seat" (`inPerson`) and enters a seat, the server pings staff on Telegram.
- **Queue order:** oldest first (`createdAt`). Each card shows live waiting time; requests waiting > 3 min get an amber badge.
- **Claiming** uses a conditional update (`status: 'open'` → `'claimed'`), so two TAs can't take the same request — the second gets 409.
- **Who gets pinged:** staff currently connected to `session:<id>:staff` with Telegram linked; if none, every TA of the course with Telegram linked.

Example ping:

```
🙋 Help needed — IS216 G2 · Mini Cart
Alex Tan · Row C, seat 4
Step 4: Emit add-to-cart from ItemCard
Open: https://<app>/help/66f1…
```

---

## 8. Spotlight Anonymisation

Spotlight projects **one** example of a common mistake next to the fix.

1. The prof clicks **Spotlight** on an error group.
2. The server picks the most recent **help-request snapshot** in that group (only code a student chose to send is eligible). No eligible snapshot → the prof sees "Nobody in this group has sent their code yet" and the Spotlight shows the error message + the fix only.
3. Excerpt = the error's file, ±6 lines around the error line.
4. Anonymise: remove comments, replace the student's name/email (and any other enrolled student's name) with `Student`, cap at 30 lines.
5. Fix = the same region from the latest checkpoint.
6. `spotlight:show` goes to everyone; the student whose code it is gets no special marker.

---

## 9. Replay & GitHub Import / Export

### Replay

`GET /api/sessions/:id/replay` returns checkpoints in `seq` order. For each, the client shows the step title, the prof's note, the code with lines changed since the previous checkpoint highlighted, and a live preview (same sandbox as the workspace, read-only). Notes and bookmarks anchor to a `checkpointId`. Search covers step titles, checkpoint notes and the user's own notes across their courses (MongoDB text index + case-insensitive regex on titles, input escaped).

### GitHub import (server token) — `github.service.importRepo()`

```
1. Parse https://github.com/<owner>/<repo>[/tree/<ref>/<dir>]
2. GET /repos/{owner}/{repo}                                  → default branch if no ref
3. GET /repos/{owner}/{repo}/git/trees/{ref}?recursive=1      → file list
4. Keep files under <dir> with .html .css .js .json .md, ≤ 20 files, ≤ 200 KB total
5. GET /repos/{owner}/{repo}/contents/{path}?ref={ref}        → base64 → text   (one per file)
6. Return files → TODO parser → draft lesson
```

≈ 2 + n requests per import.

### GitHub export (student's OAuth token) — one commit per checkpoint

Uses the Git Data API so each step becomes its own commit:

```
1. POST /user/repos { name, private, auto_init: true }              → repo with README on main
2. GET  /repos/{o}/{r}/git/ref/heads/main                            → parentSha
3. For each checkpoint (seq order):
     POST /repos/{o}/{r}/git/trees  { tree: [README.md + checkpoint files as
                                      { path, mode: '100644', type: 'blob', content }] }
                                                                      → treeSha
     POST /repos/{o}/{r}/git/commits { message: "Step 3: Fetch items for the selected category",
                                       tree: treeSha, parents: [parentSha] } → parentSha = new sha
4. PATCH /repos/{o}/{r}/git/refs/heads/main { sha: parentSha }
5. Return https://github.com/{o}/{r}/commits/main
```

≈ 3 + 2 × checkpoints requests (19 for an 8-step lesson). Trees are built **without** `base_tree`, so each commit contains exactly that checkpoint's files. The generated README lists each step with a link to its commit and, if the student opts in, their notes. Name already taken → 422 → the UI asks for another name. The OAuth token lives only in server memory for 10 minutes and is never written to MongoDB.

---

## 10. Insights Report

`server/lib/report-builder.js` runs when a session ends, using `StudentStatus.stateHistory`, checkpoints and help requests. All figures are counts — no per-student detail leaves the report.

For each step *k* (with `t_k` = time of the checkpoint for step *k*):

| Metric | Definition |
|---|---|
| `reachedDone` | Students whose `doneStepOrder` ever reached ≥ *k* |
| `dropOff` | `reachedDone(k-1) − reachedDone(k)` |
| `errorStudents` | Distinct students with any `error` state while working on step *k* |
| `peakBehind` | Max students in `behind` between `t_k` and `t_{k+1}` |
| `medianMinutesToDone` | Median of (first Done ≥ *k*) − `t_{k-1}` (session start for *k* = 1) |
| `catchUps`, `helpRequests` | Counts tagged with step *k* |

**Trouble score** (shown with its formula, so profs can trust it):
`trouble_k = 0.4 × errorStudents/joined + 0.3 × dropOff/joined + 0.2 × catchUps/joined + 0.1 × helpRequests/joined`

The report highlights the top 3 steps, the top 5 error groups and the peak "I'm lost" count.

**Trends across terms** (`GET /api/lessons/:id/trends`): for every past session of the same lesson (or a lesson with the same title in a course with the same course code), a step counts as *recurring trouble* if it was in the top 3 by trouble score in at least two sessions. Steps are matched by `stepId` when the lesson is reused, otherwise by normalised title.

**Digest** (sent through Karin's Telegram service):
- **To TAs:** attendance, top 3 trouble steps, top errors, open help requests left.
- **To absent students** (enrolled, no `StudentStatus` in the session, Telegram linked): "You missed *Week 6 · Mini Cart*. Replay: <link>. Most classmates got stuck at Step 4 — start there."

---

## 11. Telegram Bot Integration

| Concern | Design |
|---|---|
| Linking | Profile → "Link Telegram" → one-time code (10 min) → deep link `https://t.me/<bot>?start=<code>` → bot receives `/start <code>` → `bindTelegramChat` |
| Receiving updates | Long polling `getUpdates` (`timeout=30`, tracked `offset`) — works locally without a public URL. Only one process may poll a token (`TELEGRAM_POLLING=true` on the deployed instance only) |
| Sending | `sendMessage` with `parse_mode: 'HTML'`; all user text HTML-escaped |
| Rate limits | Outgoing queue, ≥ 50 ms between messages; on HTTP 429 wait `retry_after` seconds and retry once |
| Failures | Never fail the user's action — log, mark `telegramNotifiedAt: null`, show "notified in app only" |
| Bot commands | `/start <code>` (link), `/stop` (unlink), `/help` |
| Tests | `EXTERNAL_APIS=mock` swaps in `fakes/telegram.fake.js`, which records messages to an outbox that E2E tests assert on |

---

## 12. Stretch Goals (COULD)

| Stretch | Sketch | Owner |
|---|---|---|
| **Automatic step checks** | When the prof saves checkpoint *k*, generate checks from what changed since *k-1*: elements with new ids/classes (`dom-exists`), new mock-API calls seen when running the checkpoint in a hidden sandbox (`network-call`), "runs without errors" (`no-runtime-error`). The student's sandbox runs them after each run and reports `checks: { passed, total }`. The prof can switch any check off; failed checks show as "possible issue", never as a hard fail. | Dylan |
| **AI hints** | Rule table first (`shapeKey` → hint, e.g. `<id> is not defined` → "Check the spelling, or whether it was declared in this scope"). If `hintsEnabled` and `LLM_API_KEY` set: prompt with step title, error and ±10 lines; instruction "one sentence, max 25 words, never give code". Output filter rejects code blocks or > 2 lines. Cached by `(lessonId, stepId, errorKey)`; daily cap of 200 calls. | Karin |
| **VS Code reporter** | A one-line `<script src="…/reporter.js" data-token="…">` in the starter code, with a per-student reporter token, posting the same status payload over HTTPS. | Ying Qi |

---

## External API Budget

Markers test each feature up to 5–6 times a day (FAQ 19).

| API | Free limit | Our worst case per day | Safe? |
|---|---|---|:---:|
| Telegram Bot API | Free; ~30 messages/s overall, ~20/min into one group | < 200 messages | ✅ |
| GitHub REST (server token, imports) | 5,000 requests/hour | 6 imports × ~25 requests = 150 | ✅ |
| GitHub REST (student OAuth, export) | 5,000 requests/hour per user | 6 exports × ~19 requests = 114 | ✅ |
| LLM (KIV) | Depends on provider | ≤ 200 calls (capped), most served from cache | ✅ with cap |
