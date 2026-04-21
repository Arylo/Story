---
name: sync-to-obsidian
description: 'Sync story folder files (时间线/故事/设定/参考资料) to Obsidian vault while preserving directory structure. Use for: story content migration, timeline/worldbuilding backup, character design sync, story text updates via mcp-obsidian. Note: Only syncs the four main story folders, not top-level configs.'
argument-hint: 'Specify story folder name and which top-level story sections to sync'
---

# Sync to Obsidian

## When to Use

- **Content migration**: Move project files/folders into Obsidian vaults
- **Selective backup**: Sync specific directories while skipping others
- **Multi-environment workflow**: Maintain synchronized copies across workspace and Obsidian
- **Batch updates**: Keep Obsidian vault in sync with evolving project structure

## Prerequisites

✅ **MCP Configuration**
- `.vscode/mcp.json` configured with `mcp-obsidian` server
- `.env` contains valid `OBSIDIAN_API_KEY`

## Scope: Story Folder Coverage

⚠️ **This skill syncs ONLY the four main story folder subdirectories:**

| Folder | Purpose | Included |
|--------|---------|----------|
| `时间线/` | Timeline files (year-based events) | ✅ YES |
| `故事/` | Story content (正文 chapters + 大纲 outlines) | ✅ YES |
| `设定/` | World-building & character files (世界观 + 角色 / 人物设定) | ✅ YES |
| `参考资料/` | Reference materials (史料、考据、灵感摘录等) | ✅ YES |

- ✅ **Syncs within these four folders**: All markdown files and subdirectories
- ✅ **Marks as "skip"**: Already-synced subsections (e.g., 世界观 if previously synced)
- ❌ **Does NOT sync**: Top-level `.env`, `CLAUDE.md`, `AGENT.md` (except on first run)
- ❌ **Does NOT sync**: Other story folders outside the story structure
- ❌ **Does NOT sync**: Version control or build artifacts (`.git`, `node_modules`, etc.)

**Example Story Folder Structure:**
```
all-are-heroes/
├── .env                    ← Contains OBSIDIAN_VAULT
├── CLAUDE.md               ← Synced only on init
├── AGENT.md                ← Synced only on init
├── 时间线/                 ← ✅ Scope: All files synced
│   ├── 2024.md
│   ├── 2025.md
│   └── 2026.md
├── 故事/                   ← ✅ Scope: All files synced
│   ├── 正文/
│   │   ├── 01-01.md
│   │   ├── 01-02.md
│   │   └── ...
│   └── 大纲/
│       ├── 第一部.md
│       └── ...
├── 设定/                   ← ✅ Scope: All files synced
│   ├── 世界观/             (34 files)
│   └── 角色设计/           (9 files)
└── 参考资料/               ← ✅ Scope: All files synced
    ├── 史料摘录.md
    └── 灵感备忘.md
```

## Quick Checklist

### Step 1: Verify ROOT `.env` Configuration
- [ ] Check if `OBSIDIAN_API_KEY` exists in root `.env`
- [ ] Confirm API key is non-empty and valid format
- [ ] Example: `OBSIDIAN_API_KEY=xxxxxxxx`

### Step 2: Verify Obsidian Runtime
- [ ] Access `http://127.0.0.1:27123` with Authorization header
- [ ] Confirm response includes `"status": "OK"` and `"authenticated": true`
- [ ] Check Obsidian Service status (Local REST API plugin)
- [ ] Example curl: `curl http://127.0.0.1:27123 -H "Authorization: Bearer <KEY>"`

### Step 3: Check Staged Changes
- [ ] Run `git status` to view staged files
- [ ] Identify which files are new, modified, or ready for sync
- [ ] Note: Only sync files that are staged or explicitly marked for sync

### Step 4: Identify Four Main Folders (Story Structure Only)
- [ ] Confirm syncing is within a story folder (e.g., `all-are-heroes/`)
- [ ] **Only sync these four main folders:**
  * `时间线/` – Timeline files (events by year)
  * `故事/` – Story content (chapters & outlines)
  * `设定/` – World-building & characters (worldview & character files)
  * `参考资料/` – Research, reference, and source material
- [ ] Identify staged files **within** these four folders only
- [ ] Note which subdirectories to skip (e.g., already synced `世界观/`)
- [ ] ❌ **Exclude**: Top-level `.env`, `CLAUDE.md`, `AGENT.md` (unless init)

### Step 5: List Files to Sync (Within the Four Folders)
- [ ] Enumerate only staged files from Step 3 that are **in the four main folders**
- [ ] Group by target vault path
- [ ] Mark as "sync" or "skip" based on folder status
- [ ] Example output:
  ```
  Scope: Story folder (all-are-heroes)
  
  To Sync (3 files - within scope):
  ✓ 故事/正文/01-01.md → 故事/正文/01-01.md
  ✓ 故事/大纲/第一部.md → 故事/大纲/第一部.md
  ✓ 参考资料/史料摘录.md → 参考资料/史料摘录.md
  
  Already Synced (skip - within scope):
  ✓ 设定/世界观/ (34 files)
  ✓ 设定/角色设计/ (9 files)
  
  Out of Scope (will not sync):
  ⊘ CLAUDE.md (top-level)
  ⊘ AGENT.md (top-level)
  ```

### Step 6: Invoke Subagent for Sync
**For Each File:**
1. Read story folder's `.env` to get `OBSIDIAN_VAULT` value
2. Use `mcp_obsidian_append_content` to sync content:
   - `filepath`: vault-relative path (e.g., `设定/角色设计/角色A.md`)
   - `content`: full file content
   - `create_if_missing`: true
3. Verify with `mcp_obsidian_get_file_contents` after each file
4. Report status: ✅ (success) or ❌ (error)

## Common Patterns

### Story Folder Sync Workflow (Real Example)
```
Story Folder: all-are-heroes/
├── .env (contains OBSIDIAN_VAULT=all-are-heroes)
├── 时间线/
├── 故事/
│   ├── 正文/
│   └── 大纲/
├── 设定/
│   ├── 世界观/ ← Already synced, skip
│   └── 角色设计/ ← Already synced, skip
└── 参考资料/

Execution:
1. Check root .env for OBSIDIAN_API_KEY ✓
2. Verify Obsidian at 127.0.0.1:27123 ✓
3. Find staged files in all-are-heroes/
4. Record subdirectories: 时间线/, 故事/, 设定/, 参考资料/
5. List files to sync (skip already-synced)
6. Invoke subagent for each file
```

### Phased Sync (Skip Completed Sections)
```
Sync: 故事/, 参考资料/
Skip: 设定/世界观/  ← Already synced
```

### Selective Sync (Single Folder)
```
Vault: my-vault
Source: all-are-heroes/参考资料/
Target: 参考资料/  ← Preserves structure
```

### Update Existing Vault
```
1. Identify changed files since last sync
2. Read only updated file content
3. Re-sync using same paths
4. mcp-obsidian overwrites previous content
```

## Subagent Workflow

Subagent executes the following for **each file** to be synced:

### Subagent Step 1: Read Story Folder `.env`
```
Read: <story-folder>/.env
Extract: OBSIDIAN_VAULT value
Example: OBSIDIAN_VAULT=all-are-heroes
```

### Subagent Step 2: Sync File Content
```
Tool: mcp_obsidian_append_content
Parameters:
  - filepath: "<vault-relative-path>"
    (e.g., "设定/角色设计/角色A.md")
  - content: "<full-file-content>"
  - create_if_missing: true  ← creates dirs automatically

Example:
  filepath: "故事/正文/01-01.md"
  content: "# 章节标题\n\n内容..."
```

### Subagent Step 3: Verify Sync Result
```
Tool: mcp_obsidian_get_file_contents
Parameters:
  - filepath: "<vault-relative-path>"

Expected: File content matches source
If mismatch: Report error with details
```

### Subagent Output Example
```
✅ File 1/10: 设定/角色设计/角色A.md
   → Synced 2,450 bytes
   → Verified: content matches

✅ File 2/10: 设定/角色设计/角色B.md
   → Synced 2,180 bytes
   → Verified: content matches

...

Summary: 10/10 files synced successfully
Vault: all-are-obsidian
Timestamp: 2026-04-21 12:34:56
```
