# claude-resume-all

Reopen **every** [Claude Code](https://claude.com/claude-code) session of a project at once — each in its own terminal — instead of picking them one at a time from `/resume`.

> 日本語版は [README.ja.md](README.ja.md) を参照してください。

## Why

Claude Code's `/resume` (and `claude --resume`) brings back **one** session at a time. If your editor or terminal crashes while you have several Claude threads going for a project, restoring them one by one is tedious.

`claude-resume-all` finds **all** the sessions stored for the current project and fans them out into separate terminals in a single command.

```console
$ claude-resume-all
Project : /Users/me/git/app
Profile : personal  (/Users/me/.claude-personal)
Launcher: claude
Sessions: 7
  ed861b4f  Add a photo-upload feature tied to place names
  cb5cafea  Set publish:true once fact-checking is done
  ...
Open 7 terminal window(s)? [y/N] y
Launched 7 session(s) via claude.
```

## How it works

Claude Code stores each session as a `*.jsonl` file (the filename is the session id) under:

```
<config-dir>/projects/<encoded-project-path>/<session-id>.jsonl
```

where `<config-dir>` defaults to `~/.claude`, and `<encoded-project-path>` is the
project's absolute path with every non-alphanumeric character replaced by `-`
(e.g. `/Users/me/git/app` → `-Users-me-git-app`).

`claude-resume-all` discovers those directories, lists the sessions for your
project, and launches `claude --resume <id>` for each one in a fresh terminal.

## Install

Requires **zsh** (macOS default). `python3` is optional but recommended (used to
show a human-readable label per session and for the `vscode` backend).

```sh
git clone https://github.com/nakamura196/claude-resume-all.git
ln -s "$PWD/claude-resume-all/bin/claude-resume-all" ~/.local/bin/claude-resume-all
# make sure ~/.local/bin is on your PATH
```

(or copy `bin/claude-resume-all` anywhere on your `PATH`).

## Usage

```sh
claude-resume-all                 # all sessions for $PWD, profile auto-detected
claude-resume-all /path/to/proj   # another project directory
claude-resume-all -p work         # force a named profile
claude-resume-all --config-dir ~/.claude-x   # an explicit config dir
claude-resume-all --recover       # only sessions killed by a crash (see below)
claude-resume-all --recent 3h     # only sessions active in the last 3 hours
claude-resume-all --since 14:30   # only sessions active since a time today
claude-resume-all --order old     # open oldest-first (last-worked ends frontmost)
claude-resume-all --recursive     # also sessions started in sub-directories
claude-resume-all --list          # list sessions, open nothing
claude-resume-all --app iterm     # terminal | iterm | vscode | print
claude-resume-all -y              # skip the confirmation prompt
claude-resume-all --help
```

### Crash recovery — resume only what was force-closed (`--recover`)

After a force-quit you usually want back the sessions that were **open when it
died** — not every session you ever ran, and not the ones you'd cleanly exited.

While a session runs, Claude Code writes `<config-dir>/sessions/<pid>.json`
(holding its `sessionId` and `cwd`). A clean exit removes that file; a
crash/force-quit leaves it behind. So **"file present but pid dead" = a session
that was open when things died.** `--recover` resumes exactly those — skipping
cleanly-exited *and* still-running ones — and uses the recorded `cwd` (which also
sidesteps the path-encoding ambiguity that trips up sub-directory sessions).

```sh
claude-resume-all --recover                 # crashed sessions of the current project
claude-resume-all --recover --all-projects  # everything that was force-closed
```

> ⚠️ Run it **soon** after a crash. Claude Code periodically sweeps stale
> registry entries; once they're gone, fall back to `--recent` / `--since`, which
> filter on each session file's last-modified time and don't disappear.

### Filtering and ordering

| flag | effect |
|------|--------|
| `--recent <N>` | only sessions active within `90m` / `3h` / `1d` |
| `--since <when>` | only sessions active since `HH:MM` (today) or `YYYY-MM-DD HH:MM` |
| `--order recent` | open newest-first (**default**) |
| `--order old` | open oldest-first, so the last-worked session opens last and ends up frontmost |
| `--recursive` / `-r` | also include sessions started from sub-directories of the project |
| `--all-projects` | (with `--recover`) ignore the project filter |

### Terminal backends (`--app`)

| value      | behaviour |
|------------|-----------|
| `terminal` | macOS Terminal.app, one window per session (**default**) |
| `iterm`    | iTerm2, one window per session |
| `vscode`   | write a **transient** `.vscode/tasks.json` and have you run the generated *Resume ALL* task once inside VS Code (see below) |
| `print`    | just print the resume commands — pipe into tmux / WezTerm / your own launcher |

`terminal`, `iterm`, and `vscode` are macOS-only (they use `osascript`).
`print` works on any platform.

### Opening sessions inside VS Code

External CLIs **cannot** spawn VS Code integrated terminals directly, and VS Code
will only run tasks defined in `<project>/.vscode/tasks.json`. So `--app vscode`:

1. writes a `.vscode/tasks.json` containing one dedicated-terminal task per
   session, plus a compound **▶ Resume ALL (<profile>)** task (any tasks you
   already had are preserved);
2. tells you to run that task once
   (`Cmd+Shift+P` → *Tasks: Run Task* → *▶ Resume ALL …*);
3. waits — press **Enter** after the terminals have opened and it
   **removes/restores** `tasks.json`. The terminals keep running, so nothing is
   left in your repo. Pass `--keep` to leave the file in place.

## Profiles

A **profile** is just a Claude config directory. `claude-resume-all` auto-detects
them from `~/.claude` and any `~/.claude-*` directory, using the basename as the
profile name (`~/.claude` → `default`, `~/.claude-work` → `work`).

If a project has sessions under more than one profile, the tool asks you to pick
one with `-p`. No profile names or paths are hard-coded — your setup is
discovered at runtime.

### Optional config file

`${XDG_CONFIG_HOME:-~/.config}/claude-resume-all/config` — all keys optional:

```ini
# default terminal backend
app=terminal
# default executable to run
launcher=claude
# register an extra profile whose config dir is in a non-standard place
profile.foo=~/some/where/.claude-foo
# per-profile launcher (e.g. your own shell alias)
launcher.foo=claude-foo
```

Environment overrides: `CLAUDE_RA_APP`, `CLAUDE_RA_LAUNCHER`, `CLAUDE_CONFIG_DIR`,
`CLAUDE_BIN`.

## Caveats

- macOS only for the GUI backends; `--app print` is portable.
- The first time, macOS may ask to allow your terminal to control Terminal.app /
  iTerm (Automation permission).
- This reads Claude Code's on-disk session layout, which is **not a documented,
  stable API** — a future Claude Code release could change it.

## License

[MIT](LICENSE)
