---
name: sync-from-obsidian
description: 'Sync files FROM Obsidian vault back to the story workspace (时间线/故事/设定). Use for: pulling Obsidian edits back to repo, restoring vault content, selective file/folder pull by path. Supports targeting by folder-name, sub-folder-path, or file-path arguments. At least one argument must be provided.'
argument-hint: '[folder-name=<story-folder>] [sub-folder-path=<relative-path>] [file-path=<vault-relative-file>]'
---

# Sync from Obsidian

## When to Use

- **Pull Obsidian edits**: Retrieve content edited in Obsidian and write back to the workspace
- **Selective restore**: Pull a specific file or sub-folder from the vault
- **Reverse sync**: Complement `sync-to-obsidian` by syncing in the opposite direction
- **Conflict resolution**: Overwrite workspace files with the current vault version

## Arguments

At least **one** of the following three arguments must be provided:

| Argument | Required | Description | Example |
|----------|----------|-------------|---------|
| `folder-name` | optional | Story folder in the workspace | `all-are-heroes` |
| `sub-folder-path` | optional | Vault-relative sub-folder to pull (pulls all files within) | `设定/世界观` |
| `file-path` | optional | Vault-relative path of a single file to pull | `故事/正文/01-01.md` |

⚠️ **Validation**: If none of the three arguments is provided → abort with error message.

### Argument Resolution Rules

**Determining the pull scope** (evaluated in order):

1. If `file-path` is given → **single-file mode**: pull only that one file
2. Else if `sub-folder-path` is given → **sub-folder mode**: pull all files within that sub-folder
3. Else (only `folder-name`) → **full-story mode**: pull all files in `时间线/`, `故事/`, `设定/`

**Determining the target workspace story folder**:

1. If `folder-name` is given → use it directly (e.g., `all-are-heroes/`)
2. Else if `file-path` starts with a known story folder name → infer from path prefix (e.g., `all-are-heroes/故事/...` → `all-are-heroes`)
3. Else if `sub-folder-path` starts with a known story folder name → infer from path prefix
4. Else → pull relative to vault root; write to workspace root story folder (prompt user to confirm if ambiguous)

**Example invocations:**
```
# Minimum: only file-path
file-path=故事/正文/01-01.md

# Minimum: only sub-folder-path
sub-folder-path=设定/角色设计

# folder-name + sub-folder-path (scoped pull)
folder-name=all-are-heroes sub-folder-path=设定/角色设计

# All three (most explicit)
folder-name=all-are-heroes sub-folder-path=设定 file-path=设定/世界观/背景交代.md

# Pull everything for a story
folder-name=all-are-heroes
```

## Prerequisites

✅ **MCP Configuration**
- `.vscode/mcp.json` configured with `mcp-obsidian` server
- `.env` contains valid `OBSIDIAN_API_KEY`

## Scope: Story Folder Coverage

⚠️ **This skill only writes into the three main story folder subdirectories:**

| Folder | Purpose | Included |
|--------|---------|----------|
| `时间线/` | Timeline files (year-based events) | ✅ YES |
| `故事/` | Story content (正文 chapters + 大纲 outlines) | ✅ YES |
| `设定/` | World-building & characters (世界观 + 角色设计) | ✅ YES |

- ✅ **Writes to**: workspace files inside the three main folders
- ❌ **Does NOT overwrite**: Top-level `.env`, `CLAUDE.md`, `AGENT.md`
- ❌ **Does NOT pull**: Files outside the three main folders
- ❌ **Does NOT pull**: Version control or config artifacts

## Quick Checklist

### Step 1: Parse & Validate Arguments
- [ ] Read all three arguments: `folder-name`, `sub-folder-path`, `file-path`
- [ ] **Abort** if all three are missing → print error: `Error: at least one of folder-name, sub-folder-path, or file-path must be provided`
- [ ] Determine **pull mode**: `single-file`, `sub-folder`, or `full-story` (see rules above)
- [ ] Determine **target workspace folder**: use `folder-name` if given, else infer from path prefix, else prompt

### Step 2: Verify ROOT `.env` Configuration
- [ ] Check if `OBSIDIAN_API_KEY` exists in root `.env`
- [ ] Confirm API key is non-empty and valid format
- [ ] Example: `OBSIDIAN_API_KEY=xxxxxxxx`

### Step 3: Verify Obsidian Runtime
- [ ] Access `http://127.0.0.1:27123` with Authorization header
- [ ] Confirm response includes `"status": "OK"` and `"authenticated": true`
- [ ] Check Obsidian Service status (Local REST API plugin)
- [ ] Example curl: `curl http://127.0.0.1:27123 -H "Authorization: Bearer <KEY>"`

### Step 4: Read Story Folder `.env` for Vault Name
- [ ] Read `<target-story-folder>/.env`
- [ ] Extract `OBSIDIAN_VAULT` value
- [ ] Example: `OBSIDIAN_VAULT=all-are-heroes`

### Step 5: Enumerate Files to Pull from Vault
- [ ] **single-file mode**: target = `<file-path>`
- [ ] **sub-folder mode**: use `mcp_obsidian_list_files_in_dir` on `<sub-folder-path>`, collect all `.md` files recursively
- [ ] **full-story mode**: use `mcp_obsidian_list_files_in_dir` on each of `时间线/`, `故事/`, `设定/` in turn
- [ ] Validate each path is within the three main folders (reject others)
- [ ] List as "pull" or "skip" based on scope
- [ ] Example output:
  ```
  Mode: sub-folder  (sub-folder-path=设定/角色设计)
  Vault: all-are-heroes

  To Pull (3 files):
  ✓ 设定/角色设计/角色A.md → all-are-heroes/设定/角色设计/角色A.md
  ✓ 设定/角色设计/角色B.md → all-are-heroes/设定/角色设计/角色B.md
  ✓ 设定/角色设计/主要角色.md → all-are-heroes/设定/角色设计/主要角色.md

  Out of Scope (will not pull):
  ⊘ CLAUDE.md (top-level)
  ⊘ .env (top-level)
  ```

### Step 6: Invoke Subagent for Pull
**For Each File:**
1. Read vault file via `mcp_obsidian_get_file_contents`
2. Write content to workspace file path:
   - If file exists → overwrite (use `replace_string_in_file` or full replace)
   - If file does not exist → create with `create_file`
3. Verify workspace file content matches vault content
4. Report status: ✅ (success) or ❌ (error)

## Common Patterns

### Pull a Single File — No folder-name
```
Arguments:
  file-path=故事/正文/01-01.md

Execution:
1. Infer story folder from path prefix → not available, use vault root
2. Check root .env for OBSIDIAN_API_KEY ✓
3. Verify Obsidian at 127.0.0.1:27123 ✓
4. mcp_obsidian_get_file_contents("故事/正文/01-01.md")
5. Write to workspace: <story-folder>/故事/正文/01-01.md
6. Verify ✓
```

### Pull an Entire Sub-folder — With folder-name
```
Arguments:
  folder-name=all-are-heroes
  sub-folder-path=设定/角色设计

Execution:
1. Target folder: all-are-heroes/
2. Check root .env for OBSIDIAN_API_KEY ✓
3. Verify Obsidian at 127.0.0.1:27123 ✓
4. Read all-are-heroes/.env → OBSIDIAN_VAULT=all-are-heroes
5. mcp_obsidian_list_files_in_dir("设定/角色设计") → [角色A.md, 角色B.md, ...]
6. For each file: read from vault, write to workspace
7. Summary report
```

### Full Story Pull — Only folder-name
```
Arguments:
  folder-name=lone-eagle-valor

Execution:
1. Check root .env for OBSIDIAN_API_KEY ✓
2. Verify Obsidian at 127.0.0.1:27123 ✓
3. Read lone-eagle-valor/.env → OBSIDIAN_VAULT=lone-eagle-valor
4. List files in: 时间线/, 故事/, 设定/
5. Pull all files, write to workspace
6. Summary report
```

## Subagent Workflow

Subagent executes the following for **each file** to be pulled:

### Subagent Step 1: Read Vault File
```
Tool: mcp_obsidian_get_file_contents
Parameters:
  - filepath: "<vault-relative-path>"
    (e.g., "设定/角色设计/角色A.md")

Expected: Returns file content string
```

### Subagent Step 2: Write to Workspace
```
Case A – File already exists in workspace:
  Tool: replace_string_in_file
  (Replace entire content with vault content)

Case B – File does not exist in workspace:
  Tool: create_file
  Parameters:
    - filePath: "<absolute-workspace-path>"
      (e.g., "/Users/.../all-are-heroes/设定/角色设计/角色A.md")
    - content: "<vault file content>"
```

### Subagent Step 3: Verify Pull Result
```
Read workspace file and compare with vault content.

Expected: File content matches vault source
If mismatch: Report error with details
```

### Subagent Output Example
```
✅ File 1/3: 设定/角色设计/角色A.md
   ← Pulled 2,450 bytes from vault
   → Written to all-are-heroes/设定/角色设计/角色A.md
   → Verified: content matches

✅ File 2/3: 设定/角色设计/角色B.md
   ← Pulled 2,180 bytes from vault
   → Written to all-are-heroes/设定/角色设计/角色B.md
   → Verified: content matches

✅ File 3/3: 设定/角色设计/主要角色.md
   ← Pulled 5,320 bytes from vault
   → Written to all-are-heroes/设定/角色设计/主要角色.md
   → Verified: content matches

Summary: 3/3 files pulled successfully
Vault: all-are-heroes
Timestamp: 2026-04-21 12:34:56
```
