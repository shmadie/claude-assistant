---
name: daily-work-log
description: >
  Generate a daily work log — an audit of what actually got done on a given day.
  Reads the journal, daily audit file, calendar, and Slack to infer completed work. Writes
  a work log page, moves completed items to Done in master Todos.md (with date), and links
  from the journal. Triggered by: "work log for today", "what did I do today", "log what
  I did", "generate work log", "/daily-work-log".
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Glob, mcp__google-calendar__list_events, mcp__slack__search
---

# daily-work-log

Generate a detailed audit of what you actually accomplished on a given day.
Cross-references the journal, master Todos.md, calendar, and Slack to infer completion.
Writes a work log page and moves completed items to `## Done` in master Todos.md.

## Usage

```
/daily-work-log          # log for today
/daily-work-log DATE     # log for a specific date (YYYY-MM-DD)
```

---

## Configuration

Set these in your `CLAUDE.md` or at the top of the skill before first use:

| Constant | Description | Example |
|----------|-------------|---------|
| `SLACK_USER_ID` | Your Slack user ID (Settings → Profile → copy from URL) | `U012AB3CD` |
| `NOTES_PATH` | Root path for journals + pages | `~/Documents/notes/` |
| `GITHUB_USERNAME` | Your GitHub username (for PR review detection) | `your-handle` |
| `SLACK_MCP_TOOL` | Your Slack MCP search tool name | `mcp__slack__slack_search_public_and_private` |
| `CALENDAR_MCP_TOOL` | Your calendar MCP list-events tool name | `mcp__google-calendar__list_events` |

---

## Step 0 — Resolve date

Determine `TARGET_DATE`:
- If an argument was passed, parse it as `YYYY-MM-DD`
- Otherwise use today's date

Compute:
- `JOURNAL_FILE`: `journals/YYYY_MM_DD.md`
- `WORK_LOG_PAGE`: `pages/YYYY-MM-DD Work Log.md`
- `AUDIT_FILE`: `pages/YYYY-MM-DD Todos.md`
- `MASTER_TODOS`: `pages/Todos.md`

---

## Step 1 — Load state (parallel)

### 1a. Journal for TARGET_DATE
Read `journals/YYYY_MM_DD.md`. Extract:
- Meeting notes and outcomes captured during the day
- Inline `TODO` items (new captures mid-day)
- Work output: docs written, decisions made, investigations done
- OOO indicators

### 1b. Master Todos.md
Read `pages/Todos.md`. Note the full list of P0/P1/Backlog items — these are the
candidates to be marked done and moved to `## Done`.

### 1c. Daily audit file
Read `pages/YYYY-MM-DD Todos.md` if it exists. Use it to cross-reference what was
expected for that day.

---

## Step 2 — Check calendar (parallel with Step 1)

Call your calendar MCP list-events tool:
- `time_min`: `TARGET_DATE T00:00:00-07:00`
- `time_max`: `TARGET_DATE T23:59:00-07:00`
- `max_results`: 25

Extract work-relevant events only:
- Work meetings: summary, start time, key attendees
- On-call shifts
- OOO / travel blocks

Skip personal blocks (gym, lunch, etc.).

---

## Step 3 — GitHub PR status (parallel with Steps 1–2)

For every item in master Todos.md that contains a GitHub PR URL (e.g. `github.com/.../pull/NNN`):

```bash
gh pr view <NNN> --repo <owner/repo> --json state,reviews,mergedAt,closedAt,title
```

A PR todo is considered DONE if:
- `state` is `MERGED` or `CLOSED`
- OR `reviews` contains an approving review from you and the PR has no outstanding changes-requested

Note: for PRs where you are the *reviewer* (not author), DONE = you left an approving review. For PRs where you are the *author*, DONE = the PR is merged or has received the required approvals.

Collect results — use them as strong evidence in Step 5 (overrides Slack inference).

---

## Step 4 — Slack inbound (parallel with Steps 1–3)

Search for messages TO you on TARGET_DATE:
```
to:<@YOUR_SLACK_USER_ID> after:YYYY-MM-DD before:YYYY-MM-DD+1
channel_types: im,mpim
limit: 20
sort: timestamp asc
```

Look for:
- Questions you answered (conversation closed)
- Review requests fulfilled
- Decisions made

---

## Step 5 — Slack outbound (parallel with Steps 1–3)

Search for messages FROM you on TARGET_DATE:
```
from:<@YOUR_SLACK_USER_ID> after:YYYY-MM-DD before:YYYY-MM-DD+1
channel_types: im,mpim,public_channel,private_channel
limit: 20
sort: timestamp asc
```

Look for:
- Actions taken: sent a doc, shared a link, posted a review, pinged someone
- Commitments fulfilled: "here's the doc", "I reviewed", "done"
- Discussions advanced: gave direction, unblocked someone, reached alignment
- Scheduled messages sent (indicates a planned task executed)

Skip: bot messages, purely social messages, emoji-only reactions.

---

## Step 6 — Synthesize

### Completion inference
An item from master Todos.md is considered DONE if any of the following are true:
- Journal notes confirm it happened
- Slack outbound shows the action was taken
- Slack thread shows resolution (conversation concluded, question answered)

### New work discovered
Items done that were NOT in master Todos.md (ad-hoc work, untracked tasks). Note these
in the work log under "Work Done" — do NOT add them to master Todos.md (they're already done).

### OOO handling
If calendar shows OOO or no work events and Slack is quiet:
- Note that the day was OOO / limited availability
- Still capture any async work that did happen

---

## Step 7 — Write work log page

Write `pages/YYYY-MM-DD Work Log.md`:

```
tags:: work-log, daily
date:: [[YYYY-MM-DD]]

- ## Meetings & Syncs
- HH:MMam — <meeting name> (<key attendees>)
  - <outcome, decision, or key discussion point>
- ## Work Done
- <item> — <detail of what was accomplished>
- ## Todos Closed
- DONE [YYYY-MM-DD] <item from master Todos moved to Done>
- ## Notes
- <OOO, on-call incidents, blockers, anything notable>
```

**Section rules:**
- **Meetings & Syncs**: work meetings with notes or outcomes only. Skip personal/declined.
- **Work Done**: async work — PRs reviewed, docs written, Slack threads resolved, investigations, decisions made. Include ad-hoc work not in master Todos.
- **Todos Closed**: items from master Todos confirmed done today. These will be moved to `## Done` in master.
- **Notes**: OOO, freeze, incident, anything that shaped the day.

**Link policy:**
- Any entry that mentions sending, sharing, reviewing, or referencing a document MUST include the link.
- Prefer permanent links (Google Doc, Confluence, GitHub) over ephemeral ones (Slack messages).
- PR entries must include the GitHub PR URL.

---

## Step 8 — Update master Todos.md

The master `Todos.md` uses this format for each item:
```
- TODO <priority emoji> <type emoji> <description>
```

Temporal sections: `## 🔥 Overdue` · `## 📅 Due This Week` · `## 🗓️ Due This Month` · `## 🧊 No Deadline`

Priority emoji: 🚨 P0 · 🔴 P1 · 🟡 P2 · 🔵 P3 · ⚪ P4
Type emoji: 👀 review · 💬 reply/unblock · 📞 sync/outreach · ✍️ write/design · 📖 read · 🔬 investigate

For each item confirmed DONE in Step 5:

1. **Remove** the full item line (and its sub-bullets) from its current temporal section in `pages/Todos.md`
2. **Append** it to the `## ✅ Done` section, preserving the priority + type emoji:
   ```
   - DONE [YYYY-MM-DD] <priority emoji> <type emoji> <item text>
   ```

Use `Edit` for targeted replacements — do not rewrite the whole file. Always edit line by line — never rewrite the whole file, as this risks data loss.
Only move items with clear evidence of completion. When in doubt, leave in place.

Also update the `updated::` frontmatter to TARGET_DATE.

### Misplaced DONE items
You may directly edit Todos.md and change `TODO` → `DONE` inline (without moving the item). After writing the work log, scan all temporal sections (🔥/📅/🗓️/🧊) for any lines starting with `- DONE`. For each one found:
1. Remove it from the temporal section
2. Append it to `## ✅ Done` with `[YYYY-MM-DD]` date stamp (use today's date if none present)

### Close journal references
After moving items to Done, search journals for any matching `TODO` references to those items:
```bash
grep -rn "TODO.*<keyword>" $NOTES_PATH/journals/
```
For each matching journal line, change `TODO` → `DONE [YYYY-MM-DD]` in place.

---

## Step 9 — Link from journal

If `journals/YYYY_MM_DD.md` exists: prepend `- [[YYYY-MM-DD Work Log]]` as the first
bullet if not already present.

If it does not exist: create it with:
```
-
- [[YYYY-MM-DD Work Log]]
```

---

## Step 10 — Present summary to user

```
## Work Log — [Weekday] [Month] [Day]

### Meetings
- HH:MMam — Meeting name: key outcome

### Work Done
- ...

### Todos Closed (N)
- item [moved to Done in Todos.md]

### Still Open
- P0: ...
- P1: ...
```

Call out:
- How many items were moved to Done in master Todos.md
- Remaining P0 items (anything still open at P0 is worth flagging)
- OOO or incident notes if applicable
