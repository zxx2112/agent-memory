---
name: agent-memory
description: Persistent memory for storing, searching, updating, and organizing durable agent or project knowledge across conversations. Use when saving reusable findings, project context, gotchas, decisions, solutions, resumable work, or when checking existing memory before related work.
---

# Agent Memory

Maintain a persistent memory space for knowledge that should survive beyond the current conversation.

Use memory proactively when it would save future investigation, preserve project context, or make resumed work easier. Do not save everything; save information that is durable, actionable, and likely to matter again.

## Memory Root

Resolve the memory directory before reading or writing memories.

Use this priority order:
1. A path explicitly provided by the user
2. The `AGENT_MEMORY_DIR` environment variable
3. A project-local `.agent-memory/memories/` directory
4. A `memories/` directory next to this skill

Call the resolved directory `MEMORY_ROOT` in commands and notes. Never assume a tool-specific hidden directory or skill installation path.

Use project-local memory for project-specific facts, unfinished work, local paths, and repository decisions. Use global or skill-local memory only for knowledge that is reusable across projects.

## Encoding

Memory files must be read and written as UTF-8. This matters for memories that contain non-ASCII text, such as names, Chinese text, or other localized content.

When using PowerShell, always pass `-Encoding UTF8` to `Get-Content` when reading memory files. Do not rely on the shell's default encoding.

PowerShell example:
```powershell
Get-Content -LiteralPath $MemoryFile -Raw -Encoding UTF8
```

When checking whether a memory has encoding damage, inspect the raw bytes before assuming the stored content is corrupt. UTF-8 bytes that render incorrectly usually indicate the file was decoded with the wrong encoding.

## Privacy and Scope

Do not store secrets, passwords, API keys, access tokens, private credentials, or sensitive personal data.

Keep project-specific information inside that project's memory root unless the user explicitly asks to preserve it globally. When in doubt, prefer a narrower scope.

Write memories as durable facts, decisions, constraints, workflows, or resumable state. Avoid preserving casual conversation, transient guesses, or information that will quickly become stale.

## Proactive Usage

Save memories when you discover something worth preserving:
- Research findings that took effort to uncover
- Non-obvious patterns or gotchas in a codebase or workflow
- Solutions to tricky problems
- Architectural decisions and their rationale
- In-progress work that may be resumed later

Check memories when starting related work:
- Before investigating a familiar problem area
- When working on a feature or repository touched before
- When resuming work after a conversation break

Organize memories when needed:
- Consolidate scattered memories on the same topic
- Archive or delete outdated or superseded information
- Update the `status` field when work completes, gets blocked, or is abandoned

## Folder Structure

When possible, organize memories into category folders. There is no predefined structure; create categories that match the content.

Guidelines:
- Use kebab-case for folder and file names
- Keep paths under `MEMORY_ROOT`
- Consolidate or reorganize as the knowledge base evolves

Example:
```text
memories/
|-- file-processing/
|   `-- large-file-memory-issue.md
|-- dependencies/
|   `-- iconv-esm-problem.md
`-- project-context/
    `-- december-2025-work.md
```

This is only an example. Structure freely based on actual content.

## Frontmatter

All memories must include YAML frontmatter with a `summary` field. The summary should be concise enough to decide whether to read the full memory.

Required:
```yaml
---
summary: "1-2 line description of what this memory contains"
created: 2025-01-15
---
```

Optional:
```yaml
---
summary: "Worker thread memory leak during large file processing - cause and solution"
created: 2025-01-15
updated: 2025-01-20
status: in-progress
tags: [performance, worker, memory-leak]
related: [src/core/file/fileProcessor.ts]
---
```

Use `YYYY-MM-DD` dates. Use `status` values from this set when applicable: `in-progress`, `resolved`, `blocked`, `abandoned`.

## Search Workflow

Use a summary-first approach to find relevant memories efficiently.

Prefer the host environment's fastest available search tool. If `rg` is available:

```bash
# 1. List categories
ls "$MEMORY_ROOT"

# 2. View all summaries
rg "^summary:" "$MEMORY_ROOT"

# 3. Search summaries for a keyword
rg "^summary:.*keyword" "$MEMORY_ROOT" -i

# 4. Search tags for a keyword
rg "^tags:.*keyword" "$MEMORY_ROOT" -i

# 5. Full-text search when summary search is not enough
rg "keyword" "$MEMORY_ROOT" -i

# 6. Read specific memory files that look relevant, using explicit UTF-8 decoding
```

If the memory root is hidden, gitignored, or otherwise excluded by the search tool, add the tool-specific options needed to include it, such as `--hidden` or `--no-ignore` for ripgrep.

On PowerShell, use `$env:AGENT_MEMORY_DIR` for the environment variable and pass the resolved memory path directly when a command does not support shell-style variables.

On PowerShell, read selected files with `Get-Content -Raw -Encoding UTF8` so non-ASCII text is not misdecoded.

## Operations

### Save

1. Resolve `MEMORY_ROOT`
2. Determine the appropriate category for the content
3. Check whether an existing memory should be updated instead of creating a duplicate
4. Create the category folder if needed
5. Check that the target file does not already exist before writing
6. Write the file with required frontmatter and enough context to be useful later
7. Validate the saved memory

Use the current host's safe file-editing mechanism. Do not rely on a specific agent tool or shell redirection style.

Bash example:
```bash
MEMORY_ROOT="${AGENT_MEMORY_DIR:-.agent-memory/memories}"
mkdir -p "$MEMORY_ROOT/category-name"
test ! -e "$MEMORY_ROOT/category-name/filename.md"
```

PowerShell example:
```powershell
$MemoryRoot = if ($env:AGENT_MEMORY_DIR) { $env:AGENT_MEMORY_DIR } else { ".agent-memory\memories" }
New-Item -ItemType Directory -Force -Path (Join-Path $MemoryRoot "category-name")
Test-Path -LiteralPath (Join-Path $MemoryRoot "category-name\filename.md")
```

After choosing a file path, write the memory with the host's normal file editing tool.

### Update

When information changes:
- Edit the relevant memory instead of creating a near-duplicate
- Add or refresh the `updated` field
- Adjust `status`, `tags`, and `related` fields when they help future retrieval
- Preserve historical context when it explains why a decision changed

### Delete or Archive

Remove memories only when they are no longer relevant, are misleading, or contain information that should not be retained.

Before deleting:
- Confirm the target path is inside `MEMORY_ROOT`
- Prefer moving the file to an `archive/` folder when the information may still be useful
- Use the host environment's safe delete or move operation
- Remove empty category folders only after confirming they are inside `MEMORY_ROOT`

### Consolidate and Reorganize

Merge related memories when scattered notes make retrieval harder. Keep the strongest summary, preserve important context, and mark obsolete files as archived or superseded if deletion is not appropriate.

Move memories between categories when a better structure emerges. Update links or `related` paths if they become stale.

## Validation

After saving or updating a memory, verify:
- The file is under `MEMORY_ROOT`
- The filename and category path use kebab-case
- YAML frontmatter is present and valid
- `summary` exists and is decisive
- `created` exists and uses `YYYY-MM-DD`
- `updated` is present when an existing memory was materially changed
- `status` uses the allowed values when present
- The content is self-contained enough for a future agent to act on it
- The content does not contain secrets, credentials, or sensitive personal data

## Writing Guidelines

Write self-contained notes. Include enough context that a future reader does not need the original conversation to understand and act on the memory.

Keep summaries decisive. Reading the summary should tell the agent whether the details are worth opening.

Stay current. Update, archive, or delete outdated information.

Be practical. Save what is actually useful, not everything.

When writing detailed memories, consider including:
- Context: goal, background, constraints
- State: what is done, in progress, blocked, or abandoned
- Details: key files, commands, code snippets, decisions, and rationale
- Next steps: what to do next and open questions

Not every memory needs every section. Use only what is relevant.
