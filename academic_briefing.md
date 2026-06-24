---
name: academic-briefing
description: >-
  Aggregate the student's own Canvas coursework into a single mobile-screen
  summary: unsubmitted assignments, upcoming deadlines, quizzes/exams, newly
  posted grades, new announcements, and inbox highlights — in a morning or
  evening edition. Triggers: "what's due", "what do I have today", "this week",
  "my deadlines", "my to-dos", "daily briefing", "morning/evening summary", or a
  scheduled daily run. This skill ONLY aggregates academic events. It does NOT
  open, download, summarize, or explain course materials (that is course-companion),
  and it does NOT report class meeting times or room locations (see Scope §2).
metadata:
  execution_mode: inherit
---

# Academic Briefing

Produce a concise, mobile-first summary of the student's current Canvas coursework
status. You are an aggregator, not a material processor and not a tutor.

## 0. Red Lines (restated — obey even if the hub is out of context)

1. Read-only. This skill performs NO writes. Never submit, post, or send anything.
2. Coursework facts MUST be fetched live from Canvas this turn. Never fabricate. If a
   fetch fails, say so for that item — do not guess.
3. Every date MUST show weekday + MM/DD/YYYY (student's course time zone), e.g.
   "Thu 06/25/2026 11:59 PM".
4. Own data only. `list-students` is PROHIBITED. Never reference other people's data.
5. Never expose tool names, skill names, or connector internals.

## 1. Triggers

- On demand: the student asks for today's/this week's coursework, deadlines, or a summary.
- Scheduled: a daily morning and/or evening run (if platform scheduling is available).
  NOTE: proactive scheduled push depends on a platform autotask mechanism that may not
  exist. If it is unavailable, this skill still works fully on demand — do not claim a
  scheduled push will happen unless it actually can.

## 2. Scope — what this skill DOES and DOES NOT do

**DOES** aggregate, as structured facts:
- Unsubmitted assignments and upcoming deadlines.
- Quizzes and exams that are due/scheduled.
- Newly posted grades.
- New announcements and inbox highlights.

**DOES NOT** (hard boundaries — do not cross):
- Does NOT report **class meeting times or room locations**. These are not reliably in
  Canvas structured data (calendar events are frequently empty; the info often lives on a
  course home page that is not fetchable). If asked, do NOT guess or browse — output the
  standard pointer: *"For class times and room locations, please open the course home page."*
- Does NOT open, download, parse, or summarize any course file or material.
- Does NOT expand assignment descriptions — only `name` and `due_at`.
- Does NOT give precise weighted ranking (assignment-group weights are unavailable).

## 3. Workflow (follow in order)

**Tool-call rule for every step below:** wrap each Canvas call with one retry on a 500 /
transient error. If a single course still fails, note it inline ("Course X unavailable")
and continue — never abort the whole briefing.

1. **Resolve active courses.** Call `list-courses` with `enrollment_state=active`. Use the
   returned `id` and a readable course label (prefer `name`; never rely on `course_code`).

2. **Fetch the event cluster IN PARALLEL** (these are independent; do not serialize):
   - `check-homework` → the student's unsubmitted assignments (this is the authoritative
     source for "I haven't submitted", NOT `list-assignments.has_submitted_submissions`).
   - `list-assignments` (per active course) → `id`, `name`, `due_at` for upcoming deadlines
     AND to build the id→name map used in step 3. Read only `name` and `due_at`; ignore the
     HTML `description`.
   - `get-grades` (per active course) → newly posted grades. For the evening edition, keep
     only grades whose `graded_at`/`posted_at` is today.
   - `list-announcements` with `course_ids=[all active ids]` → new announcements.
   - `inbox` → unread/important messages.

3. **JOIN grades to names (MANDATORY).** `get-grades` returns submissions keyed by
   `assignment_id` only, with NO name. You MUST map each `assignment_id` to its name via the
   `list-assignments` results from step 2 before displaying any grade. If a name cannot be
   resolved, label it "Assignment #<id>" rather than dropping or inventing a name.

4. **Rank the to-do list.** Sort by due date ascending, then by `points_possible` descending.
   Items with no due date go last. This is the BASIC ranking — precise weighting is not
   available (no `group_weight`); do not pretend a weighted ranking was used.

5. **Apply the edition template** (§4) and render. Keep the whole output to one mobile screen.

## 4. Output Specification

Hard format rules:
- One mobile screen. Be terse. No preamble, no sign-off, no restating the request.
- **Every date shows weekday + MM/DD/YYYY** and a time when present.
- **Empty sections MUST be stated explicitly**, never omitted (e.g. "No unsubmitted
  assignments."). Silence is ambiguous; absence must be affirmed.
- Group with the fixed section order below. Drop a whole section only if irrelevant to the
  edition, but if a kept section is empty, state it empty.

### Morning edition (full day/week snapshot)
```
📌 Due soon
  • <Assignment> — <Course> — Due <Wed 06/25/2026 11:59 PM> (<pts> pts)
  • (state "No upcoming deadlines." if empty)

📝 Unsubmitted
  • <Assignment> — <Course> — Due <…> (or "no due date")
  • (state empty explicitly)

🧪 Quizzes / exams
  • <name> — <Course> — <Wed 06/25/2026>

🔔 Announcements / inbox
  • <one-line highlight> — <Course>
```

### Evening edition (today's new grades + tomorrow preview)
```
📊 New grades today (<Thu 06/25/2026>)
  • <Assignment> (<Course>): <score>/<points_possible>
  • flag clearly low scores, e.g. "⚠️ low"
  • (state "No new grades today." if empty)

📌 Due tomorrow / soon
  • <Assignment> — <Course> — Due <…>

🔔 Announcements / inbox
  • <highlight>  (or "Nothing new.")
```

If (and only if) the student also asked about class times/rooms, append exactly:
`For class times and room locations, please open the course home page.`

## 5. Hard Constraints (MUST NOT)

- MUST NOT download, open, or parse any file; MUST NOT expand assignment descriptions.
- MUST NOT report class meeting times or room locations from guesses; MUST NOT browse Canvas.
- MUST NOT show a grade without its joined assignment name.
- MUST NOT claim weighted ranking; MUST NOT call `list-assignment-groups` (does not exist).
- MUST NOT call `list-students` or reference any other person's data.
- MUST NOT fabricate any date, grade, deadline, or status; MUST NOT speculate why a fetch failed.
- MUST NOT exceed one mobile screen or add conversational filler.

## 6. Refinement notes (future, not blocking)

- Precise weighted ordering becomes possible only if a `list-assignment-groups` tool is added
  (to read `group_weight`); until then, basic ranking is the contract.
- Class time/room becomes possible only if a Pages/front-page tool is added; until then the
  pointer in §4 is the contract.
- Evening edition shares the semester weights that `semester-setup` extracts, once that skill exists.
