# ccq

CLI tool for querying and analyzing Claude Code data.

## Installation

```bash
cargo install --path .
```

## Configuration

Set the Claude data directory:

```bash
# Via environment variable
export CLAUDE_DATA_DIR=~/.claude

# Or via command line flag
ccq --data-dir ~/.claude <command>
```

## Commands

### prompts

Extract user prompts from history.

```bash
ccq prompts
ccq prompts --project myproject --limit 10
ccq prompts --since 2024-01-01 --until 2024-12-31
```

### sessions

List and browse sessions.

```bash
ccq sessions
ccq sessions --detailed --sort-by size
```

### stats

Display usage statistics.

```bash
ccq stats
ccq stats --group-by date
```

### search

Full-text search across all data.

```bash
ccq search "error handling"
ccq search "TODO" --scope prompts
ccq search "function\s+\w+" --regex --case-sensitive
ccq search "bug" -B 2 -A 2  # with context lines
```

### todos

List todos and their status.

```bash
ccq todos
ccq todos --status pending
ccq todos --agent main
```

### duplicates

Find repeated/similar prompts using fuzzy matching.

```bash
ccq duplicates
ccq duplicates --threshold 0.9 --min-count 3
ccq duplicates --show-variants
```

### query

Execute jq-style queries on data sources.

```bash
ccq query '.[] | select(.type == "human")' history
ccq query '.[] | .content' transcripts --file-pattern abc123
```

Data sources: `history`, `transcripts`, `stats`, `todos`

### sql

Execute SQL queries on Claude Code data using GlueSQL.

```bash
# Basic queries
ccq sql "SELECT * FROM history LIMIT 10"
ccq sql "SELECT display, project FROM history WHERE project LIKE '%myproject%'"
ccq sql "SELECT COUNT(*) as total FROM history"

# Aggregations
ccq sql "SELECT project, COUNT(*) as count FROM history GROUP BY project"

# Output formats (note: -f flag goes before subcommand)
ccq -f json sql "SELECT * FROM history LIMIT 5"
ccq -f jsonl sql "SELECT * FROM history LIMIT 5"
```

#### Write Operations

Write operations (INSERT, UPDATE, DELETE) require explicit flags for safety:

```bash
# Preview changes without modifying (dry run)
ccq sql --dry-run "DELETE FROM history WHERE timestamp < 1704067200000"

# Execute write operation (requires --write flag)
ccq sql --write "UPDATE history SET project = 'archived' WHERE project = 'old-project'"
ccq sql --write "DELETE FROM history WHERE timestamp < 1704067200000"
```

#### Available Tables

| Table | Source | Description |
|-------|--------|-------------|
| `history` | `~/.claude/history.jsonl` | User prompts and commands |
| `stats` | `~/.claude/stats-cache.json` | Usage statistics |
| `transcripts` | `~/.claude/transcripts/*.jsonl` | All conversation messages (virtual table) |
| `todos` | `~/.claude/todos/*.json` | Task lists from all sessions (virtual table) |

#### Virtual Table Metadata

The `transcripts` and `todos` tables merge multiple files and include metadata columns:

**transcripts columns:** `_source_file`, `_session_id`, `type`, `timestamp`, `content`, `tool_name`, `tool_input`, `tool_output`

**todos columns:** `_source_file`, `_workspace_id`, `_agent_id`, `content`, `status`, `activeForm`

#### Practical Query Examples

**Productivity Insights:**
```bash
# Prompt breakdown: how much is commands vs real work?
ccq sql "SELECT COUNT(*) as total,
         SUM(CASE WHEN display LIKE '/%' THEN 1 ELSE 0 END) as commands,
         SUM(CASE WHEN display LIKE '[Pasted%' THEN 1 ELSE 0 END) as pastes
         FROM history"

# Find wasted effort: undo/revert patterns
ccq sql "SELECT display FROM history
         WHERE display LIKE '%undo%' OR display LIKE '%revert%'"

# Low-value prompts: repeated short confirmations
ccq sql "SELECT display, COUNT(*) as cnt FROM history
         WHERE LENGTH(display) < 30 AND display NOT LIKE '/%'
         GROUP BY display ORDER BY cnt DESC"

# Top projects by activity
ccq sql "SELECT project, COUNT(*) as prompts FROM history
         GROUP BY project ORDER BY prompts DESC"
```

**Todo Completion:**
```bash
# Completion rate with percentages
ccq sql "SELECT status, COUNT(*) as count,
         ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM todos)) as pct
         FROM todos GROUP BY status"

# Pending todos by workspace (find stale work)
ccq sql "SELECT _workspace_id, COUNT(*) as pending FROM todos
         WHERE status = 'pending' GROUP BY _workspace_id ORDER BY pending DESC"
```

**Tool Usage:**
```bash
# Most used tools
ccq sql "SELECT tool_name, COUNT(*) as uses FROM transcripts
         WHERE tool_name IS NOT NULL GROUP BY tool_name ORDER BY uses DESC"

# Read vs Edit vs Write ratio
ccq sql "SELECT tool_name, COUNT(*) as cnt FROM transcripts
         WHERE tool_name IN ('read', 'edit', 'write') GROUP BY tool_name"

# Largest sessions by tool calls
ccq sql "SELECT _session_id, COUNT(*) as tool_calls FROM transcripts
         WHERE type='tool_use' GROUP BY _session_id ORDER BY tool_calls DESC"
```

**Command Usage:**
```bash
# Most used slash commands
ccq sql "SELECT display, COUNT(*) as cnt FROM history
         WHERE display LIKE '/%' GROUP BY display ORDER BY cnt DESC"

# Git operations
ccq sql "SELECT display FROM history
         WHERE display LIKE '%commit%' OR display LIKE '%push%'"
```

**Search & Discovery:**
```bash
# Find work on a topic
ccq sql "SELECT _session_id, timestamp, content FROM transcripts
         WHERE type='user' AND content LIKE '%authentication%' LIMIT 10"

# Bug/fix related work
ccq sql "SELECT project, display FROM history
         WHERE display LIKE '%error%' OR display LIKE '%bug%' OR display LIKE '%fix%'"
```

## Output Formats

```bash
ccq prompts -f table   # default, human-readable tables
ccq prompts -f json    # JSON array
ccq prompts -f jsonl   # JSON lines (one per line)
ccq prompts -f raw     # raw output
```

## License

MIT
