# CodeAlong — Data Model & Authorization

## Entity Overview

Ten collections in MongoDB Atlas, connected by ObjectId references. Owner of all model files: **Dylan** (others request changes by PR, so the schema stays consistent).

```
                ┌──────────┐
                │   User   │◄───────────────────────────────────────────┐
                └────┬─────┘                                            │
        owner/staff/ │ students                                          │
                ┌────▼─────┐   1:N   ┌──────────┐   1:N   ┌───────────┐ │
                │  Course  │────────▶│  Lesson  │────────▶│  Session  │ │
                └──────────┘         │ steps[]  │         └─────┬─────┘ │
                                     │ files[]  │               │       │
                                     └──────────┘               │ 1:N   │
          ┌───────────────┬──────────────┬──────────────┬───────┴──────┬┴─────────────┐
          ▼               ▼              ▼              ▼              ▼              ▼
   ┌────────────┐ ┌──────────────┐ ┌──────────┐ ┌─────────────┐ ┌──────────┐ ┌───────────────┐
   │ Checkpoint │ │StudentStatus │ │ Snapshot │ │ HelpRequest │ │   Note   │ │ SessionReport │
   │ seq, files │ │ 1 per student│ │ attempts │ │ → snapshotId│ │ per user │ │  1 per session│
   └────────────┘ └──────────────┘ └──────────┘ └─────────────┘ └──────────┘ └───────────────┘
```

**Relationships**
- A **User** (prof) owns many **Courses**; a Course lists TAs in `staff[]` and students in `students[]`.
- A **Course** has many **Lessons**; a Lesson embeds its starter `files[]` and ordered `steps[]`.
- A **Lesson** can be taught many times → many **Sessions** (one per live class).
- A **Session** has many **Checkpoints** (one per saved step), one **StudentStatus** per participating student, many **Snapshots**, **HelpRequests** and **Notes**, and one **SessionReport** after it ends.
- A **HelpRequest** points to exactly one **Snapshot** — that is the *only* path by which staff can see a student's code.

---

## Shared Sub-document: `File`

Used in `Lesson.files`, `Checkpoint.files` and `Snapshot.files`.

| Field | Type | Constraints | Notes |
|---|---|---|---|
| `path` | String | required, trimmed, max 120, pattern `^[\w\-./]+$`, no `..` | e.g. `index.html`, `js/app.js` |
| `language` | String | enum: `html`, `css`, `js`, `json`, `md` | Picks the CodeMirror mode |
| `content` | String | required (may be empty) | Whole-snapshot total ≤ `Limits.MAX_SNAPSHOT_BYTES` (200 KB) |

Max `Limits.MAX_FILES` (20) files per array. Validated in zod and in a Mongoose `validate` on the array.

---

## User

| Field | Type | Constraints | Notes |
|---|---|---|---|
| `name` | String | required, trimmed, 2–60 chars | Display name |
| `email` | String | required, unique, lowercase, trimmed, email format | Login identifier |
| `passwordHash` | String | required, `select: false` | bcrypt, 10 salt rounds |
| `role` | String | enum `Role.PROF / TA / STUDENT`, default `student` | Account type |
| `telegram.chatId` | String | optional, `select: false` | Set when the user links the bot |
| `telegram.linkCode` | String | optional, `select: false` | One-time code, expires |
| `telegram.linkCodeExpiresAt` | Date | optional | 10 minutes after issue |
| `createdAt` / `updatedAt` | Date | `timestamps: true` | |

**Indexes:** `email` unique.

> **Account bootstrap:** self-registration creates `student` accounts only. Prof and TA accounts are created by the seed script, or a prof promotes a user to TA when adding them as course staff (`POST /api/courses/:id/staff`). There is no self-service path to `prof`.

```js
// models/user.model.js
export const createUser       = (data)        => User.create(data)
export const findByEmail      = (email, withHash = false) =>
  withHash ? User.findOne({ email }).select('+passwordHash') : User.findOne({ email })
export const findById         = (id)          => User.findById(id)
export const updateProfile    = (id, data)    => User.updateOne({ _id: id }, data)
export const setTelegramLink  = (id, code, expiresAt) =>
  User.updateOne({ _id: id }, { 'telegram.linkCode': code, 'telegram.linkCodeExpiresAt': expiresAt })
export const bindTelegramChat = (code, chatId) =>
  User.findOneAndUpdate(
    { 'telegram.linkCode': code, 'telegram.linkCodeExpiresAt': { $gt: new Date() } },
    { 'telegram.chatId': chatId, $unset: { 'telegram.linkCode': 1, 'telegram.linkCodeExpiresAt': 1 } },
    { new: true })
```

---

## Course

| Field | Type | Constraints | Notes |
|---|---|---|---|
| `code` | String | required, uppercase, trimmed, e.g. `IS216` | |
| `title` | String | required, trimmed, max 100 | |
| `term` | String | required, e.g. `AY2026/27 T1` | Lets insights compare terms |
| `section` | String | required, e.g. `G2` | |
| `ownerId` | ObjectId → User | required | The prof who created it |
| `staff` | [{ `userId` → User, `role`: `prof` \| `ta` }] | unique userIds | Owner is always included as `prof` |
| `students` | [ObjectId → User] | unique | Filled by enrolment |
| `enrolCode` | String | required, unique, 6 chars | Students enrol once per course |
| `archived` | Boolean | default `false` | Archived courses are read-only |

**Indexes:** `enrolCode` unique; `{ 'staff.userId': 1 }`; `{ students: 1 }`; `{ code: 1, term: 1, section: 1 }` unique.

```js
// models/course.model.js
export const createCourse   = (data)               => Course.create(data)
export const findForUser    = (userId)             => Course.find({ $or: [{ 'staff.userId': userId }, { students: userId }], archived: false })
export const findById       = (id)                 => Course.findById(id)
export const findByEnrolCode= (code)               => Course.findOne({ enrolCode: code, archived: false })
export const addStudent     = (id, userId)         => Course.updateOne({ _id: id }, { $addToSet: { students: userId } })
export const removeStudent  = (id, userId)         => Course.updateOne({ _id: id }, { $pull: { students: userId } })
export const addStaff       = (id, userId, role)   => Course.updateOne({ _id: id, 'staff.userId': { $ne: userId } }, { $push: { staff: { userId, role } } })
export const updateCourse   = (id, data)           => Course.updateOne({ _id: id }, data)
export const deleteCourse   = (id)                 => Course.deleteOne({ _id: id })   // cascade in controller
```

---

## Lesson (embeds `files[]` and `steps[]`)

| Field | Type | Constraints | Notes |
|---|---|---|---|
| `courseId` | ObjectId → Course | required | |
| `title` | String | required, trimmed, max 100 | e.g. "Week 6 · Mini Cart" |
| `week` | Number | integer 1–13, optional | For ordering in the library |
| `description` | String | max 500 | |
| `files` | [File] | 1–20 files | The starter code (with TODO comments) |
| `entryFile` | String | required, must match a `files.path` | Usually `index.html` |
| `steps` | [Step] | ordered by `order` | See below |
| `mockApi` | [{ `method`, `path`, `status`, `body` (Mixed) }] | max 30 routes | Fake backend for lessons that call axios — see [05-core-logic.md](./05-core-logic.md#2-mock-api) |
| `source` | { `type`: `upload` \| `github`, `repo`, `ref`, `dir` } | | Where the starter came from |
| `status` | String | enum `draft`, `ready`, `archived`, default `draft` | Only `ready` lessons can start a session |
| `createdBy` | ObjectId → User | required | |
| timestamps | | | |

### Step (sub-document)

| Field | Type | Constraints | Notes |
|---|---|---|---|
| `_id` | ObjectId | auto | Referenced by checkpoints, statuses, help requests |
| `order` | Number | integer ≥ 1, unique within lesson | 1-based display order |
| `title` | String | required, max 120 | Defaults to the TODO text |
| `description` | String | max 1000 | Prof's explanation |
| `todoRefs` | [{ `file`, `line` }] | optional | Where the TODO was found (one step can span files — see [05-core-logic.md](./05-core-logic.md#5-todo-parser--lesson-plan)) |
| `checks` | [{ `id`, `label`, `kind`, `spec` (Mixed), `enabled` }] | optional (COULD) | Automatic step checks — stretch |

**Indexes:** `{ courseId: 1, week: 1 }`.

```js
// models/lesson.model.js
export const createLesson   = (data)                 => Lesson.create(data)
export const findByCourse   = (courseId)             => Lesson.find({ courseId, status: { $ne: 'archived' } }).sort({ week: 1, createdAt: 1 })
export const findById       = (id)                   => Lesson.findById(id)
export const updateLesson   = (id, data)             => Lesson.updateOne({ _id: id }, data, { runValidators: true })
export const replaceSteps   = (id, steps)            => Lesson.updateOne({ _id: id }, { steps }, { runValidators: true })
export const updateStep     = (id, stepId, data)     => Lesson.updateOne({ _id: id, 'steps._id': stepId }, prefixKeys('steps.$.', data))
export const deleteStep     = (id, stepId)           => Lesson.updateOne({ _id: id }, { $pull: { steps: { _id: stepId } } })
export const deleteLesson   = (id)                   => Lesson.deleteOne({ _id: id })  // blocked if any session exists → archive instead
```

---

## Session (one live class)

| Field | Type | Constraints | Notes |
|---|---|---|---|
| `lessonId` | ObjectId → Lesson | required | |
| `courseId` | ObjectId → Course | required | Denormalised for fast auth checks |
| `startedBy` | ObjectId → User | required | |
| `joinCode` | String | required, 6 chars from `ABCDEFGHJKMNPQRSTUVWXYZ23456789` | Unique among live sessions |
| `status` | String | enum `live`, `ended`, default `live` | |
| `currentStepId` | ObjectId | optional | Step of the latest checkpoint |
| `settings` | { `allowCatchUp`: Boolean = true, `hintsEnabled`: Boolean = false, `idleAfterSec`: Number = 180 } | | Prof can toggle during class |
| `spotlight` | { `errorKey`, `excerpt`, `checkpointId`, `shownAt` } \| null | | Anonymised; see [05-core-logic.md](./05-core-logic.md#8-spotlight-anonymisation) |
| `startedAt` | Date | default now | |
| `endedAt` | Date | set on end | |

**Indexes:** partial unique `{ joinCode: 1 }` where `status: 'live'`; `{ courseId: 1, startedAt: -1 }`; `{ lessonId: 1 }`.

```js
// models/session.model.js
export const createSession   = (data)          => Session.create(data)
export const findLiveByCode  = (joinCode)      => Session.findOne({ joinCode, status: 'live' })
export const findById        = (id)            => Session.findById(id)
export const findByCourse    = (courseId)      => Session.find({ courseId }).sort({ startedAt: -1 })
export const setCurrentStep  = (id, stepId)    => Session.updateOne({ _id: id }, { currentStepId: stepId })
export const setSpotlight    = (id, spotlight) => Session.updateOne({ _id: id }, { spotlight })
export const endSession      = (id)            => Session.updateOne({ _id: id, status: 'live' }, { status: 'ended', endedAt: new Date() })
```

---

## Checkpoint

| Field | Type | Constraints | Notes |
|---|---|---|---|
| `sessionId` | ObjectId → Session | required | |
| `stepId` | ObjectId | required, must exist in the lesson's steps | |
| `seq` | Number | required, integer ≥ 1 | 1, 2, 3… in save order |
| `files` | [File] | required | Full snapshot of the prof's code (small, simple, replayable) |
| `note` | String | max 500 | Optional explanation shown in replay |
| `createdBy` | ObjectId → User | required | |
| `createdAt` | Date | timestamps | |

**Indexes:** unique `{ sessionId: 1, seq: 1 }`.

```js
// models/checkpoint.model.js
export const createCheckpoint = (data)               => Checkpoint.create(data)
export const findBySession    = (sessionId)          => Checkpoint.find({ sessionId }).sort({ seq: 1 })
export const findLatest       = (sessionId)          => Checkpoint.findOne({ sessionId }).sort({ seq: -1 })
export const findBySeq        = (sessionId, seq)     => Checkpoint.findOne({ sessionId, seq })
export const nextSeq          = async (sessionId)    => ((await findLatest(sessionId))?.seq ?? 0) + 1
```

> Two "Save checkpoint" clicks at once could race for the same `seq`; the unique index rejects the second insert and the controller retries once with a fresh `seq`.

---

## StudentStatus (latest state only — one per student per session)

| Field | Type | Constraints | Notes |
|---|---|---|---|
| `sessionId` | ObjectId → Session | required | |
| `studentId` | ObjectId → User | required | |
| `doneStepOrder` | Number | integer ≥ 0, default 0 | Highest step the student marked Done |
| `state` | String | enum `TileState`: `done`, `in_progress`, `error`, `behind`, `idle` | Computed by the server classifier, never trusted from the client |
| `error` | { `name`, `message`, `file`, `line`, `key`, `shapeKey`, `at` } \| null | message max 300 | Normalised by `lib/error-normalizer.js` |
| `checks` | { `passed`, `total` } | | Only when automatic checks exist (COULD) |
| `lost` | Boolean | default `false` | **Never sent to staff per student** — only counted |
| `catchUps` | Number | default 0 | Times the student loaded a checkpoint (for insights) |
| `stateHistory` | [{ `state`, `stepOrder`, `at` }] | capped at 200 entries | Feeds the insights funnel |
| `lastSeenAt` | Date | required | Heartbeat or any event |
| `joinedAt` | Date | default now | Attendance for digests |

**Indexes:** unique `{ sessionId: 1, studentId: 1 }`; `{ sessionId: 1, state: 1 }`.

```js
// models/student-status.model.js
export const upsertStatus    = (sessionId, studentId, patch, historyEntry) =>
  StudentStatus.findOneAndUpdate(
    { sessionId, studentId },
    { $set: patch, ...(historyEntry && { $push: { stateHistory: { $each: [historyEntry], $slice: -200 } } }),
      $setOnInsert: { joinedAt: new Date() } },
    { upsert: true, new: true })
export const findBySession   = (sessionId) => StudentStatus.find({ sessionId }).populate('studentId', 'name')
export const countLost       = (sessionId) => StudentStatus.countDocuments({ sessionId, lost: true })
export const incrementCatchUp= (sessionId, studentId) => StudentStatus.updateOne({ sessionId, studentId }, { $inc: { catchUps: 1 } })
```

---

## Snapshot (a student's code)

| Field | Type | Constraints | Notes |
|---|---|---|---|
| `sessionId` | ObjectId → Session | required | |
| `studentId` | ObjectId → User | required | |
| `stepId` | ObjectId | optional | Step the student was on |
| `reason` | String | enum `autosave`, `before_catchup`, `help_request` | |
| `files` | [File] | required | |
| `createdAt` | Date | timestamps | |

**Rules**
- `autosave`: **one per student per session** (upsert) — the latest editor state, so a refresh or a laptop switch doesn't lose work.
- `before_catchup` and `help_request`: **immutable**, a new document each time.
- Staff can read a snapshot **only** through the help request that references it.

**Indexes:** `{ sessionId: 1, studentId: 1, reason: 1, createdAt: -1 }`; partial unique `{ sessionId: 1, studentId: 1 }` where `reason: 'autosave'`.

```js
// models/snapshot.model.js
export const upsertAutosave  = (sessionId, studentId, stepId, files) =>
  Snapshot.findOneAndUpdate({ sessionId, studentId, reason: 'autosave' }, { stepId, files }, { upsert: true, new: true })
export const findAutosave    = (sessionId, studentId) => Snapshot.findOne({ sessionId, studentId, reason: 'autosave' })
export const createSnapshot  = (data)                 => Snapshot.create(data)   // before_catchup | help_request
export const findMineBySession = (sessionId, studentId) =>
  Snapshot.find({ sessionId, studentId, reason: { $ne: 'autosave' } }).sort({ createdAt: -1 })
```

---

## HelpRequest

| Field | Type | Constraints | Notes |
|---|---|---|---|
| `sessionId` | ObjectId → Session | required | |
| `studentId` | ObjectId → User | required | |
| `stepId` | ObjectId | required | |
| `type` | String | enum `check` ("Check my work"), `help` ("Ask for help") | |
| `snapshotId` | ObjectId → Snapshot | required | Created in the same request; `reason: 'help_request'` |
| `message` | String | max 300 | Optional note from the student |
| `inPerson` | Boolean | default `false` | `help` only — triggers a Telegram ping |
| `seat` | String | max 20 | e.g. "Row C, 4" — only for in-person |
| `status` | String | enum `open`, `claimed`, `passed`, `replied`, `resolved`, `cancelled`, default `open` | Transitions below |
| `claimedBy` | ObjectId → User | optional | |
| `reply` | String | max 1000 | TA feedback |
| `telegramNotifiedAt` | Date | optional | |
| timestamps | | | `createdAt` drives queue order |

**Allowed transitions**

| From | To | Who |
|---|---|---|
| `open` | `claimed` | Staff |
| `open` | `cancelled` | The student who created it |
| `claimed` | `passed` | Claiming staff (only `type: check`) |
| `claimed` | `replied` | Claiming staff |
| `claimed` | `resolved` | Claiming staff (in-person help done) |
| `claimed` | `open` | Claiming staff (release) |

**Indexes:** `{ sessionId: 1, status: 1, createdAt: 1 }`; partial unique `{ sessionId: 1, studentId: 1, type: 1 }` where `status ∈ {open, claimed}` — one active request per type per student.

```js
// models/help-request.model.js
export const createRequest   = (data)          => HelpRequest.create(data)
export const findQueue       = (sessionId)     => HelpRequest.find({ sessionId, status: { $in: ['open', 'claimed'] } }).sort({ createdAt: 1 }).populate('studentId', 'name')
export const findMine        = (sessionId, studentId) => HelpRequest.find({ sessionId, studentId }).sort({ createdAt: -1 })
export const findById        = (id)            => HelpRequest.findById(id).populate('studentId', 'name')
export const transition      = (id, from, to, patch = {}) =>
  HelpRequest.findOneAndUpdate({ _id: id, status: { $in: [].concat(from) } }, { status: to, ...patch }, { new: true })
  // null result = someone else changed it first → controller returns 409
```

---

## Note (notes and bookmarks)

| Field | Type | Constraints | Notes |
|---|---|---|---|
| `userId` | ObjectId → User | required | Private to the author |
| `sessionId` | ObjectId → Session | required | |
| `checkpointId` | ObjectId → Checkpoint | optional | Anchor in the replay |
| `kind` | String | enum `note`, `bookmark` | |
| `body` | String | max 2000; required for `note` | |
| timestamps | | | |

**Indexes:** `{ userId: 1, sessionId: 1 }`; text index on `body` for search.

---

## SessionReport (insights, one per ended session)

| Field | Type | Notes |
|---|---|---|
| `sessionId` | ObjectId → Session, unique | |
| `courseId`, `lessonId` | ObjectId | For cross-term trends |
| `attendance` | { `joined`, `enrolled` } | |
| `steps` | [{ `stepId`, `order`, `title`, `reachedDone`, `peakError`, `peakBehind`, `medianMinutesToDone`, `catchUps`, `helpRequests` }] | Per-step funnel |
| `topErrors` | [{ `key`, `sampleMessage`, `stepOrder`, `students` }] | Top 5, counts only |
| `lostPeak` | Number | Max simultaneous "I'm lost" |
| `generatedAt` | Date | |
| `digest` | { `sentAt`, `recipients` } | Telegram digest log (counts only) |

Built by `lib/report-builder.js` when the session ends — see [05-core-logic.md](./05-core-logic.md#10-insights-report).

---

## Delete Cascades

| Deleting | Also deletes / changes |
|---|---|
| Course (owner prof only) | Its lessons, sessions and everything under each session (below) |
| Lesson | Blocked if any session exists — the lesson is archived instead (replays must keep working) |
| Session (owner prof only, ended sessions) | Checkpoints, statuses, snapshots, help requests, notes, report |
| Removing a student from a course | Their future access only; past statuses/snapshots stay for the report |

---

## Authorization Matrix

"Staff" = listed in `course.staff`. "Enrolled" = listed in `course.students`.

| Action | Student (enrolled) | TA (staff) | Prof (staff) |
|---|:---:|:---:|:---:|
| Register / log in / edit own profile / link Telegram | ✅ | ✅ | ✅ |
| Enrol in a course with `enrolCode` | ✅ | — | — |
| View course + lesson library | ✅ | ✅ | ✅ |
| **Create / edit / archive** course | ❌ | ❌ | ✅ (owner) |
| Add TA to course | ❌ | ❌ | ✅ (owner) |
| **Create / edit / delete** lesson, edit steps, import from GitHub | ❌ | ❌ | ✅ |
| Start / end a live session, save checkpoints, Spotlight | ❌ | ❌ | ✅ |
| Join a live session | ✅ | ✅ (as staff) | ✅ |
| Report own status, tap Done, toggle "I'm lost", autosave, catch up | ✅ | — | — |
| View class dashboard (tiles, counts, error groups) | ❌ | ✅ | ✅ |
| See which student tapped "I'm lost" | ❌ | ❌ | ❌ (nobody) |
| Send Check my work / Ask for help | ✅ | — | — |
| View help queue, claim, pass, reply, resolve | ❌ | ✅ | ✅ |
| View a student's code | ❌ | Only via a help request the student sent | Only via a help request the student sent |
| View replay of a session | ✅ (if enrolled) | ✅ | ✅ |
| Notes / bookmarks | Own only | Own only | Own only |
| Export replay to GitHub | ✅ (own account) | ✅ | ✅ |
| View insights report, send Telegram digest | ❌ | ✅ view | ✅ view + send |

---

## Auth Token Shape

On login the server sets an httpOnly cookie `ca_token` containing a JWT signed with `JWT_SECRET`:

```js
// JWT payload
{ sub: "6507…", role: "prof", name: "Prof Lee", iat: 1760000000, exp: 1760043200 }

// After requireAuth, controllers read:
req.user = { _id: "6507…", role: "prof", name: "Prof Lee" }
```

Cookie flags: `httpOnly`, `sameSite: 'lax'`, `secure` in production, `maxAge = JWT_TTL_HOURS`. The same cookie authenticates the Socket.IO handshake (see [09-integration-contracts.md](./09-integration-contracts.md#2-auth)).

---

## Validation Rules Summary

| Form / payload | Required | Rules |
|---|---|---|
| Register | name, email, password | email format; password ≥ 8 chars; unique email; role always `student` |
| Login | email, password | generic "Invalid email or password" on any mismatch; rate-limited (10/min/IP) |
| Course | code, title, term, section | code uppercase A–Z0–9; unique code+term+section |
| Lesson | title, files, entryFile | 1–20 files; paths safe; entryFile exists; total ≤ 200 KB |
| Step edit | title | order unique and contiguous after reorder |
| GitHub import | repo URL | `https://github.com/<owner>/<repo>` (optional `/tree/<ref>/<dir>`); only `.html/.css/.js/.json/.md` files kept |
| Start session | lessonId | lesson `status: ready`, has ≥ 1 step, caller is prof staff |
| Join session | joinCode | 6 chars; live session; caller enrolled or staff |
| Checkpoint | stepId, files | step belongs to the lesson; session live |
| Status update (socket) | stepOrder, error? | stepOrder within 0…steps.length; error message truncated to 300 chars |
| Help request | type, stepId, files | one active request per type; message ≤ 300; seat required if `inPerson` |
| Help reply | reply | 1–1000 chars; only by claiming staff |
| Note | kind, body (for notes) | body ≤ 2000 |

On validation failure the API returns **400** with `error.fields` — the Vue form shows the message under each field and keeps what the user typed.
