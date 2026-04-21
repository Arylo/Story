# MCP Obsidian Tools Reference

## Quick Reference: 6-Step Workflow

| Step | Task | Validate |
|------|------|----------|
| 1 | Verify root `.env` has `OBSIDIAN_API_KEY` | Check format & non-empty |
| 2 | Test Obsidian API at `127.0.0.1:27123` | Expect `"status": "OK"` |
| 3 | List staged files via `git status` | Identify files to sync |
| 4 | Record story folder structure | Note 时间线/, 故事/, 设定/ |
| 5 | Enumerate files with sync/skip markers | Count: to-sync vs already-synced |
| 6 | Invoke subagent for each file | Call mcp-obsidian append_content |

---

## Available Commands

The `mcp-obsidian` MCP server exposes these tools:

| Tool | Purpose | Key Parameters |
|------|---------|-----------------|
| `list_files_in_vault` | List root directory files | None |
| `list_files_in_dir` | List specific directory | `dir_path` |
| `get_file_contents` | Read file content | `file_path` |
| `search` | Search vault for text | `query` |
| `append_content` | Create/update file | `file_path`, `content`, `create_if_missing` |
| `patch_content` | Insert at location | `file_path`, `content`, `insert_location` |
| `delete_file` | Remove file/directory | `file_path` |

## Key Implementation Details

### Path Format
- All paths are **relative to vault root**
- Use forward slashes: `folder/subfolder/file.md`
- No leading/trailing slashes
- Supports nested directories

### append_content Behavior
```
file_path: "设定/角色设计/角色A.md"
content: "# 角色A\n\n..."
create_if_missing: true  ← Creates directories automatically
```
- Creates parent directories if they don't exist
- Idempotent: safe to re-run
- Overwrites existing content

### get_file_contents Verification
```
After each append_content call:
  1. Call get_file_contents with same filepath
  2. Compare returned content with source
  3. Report ✅ (match) or ❌ (mismatch)
```

### Environment Setup
```env
OBSIDIAN_API_KEY=<your_api_key_from_local_rest_api>
```
- Required in root `.env` file
- Referenced in `.vscode/mcp.json` via `envFile`
- Obtain from Obsidian Local REST API plugin
- Vault name in story folder's `.env` via `OBSIDIAN_VAULT`

---

## Real-World Workflow: Sync `all-are-heroes`

**Prerequisites Verified:**
```
✅ Root .env: OBSIDIAN_API_KEY=xxxxxxxx
✅ Obsidian: http://127.0.0.1:27123 (Local REST API v3.6.1)
✅ Story .env: OBSIDIAN_VAULT=all-are-heroes
```

**Step 3 Output:**
```
Staged Files (2):
  - all-are-heroes/CLAUDE.md
  - all-are-heroes/故事/正文/01-01.md
```

**Step 4 Output:**
```
Story Folder: all-are-heroes/
Subdirectories:
  - 时间线/ (timeline)
  - 故事/ (story content)
  - 设定/ (world-building & characters)
```

**Step 5 Output:**
```
To Sync (2 files):
  ✓ CLAUDE.md → CLAUDE.md
  ✓ 故事/正文/01-01.md → 故事/正文/01-01.md

Already Synced (43 files) - Skip:
  ✓ 设定/世界观/ (34 files)
  ✓ 设定/角色设计/ (9 files)
```

**Step 6 Subagent Execution:**

For each file:
```
File 1: CLAUDE.md
  1. Read: all-are-heroes/.env → OBSIDIAN_VAULT=all-are-heroes
  2. Call: mcp_obsidian_append_content
     - filepath: "CLAUDE.md"
     - content: "# CLAUDE.md\n\n## 写作约定\n..."
     - create_if_missing: true
  3. Verify: mcp_obsidian_get_file_contents("CLAUDE.md")
  4. Report: ✅ 66 bytes synced

File 2: 故事/正文/01-01.md
  1. Read: all-are-heroes/.env → OBSIDIAN_VAULT=all-are-heroes
  2. Call: mcp_obsidian_append_content
     - filepath: "故事/正文/01-01.md"
     - content: "" (empty)
     - create_if_missing: true
  3. Verify: mcp_obsidian_get_file_contents("故事/正文/01-01.md")
  4. Report: ✅ File created (empty, ready for content)
```

**Final Report:**
```
Summary: 2/2 files synced successfully
Vault: all-are-obsidian
Timestamp: 2026-04-21 14:22:15
Status: Ready to commit
```

---

## Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| "Vault not found" | Wrong `OBSIDIAN_VAULT` value | Check story folder `.env` |
| "API key rejected" | Invalid or expired key | Regenerate in Obsidian Local REST API settings |
| "Connection timeout" | Obsidian not running | Ensure Obsidian is open and plugin enabled |
| "Permission denied" | API key lacks permissions | Re-authenticate in Obsidian |
| Content mismatch on verify | Network/timing issue | Re-run sync (tool is idempotent) |
