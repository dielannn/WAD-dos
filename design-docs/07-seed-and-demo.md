# CodeAlong — Seed Data, Demo Script & Submission

## Seed Data

`server/seed/seed.js` (run with `npm run seed`) gives every teammate, every test and the markers the same starting point. It is **idempotent**: clear → insert users → course → lesson → past session (with checkpoints, statuses, help requests, notes, report).

All names below are fictional — don't use classmates' names in submitted data.

### Seed users (password for all: `pass1234`)

| # | Name | Email | Role | Notes |
|---|---|---|---|---|
| 1 | Prof Lee | `prof@codealong.test` | prof | Owns IS216 G2; Telegram linked in demo env |
| 2 | Ana Lim | `ta.ana@codealong.test` | ta | Staff of IS216 G2 |
| 3 | Ben Ong | `ta.ben@codealong.test` | ta | Staff of IS216 G2 |
| 4 | Alex Tan | `alex@codealong.test` | student | Enrolled — used for the live demo |
| 5 | Bea Koh | `bea@codealong.test` | student | Enrolled — second live laptop |
| 6 | Chen Wei | `chen@codealong.test` | student | Enrolled — third live laptop |
| 7–14 | Dana, Eli, Faye, Gus, Hana, Ivan, Jae, Kim | `<first>@codealong.test` | student | Enrolled — appear in the past session |
| 15 | Zoe Ng | `zoe@codealong.test` | student | **Not** enrolled — for the "not in this course" check |

> These credentials go in the README (the briefing asks for any username/password details there). Passwords are bcrypt-hashed in the seed script.

### Seed course

| Code | Title | Term | Section | Enrol code | Staff | Students |
|---|---|---|---|---|---|---|
| IS216 | Web Application Development II | AY2026/27 T1 | G2 | `WAD2G2` | Prof Lee, Ana, Ben | Alex … Kim (11) |

A second course, **IS216 G2 · AY2025/26 T1** (archived), holds one past session of the same lesson so "recurring trouble steps across terms" has data.

### Seed lesson: "Week 6 · Mini Cart" (status `ready`)

Starter files: `index.html`, `app.js`, `style.css` — our own version of a Vue mini-cart exercise (props, emits, axios, localStorage), written by the team.

| Step | TODO title | File |
|:---:|---|---|
| 1 | Create the Vue app with a list of categories | `app.js` |
| 2 | Fetch items for the selected category with axios | `app.js` |
| 3 | Create the `item-card` component with an `items` prop | `app.js`, `index.html` |
| 4 | Emit `add-to-cart` from `item-card` | `app.js` |
| 5 | Add the item to the cart in the parent (increase quantity if it's already there) | `app.js` |
| 6 | Show the cart total with a computed property | `app.js` |
| 7 | Save the cart to localStorage and load it on start | `app.js` |
| 8 | Add Reset and Checkout buttons | `index.html`, `app.js` |

Mock API: `GET /categories`, `GET /items` (filterable by `?category=`), `POST /checkout`.

### Seed past session (ended, current term)

| Data | Contents | Demonstrates |
|---|---|---|
| 8 checkpoints | One per step, with short prof notes | Replay, GitHub export |
| 11 statuses with `stateHistory` | Most students fine; 6 hit `props.items` vs `items` at Step 4; 3 fall behind at Step 7 | Insights funnel, Step 4 as top trouble step |
| 5 help requests | 2 passed checks, 2 replied helps, 1 resolved in-person | Help history in replay; Spotlight source |
| 4 notes/bookmarks | Alex's notes on Steps 4 and 7 | Notes + search |
| SessionReport | Pre-built | Report page renders immediately |

### Seed script outline

```js
// server/seed/seed.js
import mongoose from 'mongoose'
import bcrypt from 'bcrypt'
import { env } from '../config/env.js'
import * as models from '../models/index.js'
import { parseTodos } from '../lib/todo-parser.js'
import { buildReport } from '../lib/report-builder.js'
import { loadFixture } from './fixtures/load.js'

async function seed() {
  await mongoose.connect(env.MONGODB_URI)

  // 1. Clear (children first)
  for (const m of ['Note', 'HelpRequest', 'Snapshot', 'StudentStatus', 'Checkpoint',
                   'SessionReport', 'Session', 'Lesson', 'Course', 'User']) {
    await models[m].deleteMany({})
  }

  // 2. Users
  const hash = await bcrypt.hash('pass1234', 10)
  const users = await models.User.insertMany(SEED_USERS.map(u => ({ ...u, passwordHash: hash })))

  // 3. Courses (current + archived last term)
  // 4. Lesson from fixtures/mini-cart → parseTodos(files) → steps (same parser the app uses)
  // 5. Past sessions: checkpoints, statuses with stateHistory, snapshots, help requests, notes
  // 6. buildReport() for each ended session (same builder the app uses)

  console.log('Seed complete:', await countAll())
  await mongoose.disconnect()
}

seed().catch(err => { console.error(err); process.exit(1) })
```

> Reusing `parseTodos` and `buildReport` in the seed means seed data always matches what the real code would produce.

---

## Demo Setup (before recording or presenting)

| ✓ | Step |
|:---:|---|
| ☐ | `npm run seed` on the deployed database (or local for the recording) |
| ☐ | Prof, TA and three students **already logged in** in five Chrome profiles — **no login/registration on screen** (Q&A slide 11) |
| ☐ | One student window docked in DevTools device mode at **575 px** with the **console open** (Q&A slide 11) |
| ☐ | `npm run simulate -- --session <code> --students 25` ready in a terminal, so the dashboard looks like a real class |
| ☐ | Telegram open on a phone (screen-mirrored) logged in as the TA |
| ☐ | GitHub repo page open in a tab (Q&A slide 11) |
| ☐ | Screen recording at 1920×1080, browser zoom 110% for readability |
| ☐ | Notifications off on every device |

---

## Video Script (≤ 12 minutes, YouTube unlisted, 1080p)

Every member presents the part they built (check your section's rule on who must appear).

### 0:00–1:15 · The problem (Evan)

| Show | Say |
|---|---|
| Slide: a lab photo + the three pain points | One typo at step 2 and you're lost for the rest of the lesson; the prof can't see who's stuck; the file sent afterwards hides the steps |
| Slide: why chat / Live Share / ChatGPT don't fix it | Nobody shows the prof the room |
| One-sentence pitch | "CodeAlong helps IS216 profs and TAs see who is falling behind during live coding and get them back on track…" |

### 1:15–2:30 · Prof prepares the lesson (Evan, Ray Yien)

| Step | Action | What to show |
|:---:|---|---|
| 1 | Prof opens IS216 G2 → New lesson → **Import from GitHub** | Repo URL → files appear (GitHub API) |
| 2 | Review steps | 8 TODOs became an ordered plan; reorder one, edit a title |
| 3 | **Explain:** TODO parser rules (numbers, multiple files) | Code: `lib/todo-parser.js` + its unit test |

### 2:30–5:30 · Live class (Ying Qi, Dylan, Evan)

| Step | Action | What to show |
|:---:|---|---|
| 4 | Start session | Join code + QR on the "projector" |
| 5 | Three students join by code; simulator adds 25 more | Tiles fill in live |
| 6 | Prof completes Step 3, clicks **Save checkpoint** | Every student gets the toast instantly |
| 7 | Alex (575 px window) types `items` instead of `props.items` | Console shows the real error; gutter marker; Alex's tile turns **red** |
| 8 | Several simulated students hit the same error | Error group "`items is not defined` ×7" |
| 9 | Bea and Chen tap **I'm lost** | Lost count 2 — no names anywhere |
| 10 | **Explain:** sandbox + probe, status classifier, error normaliser | Diagram slide + `status-classifier.js`, `error-normalizer.js` |

### 5:30–7:00 · Catch up (Dylan, Ray Yien)

| Step | Action | What to show |
|:---:|---|---|
| 11 | Alex clicks **Load checkpoint** | "What you missed": the exact line he got wrong |
| 12 | Alex taps **Done** | Tile turns **green** on the prof's screen |
| 13 | Alex restores his old attempt to compare, then switches back | Nothing lost |
| 14 | Resize to 375 px and rotate | Workspace switches to tabs; bottom bar stays usable |

### 7:00–8:45 · Help (Karin, Evan)

| Step | Action | What to show |
|:---:|---|---|
| 15 | Bea → **Check my work** | TA queue shows the card instantly (no code until opened) |
| 16 | TA claims → side-by-side → **Pass** | Bea's step gets a tick |
| 17 | Chen → **Ask for help** → "Come to my seat" Row C 4 | Telegram message arrives on the TA's phone (Telegram Bot API) |
| 18 | Prof clicks **Spotlight** on the error group | Anonymised excerpt + fix on every screen |
| 19 | **Explain:** privacy by design — prof sees status, not code | Slide: what each role can see |

### 8:45–10:15 · After class (Ray Yien, Andric)

| Step | Action | What to show |
|:---:|---|---|
| 20 | Prof ends session | Students see "Class ended — open replay" |
| 21 | Replay: step through checkpoints, add a note, search | Changed lines highlighted |
| 22 | **Export to GitHub** | GitHub commits page: one commit per step |
| 23 | Insights report (seeded past session) | Funnel; Step 4 top trouble step; recurring across terms |
| 24 | Send digest | Telegram: TA summary + absent-student message |

### 10:15–11:45 · How it's built & tested (Andric, all)

| Slide | Content |
|---|---|
| Architecture | Vue 3 + Pinia + Bootstrap ↔ Express REST + Socket.IO ↔ MongoDB Atlas; Telegram + GitHub APIs |
| Data store | MongoDB Atlas collections and what's in each; privacy rule for snapshots |
| Responsive & styling | Breakpoint table; workspace tabs vs panes; tokens; colour + icon + label |
| Testing | Playwright run with prof + 3 students in one test (clip of the HTML report), 137 tests, unit tests for core logic |
| Credits & AI use | Libraries table; what AI was (and wasn't) used for |

### 11:45–12:00 · Close

What we'd do next (automatic step checks, VS Code reporter) + thanks.

---

## Presentation Slides (~10, PPTX or PDF)

| # | Slide | Content |
|:---:|---|---|
| 1 | Title | CodeAlong · T4 · G2 · members · **YouTube link** (required on slide 1) |
| 2 | Problem | Three pain points + why existing tools fall short |
| 3 | Solution & users | One-sentence pitch; prof/TA/student roles; core loop |
| 4 | Features & tiers | F1–F6 mapped to tiers; signature algorithm highlighted |
| 5 | Architecture | Stack diagram, REST vs Socket.IO, single-origin deployment |
| 6 | Data store | Collections, relationships, privacy rule |
| 7 | Core logic | Sandbox + classifier + error grouping (diagram) |
| 8 | External APIs | Telegram (pings, digests), GitHub (import, export as commits) — integration details |
| 9 | Design & responsiveness | Screens at 375 px and 1280 px; IDP rules applied |
| 10 | Testing | Strategy, repeatability, stable selectors, how to run |
| 11 | Credits, AI use, next steps | |

---

## Live Q&A Prep (at the front, with the app open)

Before Q&A: log in first, open pages at 575 px in DevTools with the console visible, have the GitHub repo open (Q&A slide 11).

| Likely question | Who answers | Short answer |
|---|---|---|
| Why not just use VS Code Live Share? | Evan | Live Share shows the prof's editor to students; it gives the prof no view of the class, no checkpoints to catch up to, no replay |
| Isn't this surveillance? | Karin | Status only, no keystrokes; staff see code only when a student sends it; "I'm lost" is anonymous; students can see their own data |
| How does the tile know I'm stuck? | Ying Qi | Errors are caught in your sandbox; Done is your own tap; the server compares your progress with the latest checkpoint |
| What stops student code from breaking the app? | Dylan | Sandboxed iframe with an opaque origin — no access to our cookies, DOM or API; infinite loops are cut off after 5 s |
| How do you group errors? | Evan | Normalise volatile parts, group by step + key, merge by shape — walk through the unit test |
| What happens with 40 students? | Ying Qi | Updates only on change, summaries ≤ 1/s, buffered writes; load-tested with 40 simulated students |
| Where's the external API used? | Karin / Ray Yien | Telegram pings + digests; GitHub import + export via the Git Data API |
| How are tests repeatable? | Andric | In-memory MongoDB re-seeded per spec, mocked external APIs, `data-testid` selectors |
| What did you use AI for? | Any | Lower-order only (boilerplate, test data, debugging hints); listed in the README |

---

## README Skeleton

```markdown
# CodeAlong — live coding classes where nobody gets left behind
IS216 G2 · Team T4 · Andric Ang, Yew Ying Qi, Dylan Shuck, Evan Sim, Karin Tan, Tan Ray Yien

**Deployed app:** https://<url>            **Repository:** https://github.com/<org>/codealong
**Video:** https://youtu.be/<id>

## Demo accounts (password for all: pass1234)
| Role | Email |
| Prof | prof@codealong.test |
| TA | ta.ana@codealong.test |
| Student | alex@codealong.test (also bea@, chen@) |
Enrol code for IS216 G2: WAD2G2

## Set up
1. Node 20+, npm 10+
2. `npm install`
3. Copy `.env.example` to `server/.env` and fill in MONGODB_URI, JWT_SECRET (Telegram/GitHub keys optional — set EXTERNAL_APIS=mock to run without them)
4. `npm run seed`

## Run
- Development: `npm run dev` → http://localhost:5173
- Production build: `npm run build && npm start` → http://localhost:3000 (landing page at the root)

## Test
- `npx playwright install chromium`
- Unit: `npm run test:unit`
- End-to-end: `npm run test:e2e` (in-memory DB + mocked APIs; no keys needed) → `npx playwright show-report`
- Load: start a session, then `npm run simulate -- --session <joinCode> --students 40`
- Test case list: docs/06-stages-and-tests.md

## External APIs
Telegram Bot API — TA pings + digests · GitHub REST API — starter import + replay export

## Third-party libraries
(table from docs/02-tech-stack.md)

## Use of AI
(per member: what was AI-assisted, per IS216 FAQ 20)

## Known limitations
ES-module imports between lesson files; single server instance for Socket.IO
```

---

## Submission Checklist (Week 12 Friday 9:00 am — no extensions; aim for Thursday)

| ✓ | Item |
|:---:|---|
| ☐ | Zip contains **slides** (PPTX/PDF, YouTube link on slide 1), **README**, **application** source |
| ☐ | Video ≤ 12:00, ≥ 720p (1080p), unlisted or public, plays logged out |
| ☐ | Deployed link submitted the way our section faculty asked |
| ☐ | Fresh clone → README steps → app runs → `npm run test:e2e` green |
| ☐ | `.env` not in the zip/repo; demo credentials in the README |
| ☐ | Third-party credits in slides and code comments |
| ☐ | AI usage section complete |
| ☐ | Deployed app left running, untouched, until Week 18 |
