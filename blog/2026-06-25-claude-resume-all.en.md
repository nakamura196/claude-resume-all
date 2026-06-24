# Reopening every Claude Code session at once

*2026-06-25*

[Claude Code](https://claude.com/claude-code) has a `/resume` picker that lets
you jump back into a past session. It's great — until your editor crashes and
you realize you had **seven** sessions going for one project. `/resume` only
reopens one at a time, and there's no built-in "resume all."

So I built [`claude-resume-all`](https://github.com/nakamura196/claude-resume-all):
one command that finds every session for a project and fans them out into
separate terminals. This is the story of how it works and the handful of macOS
gotchas I hit along the way.

## Where Claude Code keeps sessions

Everything starts with one question: where are sessions stored? A little
spelunking turns up this layout:

```
<config-dir>/projects/<encoded-project-path>/<session-id>.jsonl
```

- `<config-dir>` defaults to `~/.claude`. If you set `CLAUDE_CONFIG_DIR`, it
  points there instead — handy for keeping separate "personal" and "work"
  histories.
- `<encoded-project-path>` is the project's absolute path with every
  non-alphanumeric character replaced by `-`. So `/Users/me/git/app` becomes
  `-Users-me-git-app`.
- Each `*.jsonl` file is one session, and the **filename is the session id** you
  feed to `claude --resume <id>`.

That's the whole key to the tool. List the `.jsonl` files for your project's
encoded path, and you have every resumable session.

```sh
ENC=$(print -r -- "$PWD" | sed 's/[^a-zA-Z0-9]/-/g')
ls -t "$HOME/.claude/projects/$ENC"/*.jsonl
```

For a nicer listing, each `.jsonl` is a stream of JSON objects; the first
`type:"user"` line gives you a label to show next to the id.

## Fanning out into terminals

On macOS, the most reliable way to open N terminal windows and run a command in
each is AppleScript via `osascript`:

```sh
osascript -e 'tell application "Terminal" to do script "claude --resume <id>"'
```

Loop that over the session ids and you get one window per session. iTerm2 has an
equivalent `create window ... write text` incantation.

### Gotcha #1: the windows open, you just can't see them

The first time a user ran it, the script proudly printed `Launched 7 session(s)`
— but no windows appeared. Were they not opening?

```sh
$ osascript -e 'tell application "Terminal" to count windows'
19
```

They were opening fine. The catch: `do script` creates windows but doesn't bring
Terminal to the front, and the user was in **full-screen VS Code on its own
macOS Space**. The windows were all there, on another Space, invisible. One line
fixes it:

```sh
osascript -e 'tell application "Terminal" to activate'
```

### Gotcha #2: `command not found: claude`

I first tried the VS Code route (more on that below), generating tasks that ran
`claude --resume …`. They failed:

```
zsh:1: command not found: claude
```

VS Code runs task commands in a **non-interactive login shell**
(`/bin/zsh -l -c '…'`). A login shell sources `.zprofile`/`.zlogin` but **not**
`.zshrc`, where most people put their `PATH` tweaks and aliases. So neither
`claude` (installed under `~/.local/bin`) nor a personal `claude-personal` alias
existed in that shell.

The fix is to never rely on interactive-shell state for spawned commands:
resolve the real binary with `command -v claude` and pass the config directory
explicitly as an environment variable, rather than leaning on an alias.

```sh
CLAUDE_CONFIG_DIR=/Users/me/.claude-personal /Users/me/.local/bin/claude --resume <id>
```

(Plain Terminal.app windows *are* interactive, so aliases work there — but
making the tool alias-independent means it behaves the same everywhere.)

## The VS Code problem (and a clean trick)

Lots of people, me included, live inside VS Code and want the sessions in its
**integrated** terminals, not a separate app. Here's the wall you hit:

> There is no public CLI to spawn a VS Code integrated terminal running a
> command. VS Code will only run tasks defined in `<project>/.vscode/tasks.json`.

So the only path is to write a `tasks.json` with one dedicated-terminal task per
session plus a compound task that runs them all in parallel, then trigger it once
from inside VS Code (`Tasks: Run Task`).

But that file is committed project config — leaving a generated `tasks.json`
lying around pollutes the repo. The fix is a small insight:

> Once a task has spawned its integrated terminal and `claude` is running, the
> terminal keeps running even if you delete `tasks.json`.

So the tool treats the file as **transient**: it backs up any existing
`tasks.json`, writes the generated one, waits while you run the task, and then —
on a single Enter — restores your original (or removes the file and the
`.vscode/` dir if it created them). The terminals stay open; the repo stays
clean. A `--keep` flag opts out.

The generator also merges instead of clobbering: it preserves any tasks you
already had and only replaces the ones it owns (it tags them with a marker
prefix), so re-running is idempotent.

## No personal paths baked in

My own machine has three config dirs (`~/.claude`, `~/.claude-personal`,
`~/.claude-work`) behind shell aliases. The tempting thing is to hard-code those
"profiles." But that's *my* setup, and this is a public tool.

Instead it **discovers** profiles at runtime by globbing `~/.claude` and
`~/.claude-*`, using each directory's basename as the profile name. On my machine
that finds personal and work automatically; on yours it finds whatever you have
— with zero personal paths in the source. If a project has sessions under more
than one profile, it asks you to pick with `-p`. A optional config file lets
power users register config dirs in odd locations or map a profile to their own
alias.

## Bonus gotcha: the classic awk `sub()` trap

The `--help` output came out empty. The culprit was a one-liner meant to print
the leading comment block:

```awk
NR>1 && /^#/{sub(/^# ?/,"");print}  NR>1 && !/^#/{exit}
```

awk evaluates *both* rules for each line. The first rule's `sub()` strips the
`#` from `$0` — and then the second rule tests the **modified** `$0`, which no
longer starts with `#`, so it `exit`s after the very first line. The fix is to
short-circuit with `next`:

```awk
NR==1{next}  /^#/{sub(/^# ?/,"");print;next}  {exit}
```

## Takeaways

- Claude Code's session layout is simple and scriptable — but it's an
  undocumented on-disk format, so treat it as something that could change.
- On macOS, `osascript` is the lever for driving Terminal.app and iTerm; remember
  to `activate`.
- Spawned commands shouldn't depend on interactive-shell state (PATH, aliases).
- You can't drive VS Code's integrated terminals from outside, but a transient,
  self-cleaning `tasks.json` gets you a clean equivalent.

The tool is on GitHub:
[nakamura196/claude-resume-all](https://github.com/nakamura196/claude-resume-all).
