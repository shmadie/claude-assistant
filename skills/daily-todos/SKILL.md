---
name: daily-todos
description: >
  Generate your daily todo list. Reads recent journals, checks calendar, and scans
  Slack to find new action items. Adds new items to the master Todos.md (auto-assigned
  priority), writes a lightweight daily audit file, and presents the full master list.
  Triggered by: "todos for today", "give me my todo list", "generate todos", "/daily-todos".
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Glob, mcp__google-calendar__list_events, mcp__slack__search
---

# daily-todos

Discover new action items from journals, calendar, and Slack. Add them to the single
master `Todos.md` with auto-assigned priority. Write a lightweight daily audit file.
Present the full master list as the daily picture.

## Usage

```
/daily-todos          # generate for today
/daily-todos DATE     # generate for a specific date (YYYY-MM-DD)
```

---

## Configuration

Set these in your `CLAUDE.md` or at the top of the skill before first use:

| Constant | Description | Example |
|----------|-------------|---------|
| `SLACK_USER_ID` | Your Slack user ID (Settings → Profile → copy from URL) | `U012AB3CD` |
| `NOTES_PATH` | Root path for journals + pages | `~/Documents/notes/` |
| `SLACK_MCP_TOOL` | Your Slack MCP search tool name | `mcp__slack__slack_search_public_and_private` |
| `CALENDAR_MCP_TOOL` | Your calendar MCP list-events tool name | `mcp__google-calendar__list_events` |

---

## Fast mode

If the user says "use local data", "just show me", "what's left", or similar — **skip Steps 2–4** entirely. Read master Todos.md and present it directly. Do not make calendar or Slack calls.

---

## Step 0 — Resolve date

Determine `TARGET_DATE`:
- If an argument was passed, parse it as `YYYY-MM-DD`
- Otherwise use today's date

Compute:
- `JOURNAL_FILE`: `journals/YYYY_MM_DD.md` (underscores)
- `AUDIT_FILE`: `pages/YYYY-MM-DD Todos.md` (dashes)
- `MASTER_TODOS`: `pages/Todos.md`
- `SINCE_DATE`: last modified date of `Todos.md`, or 3 days ago if not determinable

---

## Step 1 — Load current state (parallel)

Run all reads in parallel:

### 1a. Recent journals

```bash
ls $NOTES_PATH/journals/ | sort | tail -5
```

Read the last 2–3 journal files. Extract:
- Inline `TODO` items added during meetings
- Meeting outcomes that imply follow-up actions
- Initiative context and ownership decisions

### 1b. Master Todos.md

Read `pages/Todos.md` if it exists. Note every existing item (text + priority bucket) to
avoid duplicates when adding new items.

If it does not exist yet, it will be created in Step 6.

---

## Step 2 — Check today's calendar (parallel with Step 1)

Call your calendar MCP list-events tool:
- `time_min`: `TARGET_DATE T00:00:00-07:00`
- `time_max`: `TARGET_DATE T23:59:00-07:00`
- `max_results`: 25

Extract for the **audit file only** (meetings do NOT go in master Todos.md):
- Work meetings: summary, start time, key attendees (first 3–4)
- On-call shifts
- OOO / travel blocks

Skip personal blocks (gym, lunch, etc.).

---

## Step 3 — Slack inbound (parallel with Steps 1–2)

Search for messages sent TO you since `SINCE_DATE`:
```
to:<@YOUR_SLACK_USER_ID> after:SINCE_DATE
channel_types: im,mpim
limit: 20
sort: timestamp asc
```

Look for new action items:
- Requests or asks not yet in master
- Unanswered questions that need a response
- Review or approval requests

---

## Step 4 — Slack outbound (parallel with Steps 1–2)

Search for messages FROM you since `SINCE_DATE`:
```
from:<@YOUR_SLACK_USER_ID> after:SINCE_DATE
channel_types: im,mpim,public_channel,private_channel
limit: 20
sort: timestamp asc
```

Look for new action items:
- Commitments made not yet in master ("will review", "I'll ping", "I'll send")
- Questions you asked that need a follow-up if unanswered
- PRs shared that need reviews or approvals

---

## Step 5 — Identify new items

Compare discovered items against existing master Todos.md entries.

**Deduplicate:** skip any item already captured in the master (fuzzy match on subject/person/PR number).

**New items only** — anything not already tracked. For each new item note:
- Description (concise, action-oriented)
- Source (journal date, Slack sender, PR link, etc.)
- Any relevant context or links

---

## Step 6 — Auto-assign priority + type

### Priority emoji (contextual importance, NOT time-based)

| Emoji | Level | Meaning |
|-------|-------|---------|
| 🚨 | P0 | Blocking someone, major risk if missed, critical commitment |
| 🔴 | P1 | High importance — significant commitment or strategic impact |
| 🟡 | P2 | Medium — important but not urgent, no one is waiting |
| 🔵 | P3 | Low — nice to do, limited impact |
| ⚪ | P4 | Someday — exploratory, no real consequence if never done |

**Priority is about importance/impact, NOT deadlines.** An overdue item can be P3 if it has low impact. A "due this month" item can be P0 if missing it would be critical.

Rules:
- 🚨 P0: blocking another person's work, active incident follow-up, explicitly critical to a manager/director, or missing it has major consequences
- 🔴 P1: explicit commitment made, significant strategic impact, PR review requested with a stated deadline
- 🟡 P2: follow-up with no clear dependency, coordination tasks, medium-impact writing
- 🔵 P3: low-consequence reading, research, or tasks with no active dependency
- ⚪ P4: exploratory, speculative, or "someday" ideas

### Type emoji

| Emoji | Type | Typical effort |
|-------|------|----------------|
| 👀 | PR review / approval | Quick (< 30 min) |
| 💬 | Reply / unblock / answer | Quick (< 15 min) |
| 📞 | Sync / outreach / schedule | Medium |
| ✍️ | Write / design / review doc | Deep (60+ min) |
| 📖 | Read / research | Deep (60+ min) |
| 🔬 | Investigate / explore | Variable |

### Temporal section assignment (deadline, NOT importance)

Place each item in the correct temporal section:
- **🔥 Overdue** — deadline has passed or commitment was made and not kept
- **📅 Due This Week** — has a stated or implied deadline within 7 days
- **🗓️ Due This Month** — needed within ~30 days, no immediate deadline
- **🧊 No Deadline** — backlog, research, someday items

### Outreach todo quality rule
For any 📞 sync/outreach item, the description MUST include what to discuss — not just who to contact. Bad: "Reach out to Alice". Good: "Reach out to Alice — confirm X for Y project". If context is unknown, flag with `⚠️ context unclear`.

### Uncertainty flag
If priority is ambiguous, assign the most likely level and append:
`⚠️ uncertain priority`

Collect all uncertain items — present them for confirmation in Step 9.

---

## Step 7 — Update master Todos.md

Read current `pages/Todos.md` (or create it if missing).

**Format:**
```
tags:: todos, master
updated:: [[YYYY-MM-DD]]

- ## Legend
- Priority: 🚨 P0 blocking/critical · 🔴 P1 high · 🟡 P2 medium · 🔵 P3 low · ⚪ P4 someday
- Type: 👀 review · 💬 reply/unblock · 📞 sync/outreach · ✍️ write/design · 📖 read · 🔬 investigate

- ## 🔥 Overdue
- TODO <priority> <type> <item>
  - <context / link>

- ## 📅 Due This Week
- TODO <priority> <type> <item>
  - <context / link>

- ## 🗓️ Due This Month
- TODO <priority> <type> <item>
  - <context / link>

- ## 🧊 No Deadline
- TODO <priority> <type> <item>
  - <context / link>

- ## ✅ Done
- DONE [YYYY-MM-DD] <priority> <type> <item>
```

Each line format: `TODO <priority emoji> <type emoji> <description>`

For each new item from Step 6:
- Append it under the correct temporal section
- Include source context as a sub-bullet (Slack sender, journal date, PR link)

Use `Edit` to append within the correct section — do not rewrite the whole file.
Update the `updated::` frontmatter field to TARGET_DATE.

---

## Step 8 — Write daily audit file

Write `pages/YYYY-MM-DD Todos.md` as a lightweight reference:

```
tags:: todos, daily
date:: [[YYYY-MM-DD]]
master:: [[Todos]]

- ## On-call Today
- <on-call shifts from calendar>
- ## Calendar Today
- HH:MMam — <meeting name> (<attendees>)
- ## New Items Added to Master
- <each new item added, with assigned priority>
- ## Flagged for Priority Review
- <items marked ⚠️ uncertain priority>
```

Then link from journal: prepend `- [[YYYY-MM-DD Todos]]` to `journals/YYYY_MM_DD.md`
(create the journal file if it doesn't exist).

---

## Step 9 — Present to user

Output in two parts:

### Part 1 — Master todo list (full picture)

```
## Todos — [Weekday] [Month] [Day]

### P0
- item (source)

### P1
- item (source)

### Backlog
- item (source)
```

### Part 2 — What's new + needs your input

```
### Added today
- item → P0/P1/Backlog

### ⚠️ Priority uncertain — please confirm
- item (suggested: P1) — reason it's uncertain
```

If nothing is uncertain, skip Part 2.
