---
name: quark-auto-save
description: Manage quark-auto-save(QAS, 夸克自动转存, 夸克转存, 夸克订阅) tasks via CLI.
metadata:
  openclaw:
    emoji: "💾"
    homepage: "https://github.com/Cp0204/quark-auto-save"
    requires:
      env:
        - QAS_BASE_URL
        - QAS_TOKEN
      anyBins:
        - curl
        - python3
    primaryEnv: QAS_TOKEN
---

# quark-auto-save

Manage quark-auto-save(QAS, 夸克自动转存, 夸克转存, 夸克订阅) tasks via CLI.

When user send message like `https://pan.quark.cn/s/***`, get detail, add a QAS task.

**WIKI:**
- RegexRename: https://github.com/Cp0204/quark-auto-save/wiki/%E6%AD%A3%E5%88%99%E5%A4%84%E7%90%86%E6%95%99%E7%A8%8B
- MagicRegex: https://github.com/Cp0204/quark-auto-save/wiki/%E9%AD%94%E6%B3%95%E5%8C%B9%E9%85%8D%E5%92%8C%E9%AD%94%E6%B3%95%E5%8F%98%E9%87%8F

## ⚠️ Prerequisites

**Env:**
- `QAS_BASE_URL` -  User provided, e.g., http://192.168.1.x:5005
- `QAS_TOKEN` - User provided

**Actual configuration values are recorded in TOOLS.md, Do not modify SKILL.md**

## First Configuration: Analyze User Habits

After the user sets the token, the following analysis must be performed and recorded in TOOLS.md:

1. **Get Current Configuration**:
   ```bash
   python3 {baseDir}/scripts/qas_client.py data
   ```

2. **Analyze Saving Habits**:
   - Extract `savepath` directory patterns from existing tasks (e.g., `/video/tv/`, `/video/anime/`, `/video/movie/`)
   - Observe file naming format from `pattern` and `replace` fields — e.g., `{TASKNAME}.S01E01.mp4` vs `01.mp4`
   - Note which `magic_regex` key the user prefers (e.g. `$TV_MAGIC`)

3. **Record to TOOLS.md**:
   ```markdown
   ### quark-auto-save habits
   - TV Series Directory: /video/tv/{name}
   - Anime Directory: /video/anime/{name}
   - Movie Directory: /video/movie/{name}
   - Naming Pattern: $TV_MAGIC (e.g., 都是她的错.S01E01.mp4)
   ```

## Python Client

Use `{baseDir}/scripts/qas_client.py` for all operations:

```bash
python3 {baseDir}/scripts/qas_client.py data                            # Get all config & tasks
python3 {baseDir}/scripts/qas_client.py search "query" [-d]            # Search resources
python3 {baseDir}/scripts/qas_client.py detail "<shareurl>"            # Get share detail
python3 {baseDir}/scripts/qas_client.py savepath "/path"               # Check savepath
python3 {baseDir}/scripts/qas_client.py add-task '{"taskname": "Name", ...}' # Add task
python3 {baseDir}/scripts/qas_client.py run-task [taskname|json]            # Run task(s)
python3 {baseDir}/scripts/qas_client.py update-task "Name" '{"savepath": "/new"}' # Update task
python3 {baseDir}/scripts/qas_client.py delete-task "TaskName"         # Delete task by name
python3 {baseDir}/scripts/qas_client.py update '{"key": "value"}'      # Update config
```



## Task Schema

```json
{
  "taskname": "MediaName",
  "shareurl": "https://pan.quark.cn/s/xxx#/list/share/fid",
  "savepath": "/video/tv/MediaName",
  "pattern": "$TV_MAGIC",
  "replace": "",
  "update_subdir": "",
  "ignore_extension": false,
  "runweek": [1,2,3,4,5,6,7]
}
```

**Required Fields:** `taskname`, `shareurl`, `savepath`

**Optional Fields:** `pattern`, `replace`, `update_subdir`, `ignore_extension`, `runweek`, `addition`

> `add-task` auto-detects zip/rar/7z files and enables `auto_unarchive` plugin automatically. No manual `addition` config needed for this.

## Configuration Rules

### `shareurl` Format
- `https://pan.quark.cn/s/{abc123}`
- `https://pan.quark.cn/s/{abc123}#/list/share/{fid}`

### Subdirectory Priority
1. Video files (mp4, mkv, avi)
2. Resolution: 4K > 1080P > 720P

> Archive files (zip, rar, 7z) are also supported — auto-unarchive is enabled automatically by `add-task` `run-task`.

**Get subdir info:**
```bash
python3 {baseDir}/scripts/qas_client.py detail "<shareurl>"
```

**Add task example:**
```bash
python3 {baseDir}/scripts/qas_client.py add-task '{"taskname": "Black Mirror", "shareurl": "https://pan.quark.cn/s/xxx", "savepath": "/video/tv/Black Mirror", "pattern": "$TV_MAGIC"}'
```

## `pattern` & `replace`

| Pattern | Replace | Result |
|---|---|---|
| `.*` | | Save all files |
| `\.(mp4\|mkv)$` | | Save video files only |
| `^(\d+)\.mp4` | `S02E\1.mp4` | 01.mp4 → S02E01.mp4 |
| `$TV_MAGIC` | | Use custom magic regex |

### `replace` Magic Variables

| Variable | Description |
|---|---|
| `{TASKNAME}` | Task name |
| `{II}` | Index number (01, 02...) |
| `{EXT}` | File extension |
| `{SXX}` | Season (S01, S02...) |
| `{E}` | Episode number |
| `{DATE}` | Date (YYYYMMDD) |

## Workflows

### Add New Task
1. **Search**: `python3 {baseDir}/scripts/qas_client.py search "MediaName" -d`
2. **Verify & Get Details**: `python3 {baseDir}/scripts/qas_client.py detail "<shareurl>"`
   - Check if `shareurl` is valid (not banned)
   - Check file list for video files and select the subdir
3. **Analyze `pattern` & `replace`**: Compare the share's filenames with the user's preferred format (from TOOLS.md).
   - If filenames already match the preferred format → use `".*"` (save as-is)
   - If filenames need renaming → write regex `pattern` to capture episode/season info, and `replace` with magic variables to produce the preferred format
   - Example: source `01.mp4`, preferred `{TASKNAME}.S01E01.mp4` → `"pattern": "^(\\d+)\\.mp4$", "replace": "{TASKNAME}.S01E\\1.{EXT}"`
4. **Add task**: `python3 {baseDir}/scripts/qas_client.py add-task '{"taskname": "MediaName", "shareurl": "https://pan.quark.cn/s/xxx#/list/share/fid", "savepath": "<user_habit_path>", "pattern": "<pattern>", "replace": "<replace>"}'`
   - `savepath` and `pattern` must follow the user's existing habits recorded in TOOLS.md

**Tip:** If the search result `taskname` contains keywords like `全X集`, `完结`, `全集`, it's a completed series — run directly with `run-task <json>` (one-time transfer) instead of `add-task` (subscription). Still do step 3 to determine the correct `pattern` & `replace`.

### Check Invalid Tasks
1. **Get tasks**: `python3 {baseDir}/scripts/qas_client.py data`
2. **Check for `shareurl_ban` key in tasklist**
3. **Replace invalid URLs and remove `shareurl_ban` key**

### Delete Task
```bash
python3 {baseDir}/scripts/qas_client.py delete-task "TaskName"
```

### Update Task
```bash
# Partial update (only specified fields are changed)
python3 {baseDir}/scripts/qas_client.py update-task "TaskName" '{"savepath": "/new/path"}'
python3 {baseDir}/scripts/qas_client.py update-task "TaskName" '{"pattern": "$TV_MAGIC", "runweek": [1,3,5]}'
```

### Update Config
```bash
# Update global config (allowed keys: cookie, crontab, push_config, tasklist, magic_regex, plugins, source)
python3 {baseDir}/scripts/qas_client.py update '{"crontab": "0 9 * * *"}'
```

### Run Tasks
```bash
python3 {baseDir}/scripts/qas_client.py run-task                    # All tasks
python3 {baseDir}/scripts/qas_client.py run-task "TaskName"       # Specific task
python3 {baseDir}/scripts/qas_client.py run-task '{"taskname": "Test", ...}'  # Direct task
```

## Output Format

All commands output text. First word indicates status:

- **OK** — success, no data
- **OK {json}** — success with data (JSON on same line)
- **ERROR: message** — failure
- **run-task**: `OK` on first line, followed by log lines

