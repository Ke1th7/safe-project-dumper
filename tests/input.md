# Setup Claude Code Statusline

Set up a rich multi-line Claude Code statusline matching this style:

```
[Opus 4.6] 📁 my-project | 📍 main | 📜 45 files +21 -392
█████████░░░░░░ 37% (74k/200k) | ↑22k ↓106k | $12.03 | ⏱ 28m 39s
```

## Requirements

1. Use **Python** (not `jq`) for JSON parsing — Python is available in most environments, `jq` may not be.
2. Create **two files** under the user's `~/.claude/` directory:
   - `~/.claude/statusline-command.sh` — thin bash wrapper that calls the Python script via `exec python3 ~/.claude/statusline.py` (use `$HOME` expansion, no hardcoded usernames)
   - `~/.claude/statusline.py` — the actual statusline logic
3. Add a `statusLine` entry to `~/.claude/settings.json` (preserve all existing settings, only add/update the `statusLine` key):

```
"statusLine": {
  "type": "command",
  "command": "bash $HOME/.claude/statusline-command.sh"
}
```

**Important**: The `command` value must use the user's actual expanded `$HOME` path (e.g. `bash /home/alice/.claude/statusline-command.sh`), not a literal `$HOME` string. Detect the home directory at setup time.

## Claude Code Statusline JSON Schema

The script receives this JSON on stdin (from Claude Code official docs):

```
{
  "cwd": "/current/working/directory",
  "session_id": "abc123...",
  "model": { "id": "claude-opus-4-6", "display_name": "Opus" },
  "workspace": { "current_dir": "/current/working/directory", "project_dir": "/original/project/directory" },
  "version": "1.0.80",
  "cost": {
    "total_cost_usd": 12.03,
    "total_duration_ms": 106939000,
    "total_api_duration_ms": 2300,
    "total_lines_added": 21,
    "total_lines_removed": 392
  },
  "context_window": {
    "total_input_tokens": 22000,
    "total_output_tokens": 106000,
    "context_window_size": 200000,
    "used_percentage": 37,
    "remaining_percentage": 63,
    "current_usage": {
      "input_tokens": 74000,
      "output_tokens": 1200,
      "cache_creation_input_tokens": 5000,
      "cache_read_input_tokens": 2000
    }
  },
  "exceeds_200k_tokens": false
}
```

**Nullable fields**: `context_window.current_usage` can be `null` before the first API call. `used_percentage` / `remaining_percentage` may be `null` early in a session. Handle these gracefully with fallback to 0.

## Field Mapping

| Display element         | JSON source                                                  |
| ----------------------- | ------------------------------------------------------------ |
| `[Opus 4.6]`            | `model.display_name` + version parsed from `model.id` (e.g. `claude-opus-4-6` → `4.6`). If `display_name` already contains digits, use as-is to avoid duplication like "Opus 4.6 4.6" |
| ` 📁  project`           | `os.path.basename(workspace.current_dir)`                    |
| ` 📍  main`              | `git rev-parse --abbrev-ref HEAD` run in `workspace.current_dir` (not in JSON, needs subprocess with timeout) |
| ` 📜  45 files +21 -392` | file count from `git diff --name-only` (subprocess), lines from `cost.total_lines_added` / `cost.total_lines_removed`. Only shown if there are changes |
| Progress bar + `37%`    | `context_window.used_percentage` rendered as 15-char bar (█ filled, ░ empty) |
| `(74k/200k)`            | used_tokens = `round(used_percentage) × context_window_size / 100`, then `/context_window_size` |
| `↑22k ↓106k`            | `context_window.total_input_tokens` / `total_output_tokens`, formatted as Xk or X.XM |
| `$12.03`                | `cost.total_cost_usd`                                        |
| ` ⏱  28m 39s`           | `cost.total_duration_ms` converted to minutes + seconds      |

## Output Format (2 lines)

```
Line 1: [Model Version] 📁 dirname | 📍 branch | 📜 XX files +XX -XX
Line 2: █████░░░░░░░░░░ XX% (Xk/Xk) | ↑Xk ↓Xk | $X.XX | ⏱ Xm Xs
```

The ` 📜  XX files +XX -XX` segment on line 1 is only shown when there are file changes.

## Verification

After creating both files and updating settings.json, test with mock JSON:

```
echo '{"model":{"id":"claude-opus-4-6","display_name":"Opus"},"workspace":{"current_dir":"'$(pwd)'"},"cost":{"total_cost_usd":12.03,"total_duration_ms":1719000,"total_lines_added":21,"total_lines_removed":392},"context_window":{"total_input_tokens":22000,"total_output_tokens":106000,"context_window_size":200000,"used_percentage":37,"current_usage":{"input_tokens":74000}}}' | python3 ~/.claude/statusline.py
```

Expected output (project name and git files will vary):

```
[Opus 4.6] 📁 <project-dir> | 📍 <branch> | 📜 <N> files +21 -392
█████░░░░░░░░░░ 37% (74k/200k) | ↑22k ↓106k | $12.03 | ⏱ 28m 39s
```