# claude-assistant

Personal assistant skills for Claude Code. Daily rhythm in 3 commands.

## Skills

| Skill | Command | When |
|---|---|---|
| [daily-todos](skills/daily-todos/SKILL.md) | `/daily-todos` | Morning — find new action items, update master todo list |
| [daily-work-log](skills/daily-work-log/SKILL.md) | `/daily-work-log` | Evening — audit what got done, close todos, write log |
| [logseq-brain](https://github.com/jame581/LogseqBrain) | `/brain-save` `/brain-load` | Session boundaries — persist context across sessions |

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed and authenticated
- [GitHub CLI (`gh`)](https://cli.github.com) installed and authenticated (used by `daily-work-log` to check PR states)
- Slack MCP server configured in Claude Code
- Google Calendar MCP server configured in Claude Code

See [Claude Code MCP docs](https://docs.anthropic.com/en/docs/claude-code/mcp) for MCP server setup.

## Install

### 1. Clone this repo

```bash
git clone https://github.com/shmadie/claude-assistant.git
cd claude-assistant
```

### 2. Copy skills into Claude Code

Claude Code loads skills from `~/.claude/skills/`. Each subdirectory with a `SKILL.md`
becomes an available `/skill-name` slash command.

```bash
cp -r skills/daily-todos ~/.claude/skills/
cp -r skills/daily-work-log ~/.claude/skills/
```

**Restart Claude Code** (or start a new session) after copying — skills are discovered at startup.

### 3. Install logseq-brain

logseq-brain is a Claude Code skills plugin that provides `/brain-save`, `/brain-load`, and `/brain-status`.

```bash
git clone https://github.com/jame581/LogseqBrain.git
# follow the install instructions in that repo's README
# the skills end up in ~/.claude/skills/logseq-brain/
```

After install, `/brain-save` and `/brain-load` will be available as slash commands in Claude Code, same as the skills above.

### 4. Find your MCP tool names

The skills call your Slack and Google Calendar MCP servers by their registered tool names. These names depend on how you configured the servers.

To find your exact tool names, run this in a Claude Code session:

```
what MCP tools do I have available?
```

Look for tools matching `slack` and `calendar`. Common names:
- Slack: `mcp__slack__slack_search_public_and_private`
- Google Calendar: `mcp__google-calendar__list_events`

You'll use these in the next two steps.

### 5. Update SKILL.md frontmatter

Each SKILL.md has an `allowed-tools` line in its frontmatter that whitelists which tools the skill can call. Update both files with your actual MCP tool names:

**`~/.claude/skills/daily-todos/SKILL.md`** — change line 9:
```yaml
allowed-tools: Bash, Read, Write, Edit, Glob, YOUR_CALENDAR_TOOL, YOUR_SLACK_TOOL
```

**`~/.claude/skills/daily-work-log/SKILL.md`** — change line 10:
```yaml
allowed-tools: Bash, Read, Write, Edit, Glob, YOUR_CALENDAR_TOOL, YOUR_SLACK_TOOL
```

### 6. Configure in CLAUDE.md

Add this block to `~/.claude/CLAUDE.md`:

```markdown
## Personal Assistant Config

- Slack user ID: `UXXXXXXXXXXX`
- Notes path: `/Users/yourname/Documents/notes`
- GitHub username: `your-github-handle`
- Slack MCP tool: `mcp__slack__slack_search_public_and_private`
- Calendar MCP tool: `mcp__google-calendar__list_events`
- Timezone offset: `+00:00`
```

**Finding your Slack user ID:** Open Slack → click your name in the sidebar → three-dot menu → Copy member ID. Looks like `U012AB3CD`.

Claude Code reads `CLAUDE.md` as context on every session — the skills pick up these values automatically. No editing of SKILL.md files is needed beyond step 5.

### 7. Notes structure

Skills expect this layout under your notes path:

```
notes/
  journals/
    YYYY_MM_DD.md              # daily journal (underscores in filename)
  pages/
    Todos.md                   # master todo list (auto-created on first run)
    YYYY-MM-DD Todos.md        # daily audit snapshot
    YYYY-MM-DD Work Log.md     # daily work log
```

Compatible with [Logseq](https://logseq.com)'s default file structure. If you use Logseq, point `Notes path` at your graph root.

## Daily loop

```
morning:  /daily-todos
          /brain-load   (optional — restore cross-session context)

work day: ...

evening:  /daily-work-log
          /brain-save   (optional — persist context before closing)
```

## Verify it's working

After setup, run `/daily-todos`. A working run looks like:

```
## Todos — [Today's date]

Checked calendar: N events
Scanned Slack: N messages since [date]

### P0
(none)

### P1
- ...
```

If the Slack or calendar sections are missing entirely (no "Checked..." line), the MCP tool names in your SKILL.md frontmatter likely don't match your configured servers. Re-check step 5.
