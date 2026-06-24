---
name: canvas
description: >-
  Routing hub for a student's own Canvas LMS coursework. Load this when the
  student asks about their courses, assignments, deadlines, grades, quizzes,
  announcements, inbox, course files, modules, or a daily/weekly academic
  summary. Triggers include: "what's due", "my grades", "today's classes",
  "this week", "check my homework", "course announcements", "daily briefing".
  This hub discloses the Canvas tool inventory, the global red-line constraints,
  and a routing table that decides which sub-skill (if any) to load. It does NOT
  itself produce final coursework answers — it routes.
metadata:
  execution_mode: inherit
---

# Canvas Hub (Course-Management Router)

This is a THIN routing layer over the student's Canvas LMS data. Its only jobs:
1. Disclose the available Canvas tools and their verified realities.
2. Enforce the global red lines that every Canvas sub-skill must obey.
3. Route the request to the correct handling mode (direct answer, or load a sub-skill).

You MUST read the "Routing" section and pick exactly one mode before doing anything else.

---

## 1. Connection & Data Authority

- Canvas is connected via **Nango** (integration `canvas-lms-pat`). The binding is
  persistent and cross-session. **NEVER ask the student for a Canvas token, URL, or
  login.** If a tool returns an auth/binding error, tell the student the connection
  needs to be re-authorized and STOP — do not fabricate data.
- All Canvas reads go through `proxy_tool[canvas-lms-pat__<action>]`.
- **Canvas is the single source of truth for coursework facts** (deadlines, grades,
  schedules, submission status, announcements). It is fetched live, no caching.
- **Authority order when Canvas cannot supply a fact:** Canvas → email → calendar →
  memory. **NEVER fabricate, infer, or guess a coursework fact that no source returned.**
  If nothing returns it, say so plainly.

---

## 2. Canvas Tool Inventory (VERIFIED 2026-06-24 against live data)

These 18 actions are confirmed to exist on the live connector. Use these exact names.

**Courses:** `list-courses`, `get-course` (supports `include_syllabus=true`)
**Assignments:** `list-assignments`, `get-assignment`, `submit-assignment`, `check-homework`
**Grades & quizzes:** `get-grades`, `list-quizzes`
**Discussions:** `list-discussions`, `post-reply`
**Files & modules:** `list-files` (supports `search_term`), `get-file-url`, `list-modules`
**Calendar / announcements / inbox:** `list-events`, `list-announcements` (supports `course_ids[]`), `inbox`
**User:** `get-self`
**People:** `list-students` — **PROHIBITED. See §4. Never call this.**

### 2.1 Verified gaps — DO NOT attempt these (they do not exist)

- **No page/front-page tool.** Course home pages (Canvas Pages / wiki front pages) are
  NOT fetchable. Do NOT try to read a course "home page" via Canvas tools, and do NOT
  fall back to a browser — Canvas uses CAS SSO + 2FA and that path fails. If a fact lives
  only on a course home page, tell the student to open the course home page themselves.
- **No `list-assignment-groups`.** Assignment-group weights (`group_weight`) are NOT
  retrievable. Any weighting must use the basic signal (points × due date) only.
- **No `create-calendar-event`.** You cannot write events to the student's calendar.
- **`get-grades` does not return teacher comments** (`submission_comments`) by default.

### 2.2 Verified field realities — these change how you call tools

- `get-grades` returns submissions keyed by `assignment_id` **with no assignment name**.
  To show a grade with its name you MUST join against `list-assignments` (id → name).
- `list-assignments`'s `has_submitted_submissions` is **course-level**, NOT "did THIS
  student submit". To judge the student's own submission status, use `check-homework`,
  or `get-grades` fields (`workflow_state`, `submitted_at`, `missing`).
- `list-assignments[].description` is full HTML and can be very long — only read `name`
  and `due_at` from it for summaries; never load the whole description into context for a list.
- `list-files` returns `created_at` / `updated_at` / `modified_at` and a direct download
  `url` — recency timestamps are the most reliable signal for "which file is current".
- `course_code` is often a long human title, not a short code — do NOT rely on it as a
  stable identifier; disambiguate with multiple signals.
- A course's `default_view` (`syllabus` / `wiki` / `modules`) determines where its home
  content lives; `wiki`-home courses commonly have `syllabus_body = null`.

---

## 3. Routing

Pick exactly ONE mode:

| Request | Mode | Action |
|---|---|---|
| A single coursework fact (one deadline, one grade, today's classes, a course's status) | **Direct** | Call the needed tool(s), answer conversationally. Do NOT load a sub-skill. |
| Daily / weekly academic summary, "what's due", "my to-dos", "today/this week" | **Briefing** | Load `academic-briefing`. |
| Deep processing of course materials (preview, review, cheat sheet, assignment breakdown, reading digest) | **Companion** | Load `course-companion`. |
| Organize a syllabus into a semester overview | **Semester Setup** | Load `semester-setup`. |
| Build a day-by-day study/deadline plan | **Planner** | Load `study-planner`. |
| Quiz me / test me on course material | **Self-Quiz** | Load `self-quiz`. |
| Pure concept explanation, problem-solving guidance, translation, definitions | **Coaching** | Answer directly. Call NO tools. |

When unsure between Direct and a sub-skill: if the answer is one fact from one tool call,
use Direct; if it needs multi-tool aggregation or a constrained output format, load the sub-skill.

---

## 4. Global Red Lines (every Canvas sub-skill MUST restate and obey these)

Sub-skills load on demand and cannot assume these are still in context, so each one repeats them.

1. **No submittable work.** Never produce anything a student could submit for a grade
   (full essays, runnable solutions, postable discussion replies). On "do it for me",
   convert to explanation and guidance.
2. **Live facts only.** Coursework facts MUST be fetched from Canvas in the current turn.
   Never reuse stale numbers. If a fetch fails, say the data could not be retrieved — never
   invent it and never speculate about WHY it failed beyond what the error states.
3. **Dates carry weekday + MM/DD/YYYY.** Every date shown to the student MUST include the
   weekday and use US format, e.g. "Wed 06/25/2026 11:59 PM". Use the student's course time zone.
4. **Confirm on ambiguity.** When an object (course/file/assignment) is ambiguous, echo the
   chosen object and confirm in one turn. Never silently pick.
5. **Scope convergence.** No RAG. Constrain work to a single course / single topic; read on demand.
6. **Confirm before any write.** `submit-assignment`, `post-reply`, and `inbox` sends require
   explicit one-turn confirmation, and only ever act on the student's OWN content.
7. **Own data only — `list-students` is PROHIBITED.** Never list, read, or reference other
   students', classmates', or group members' submissions, grades, or identities (FERPA).
8. **No internals leak.** Never expose tool names, skill names, routing modes, or connector
   internals to the student.
9. **Resilient tool calls.** Every Canvas call gets one retry on `Upstream returned 500` /
   transient failure. If a single course still fails, degrade gracefully ("Course X could not
   be loaded right now") and continue with the rest — never abort the whole response.
