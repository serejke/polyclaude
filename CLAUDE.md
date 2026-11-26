# polyclaude

polyclaude is the entry point to Claude Code. It shows the running sessions, starts a new session in a directory, and resumes an old session. It is one Python file that uses curses and has no dependencies.

## Project structure

```
src/polyclaude/
  __init__.py          # package marker
  main.py              # all logic: config, parsing, TUI, entry point
  _version.py          # written by hatch-vcs at build time (gitignored)
pyproject.toml         # hatchling + hatch-vcs build; entry point polyclaude -> main:main
install.sh             # interactive installer: asks for the binary name, runs uv tool install
```

## Tabs

`Tab` moves to the next tab. The `TAB_*` constants give the tab indices.

- **Start** (tab 0). This tab has two sections:
  - *Running* shows the live sessions from `claude agents --json`. Enter attaches to a background session. A `terminal` session is a foreground `claude` in a different terminal, and you cannot attach to it. For such a session, Enter shows its tty and tells you to type `/bg` in it.
  - *New session in* shows the directories from the session history. The current directory is first, and the cursor starts on it. Enter starts a new session in the directory.
- **Resume** (tab 1). This tab shows the recent sessions, newest activity first. Each row shows the time (relative up to 7 days, then the date), the name or path, the stats, the first and last user messages, and the last agent reply. `/` searches the session text. Enter expands a session into the full chat, scrolled to the end. Enter again resumes it. A running session has a `● status (kind)` marker. Enter attaches to a running background session. Enter does not resume a running terminal session, because that makes a copy.
- **Settings** (tab 2). Space or Enter toggles a setting. polyclaude saves the change immediately.

**Mouse.** A click on a tab label opens the tab. On Start, a click selects an item, and a double-click (or a click on the selected item) activates it. On Resume, a click expands a session. The wheel moves the cursor in a list and scrolls the chat. The draw functions return `{screen row: item index}` for the click hit test. polyclaude captures the mouse, so the terminal needs Shift (or Option) to select text.

## Launch

- polyclaude does `os.chdir` to the session directory and then `os.execvp`. The new process replaces polyclaude.
- **Background launch** (the default). polyclaude runs `claude --bg [--resume <id>]` and reads the id from the `claude attach <id>` line on stdout. Then it execs `claude attach <id>`. The session runs in the Claude background service, not in this terminal. Thus the session continues when the terminal or the ssh connection closes.
- **Workspace trust.** `claude --bg` refuses an untrusted directory ("Workspace not trusted"), because it cannot show the dialog. Claude Code reads `projects[<dir>].hasTrustDialogAccepted` in `~/.claude.json` (`$CLAUDE_CONFIG_DIR/.claude.json`): from the cwd up to the git root inside a repository, or up to `/` outside one. So a trusted `~/Desktop` does not cover a repository in it. `untrusted_note` mirrors this check (read-only) before a background launch; when the record is missing, `_exec_claude` falls back to the foreground so claude shows its own dialog. Never write `~/.claude.json`: it is Claude Code's internal state, not an interface. The home directory is skipped: Claude Code never records its trust.
- polyclaude captures only stdout of `claude --bg`. stdin and stderr stay on the terminal, because `claude --bg` can show progress or ask a question there.
- `--fg` starts claude in the foreground for one run. `--unsafe` adds `--dangerously-skip-permissions` for one run.

## Loading

Session files can be larger than 100 MB. polyclaude must show the screen without a full read of these files. It loads data in four steps:

1. **cwd cache.** The Start tab needs the cwd of each session file. polyclaude reads the first 5 lines of a file one time and keeps `{path: cwd}` in `~/.cache/polyclaude/cwds.json`. The cwd of a session does not change, so the cache needs no invalidation. polyclaude does not cache a file that has no cwd yet.
2. **Head and tail.** `scan_session_meta` reads the start of a file (sessionId, cwd, first user message) and the last 256 KB (last user message, last activity time, `custom-title`). Claude Code writes a `last-prompt` line again and again at the end of the file. polyclaude uses it when tool output pushes the last user message out of the tail.
3. **Background stats.** `start_stats_loader` parses each full file on a daemon thread. It sets the counters, the duration and `stats_loaded`.
4. **On demand.** The first `/` search reads the full text (`ensure_raw`). An expand reads the full conversation (`load_messages`).

`LivePoller` runs `claude agents --json` every 2 s on a daemon thread. Each poll replaces the list, so the UI thread never sees a half-updated list.

## Config

The config file is `~/.config/polyclaude/config.json` (`XDG_CONFIG_HOME` changes the location). Schema: `{"skip_permissions": bool, "background": bool}`. The defaults are `skip_permissions: false` and `background: true`.

`--binary` or `CLAUDE_BINARY` sets the claude binary. The default is `claude`.

## Session data format

Claude Code keeps each session in `~/.claude/projects/<path-with-dashes>/<uuid>.jsonl`. Each line is one JSON object:

- `type`: "user" | "assistant" | "progress" | "file-history-snapshot" | "custom-title" | "last-prompt" | "mode"
- `message.content`: a string, or an array of `{type: "text"}` / `{type: "tool_use"}` / `{type: "tool_result"}` blocks
- `sessionId`, `cwd`, `timestamp` (ISO 8601)
- `isSidechain`, `agentId` for sub-agent messages

**Session names.** The name comes from the last `custom-title` line (`{type: "custom-title", customTitle: "..."}`), because a session can get a new name. Older Claude Code versions kept the name in `~/.claude/sessions/<pid>.json` (`load_session_names`). polyclaude uses that file only when the jsonl has no `custom-title`.

## Conventions

- Keep all code in `main.py`. Split the file only if it gets much larger than 1500 lines.
- Use `safe_addstr()` for all curses writes. It clips the text at the screen edge.
- `init_colors()` defines the color pairs. Use the `C_*` constants.
- Each tab draw function calls `draw_tab_bar()` to draw row 0.
- The main loop uses `stdscr.timeout`: 150 ms while the stats load, 1 s for the live status, no timeout in the expanded view. On a timeout `getch` returns -1. The key handlers must ignore -1.
- The background loader sets the stats. The draw code must accept `stats_loaded == False`.
- To add a setting, add a row to `SETTINGS_ITEMS` (key, label, help text) and a field to `Config`. Update `load_config` and `save_config`.

## Versioning and releases

- Git tags are the only source of the version. `hatch-vcs` reads the version from `git describe`. Do not write a version string in the code.
- `main.py` reads the version at run time with `importlib.metadata.version("polyclaude")`.
- A build between tags gets a dev version, for example `0.5.1.dev3+g<hash>`.
- To make a release: `git tag -a vX.Y.Z -m "..."`, then `git push --tags`. Bump rules: `feat` → minor, `fix` / `chore` → patch, breaking change → major.

## Install and update

- From a local clone: `uv tool install --force --reinstall .`
- From GitHub: `uv tool install git+https://github.com/serejke/polyclaude.git`. Add `@vX.Y.Z` to pin a tag.
- To update from GitHub, add `--reinstall`. uv does not upgrade a git source without it.
