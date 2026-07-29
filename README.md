# claude-assistant

Personal assistant skills for Claude Code. Daily rhythm in 3 commands.

## Skills

| Skill | Command | When |
|---|---|---|
| [daily-todos](skills/daily-todos/SKILL.md) | `/daily-todos` | Morning — find new action items, update master todo list |
| [daily-work-log](skills/daily-work-log/SKILL.md) | `/daily-work-log` | Evening — audit what got done, close todos, write log |
| logseq-brain (plugin) | `/brain-save` `/brain-load` | Session boundaries — persist context across sessions |

## Setup

### 1. Install skills

Copy `skills/` into your Claude Code skills directory:

```bash
cp -r skills/daily-todos ~/.claude/skills/
cp -r skills/daily-work-log ~/.claude/skills/
```

### 2. Install logseq-brain plugin

logseq-brain is a public Claude Code plugin. Install via skillsmith:

```
/skillsmith install logseq-brain
```

Or see: https://github.com/anthropics/claude-code-skills (community registry)

### 3. Configure

Both skills require a few personal values. Set them in your `~/.claude/CLAUDE.md`:

```markdown
## Personal Assistant Config

- Slack user ID: `UXXXXXXXXXXX`  (Settings → Profile → three-dot menu → Copy member ID)
- Notes path: `~/Documents/notes/`
- GitHub username: `your-handle`
- Slack MCP tool: `mcp__slack__slack_search_public_and_private`
- Calendar MCP tool: `mcp__google-calendar__list_events`
```

Then update the `Key Constants` tables in each SKILL.md to match.

### 4. MCP servers needed

| MCP server | Purpose |
|---|---|
| Slack MCP | Read inbound/outbound messages for action items |
| Google Calendar MCP | Read today's events |

See Claude Code docs for MCP server setup.

### 5. Notes structure

Skills expect this file layout under your notes path:

```
notes/
  journals/
    YYYY_MM_DD.md     # daily journal (underscores)
  pages/
    Todos.md          # master todo list (auto-created on first run)
    YYYY-MM-DD Todos.md       # daily audit snapshot
    YYYY-MM-DD Work Log.md    # daily work log
```

Compatible with Logseq's default file structure.

## Daily loop

```
morning:  /daily-todos
          /brain-load   (optional — restore cross-session context)

work day: ...

evening:  /daily-work-log
          /brain-save   (optional — persist context before closing)
```
