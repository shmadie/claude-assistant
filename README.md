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
- Slack MCP server configured in Claude Code
- Google Calendar MCP server configured in Claude Code

## Install

### 1. Clone this repo

```bash
git clone https://github.com/shmadie/claude-assistant.git
```

### 2. Copy skills into Claude Code

Claude Code loads skills from `~/.claude/skills/`. Each subdirectory with a `SKILL.md`
becomes an available `/command`.

```bash
cp -r claude-assistant/skills/daily-todos ~/.claude/skills/
cp -r claude-assistant/skills/daily-work-log ~/.claude/skills/
```

### 3. Install logseq-brain

Follow install instructions at https://github.com/jame581/LogseqBrain.
Provides `/brain-save`, `/brain-load`, `/brain-status`.

### 4. Configure

Both skills need a few personal values. Add to your `~/.claude/CLAUDE.md`:

```markdown
## Personal Assistant Config

- Slack user ID: `UXXXXXXXXXXX`  (Slack → Profile → three-dot menu → Copy member ID)
- Notes path: `~/Documents/notes/`
- GitHub username: `your-handle`
- Slack MCP tool: `mcp__slack__slack_search_public_and_private`
- Calendar MCP tool: `mcp__google-calendar__list_events`
```

Then update the `Key Constants` tables in each SKILL.md to match your values.

### 5. MCP servers needed

| MCP server | Purpose |
|---|---|
| Slack MCP | Read inbound/outbound messages for action items |
| Google Calendar MCP | Read today's events |

See [Claude Code MCP docs](https://docs.anthropic.com/en/docs/claude-code/mcp) for setup.

### 6. Notes structure

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

Compatible with [Logseq](https://logseq.com)'s default file structure.

## Daily loop

```
morning:  /daily-todos
          /brain-load   (optional — restore cross-session context)

work day: ...

evening:  /daily-work-log
          /brain-save   (optional — persist context before closing)
```
