# agy-task

Hand a bounded coding task to the [Antigravity CLI](https://antigravity.google.com) (`agy`),
headless, in a git worktree of its own — and get back a receipt from **git**, not the agent's
own account of what it did.

```sh
# Read task: a detached worktree at the remote's main, removed afterwards
agy-task -r ~/src/myapp -b origin/main -n auth-audit -p "List every route that skips the auth middleware."

# Write task: branch agy/fix-x in ~/worktrees/agy-fix-x, left for you to review
agy-task -r ~/src/myapp -b origin/main -w -a "npm test" -n fix-x -p "Fix the off-by-one in paginate() and run npm test."
```

It exists because `agy -p` on its own reports success in situations where nothing happened,
and the traps below each cost real time to find. It is also packaged as a
[Claude Code](https://code.claude.com) plugin, so Claude can offload mechanical work to your
Antigravity quota instead of its own.

> **Your code is sent to Google.** Everything in the worktree the agent reads goes to the model
> you pick. Do not point it at secrets, `.env` files, production data or anything you would not
> paste into a chat. A fresh worktree has no untracked files, which helps; it does not make an
> unsuitable repository suitable.

Not affiliated with or endorsed by Google.

## What it does

1. **Makes a worktree** of `REPO` at `REF` (`-r REPO -b REF`), fetching the remote first so the
   agent never builds on a stale local branch. A write task gets a branch `agy/NAME`; a read task
   gets a detached worktree that is removed afterwards. `-C DIR` runs in a directory as it is.
2. **Locks the permissions down.** Read tasks deny every file write. Shell commands are denied
   unless they start with an allowed prefix (`-a`), plus a few harmless defaults
   (`pwd`, `git status/log/diff/show`, `ls`, `wc`, `rg`).
3. **Tells the agent the rules** in a short preamble: where it is, what it may run, and to write
   `MISSING COMMAND: …` instead of inventing output for a command it was not allowed to run.
4. **Runs `agy -p`** with a time limit, JSON output and, by default, remote control, so you can
   follow the run on antigravity.google.com.
5. **Prints a receipt** — commits and working-tree changes since the base, from git — beside the
   agent's answer, and exits non-zero whenever something was denied, missing or wrong.
6. **Logs one line per run** to a ledger, so `agy-task -u 7` shows what the week cost per quota pot.

## Install

Needs `bash`, git 2.17 or later, `jq`, `flock`, GNU coreutils 8.28 or later (for `timeout -k`, and
for the `env -C` the agent is told to use), `pgrep` (procps) and `agy`, signed in. Tested on Linux
(Debian) with agy 1.5.9. BusyBox systems such as Alpine lack `env -C`. macOS is untested; you will
at least need `brew install flock coreutils`.

**As a standalone command:**

```sh
curl -fsSL https://raw.githubusercontent.com/kanylbullen/agy-task/main/bin/agy-task -o ~/.local/bin/agy-task
chmod +x ~/.local/bin/agy-task
agy-task -h
```

**As a Claude Code plugin** (the script plus a skill that tells Claude when and how to use it):

```
/plugin marketplace add kanylbullen/agy-task
/plugin install agy-task@agy-task
```

The skill runs the plugin's own copy by its full path, so an older `agy-task` elsewhere on your
`PATH` does not get in the way.

**Optional: follow runs from the browser.** `agy remote-control start` registers a daemon that
shows your conversations on antigravity.google.com. agy-task passes `--remote-control` to every
run (turn it off with `-L`) and files each run under a project named after the repository.

## Usage

```
agy-task [options] -p "<task>"      (or -f FILE, or the task on stdin)

  -r REPO -b REF   fresh worktree of REPO at REF (remote fetched first)
  -C DIR           run in DIR as it is
  -w               may write files
  -a PREFIX        allow shell commands starting with PREFIX (repeatable)
  -m MODEL         short name (hydrogen, argon, flash, opus, sonnet … with -low/-medium/-high/-xhigh)
                   or a full id from `agy models`
  -n NAME          job name; -t TIME time limit (default 30m); -s SCHEMA JSON schema for the answer
  -c ID            continue a conversation; -k keep a read task's worktree
  -P PROJECT       Antigravity project; -L no remote control
  -u DAYS          usage per quota pot over the last DAYS days
```

`agy-task -h` has the rest. Settings come from the environment:

| Variable | Default | |
|---|---|---|
| `AGY_TASK_MODEL` | `hydrogen` | model when `-m` is not given |
| `AGY_TASK_WORKTREES` | `~/worktrees` | where worktrees go |
| `AGY_TASK_HOME` | `$XDG_CACHE_HOME/agy-task`, else `~/.cache/agy-task` | logs, ledger, model list |
| `AGY_TASK_DAEMON_UNIT` | `antigravity-cli-daemon` | systemd user unit of the remote-control daemon |

### Exit codes

`0` is the only finished outcome. Everything else means read the log.

| Code | Meaning |
|---|---|
| 0 | done, nothing denied |
| 2 | bad invocation or a setup failure (fetch, worktree, project) — the agent never ran |
| 3 | agy error, or no JSON answer |
| 4 | something was denied, or the agent reported a `MISSING COMMAND` — the answer may be incomplete |
| 5 | empty answer despite `"status":"SUCCESS"` |
| 6 | a read task changed files (the worktree is kept) |
| 7 | the agent was not in the worktree |
| 124 | time limit (agy's own, or the backstop around it); any answer is partial |

The receipt compares the whole working tree before and after the run, untracked files included,
so with `-C` an edit to a file you had already changed still counts.

Each run leaves `prompt.txt`, `result.json`, `stdout`, `stderr` and `git.txt` in
`~/.cache/agy-task/<run>/`.

### After a write task

The agent does not commit unless you allow it (`-a "git commit"`). Read the diff in the worktree,
then commit, push and open the PR yourself. Clean up with:

```sh
git -C REPO worktree remove ~/worktrees/agy-NAME && git -C REPO branch -D agy/NAME
```

## The traps

Measured on **agy 1.5.9, November 2026**. agy moves fast; if one of these stops being true,
please open an issue.

- **`"status":"SUCCESS"` even when nothing was done.** A denied command gives an empty `response`
  and one entry in `denied_actions`. The status tells you nothing.
- **The default mode writes files without asking, and `--mode plan` runs nothing.** Plan mode
  writes a plan and waits for an approval that headless never gives. So a read-only task needs a
  deny rule, `write_file(*)` — and a hit on that rule does **not** appear in `denied_actions`;
  only git shows it.
- **Allow rules are prefixes.** `command(npm test)` allows `npm test -- foo`. `command(npm*)`
  matches nothing, silently.
- **A command with a shell operator is always denied headless**, whatever rule matches.
  `&&`, `;`, `|` and redirections trigger agy's `unquoted_shell_operators` check, which needs a
  confirmation nobody is there to give. `denied_actions` does not say why; the reason is only in
  `~/.gemini/antigravity-cli/log/cli-<time>.log`. agy-task refuses such `-a` values and tells the
  agent to use `env -C DIR CMD` instead of `cd DIR && CMD`.
- **Models fabricate output rather than say no.** With `python3` not allowed, one model computed
  `1+1` itself and reported it as the command's output, with exit code 0. agy-task asks for a
  `MISSING COMMAND:` line instead and exits 4 on it. Give the right `-a` up front.
- **Half-done work stays.** A file change lands even if the test command after it is denied.
- **A `--project` name that does not exist falls back silently to "CLI Project"**, and
  `--new-project` names the project after the directory and ignores the name. agy-task writes the
  project file itself in `~/.gemini/config/projects/` and passes the id.
- **A project must not point at a directory.** The project's folder goes first in the
  conversation's working directories, and `run_command` has no `cwd` of its own, so the agent
  ran commands in the project's folder — the main checkout — instead of the worktree. agy-task
  creates projects with empty `projectResources`, refuses to use one that has folders, and has the
  agent run `pwd` first (exit 7 if it is wrong). **A resumed conversation (`-c`) keeps its
  original directories**, so the same applies there.
- **The remote-control daemon reads projects only when it starts.** A project created later
  shows on the page, but its conversations land under "CLI Project" until the daemon restarts.
  agy-task restarts it after creating a project, but only when no other print-mode run is going,
  since a restart cuts off runs being followed live.
- **The remote page names a conversation after the start of its first message.** agy-task opens
  the prompt with `<job name> — <first line of the task>`, so a descriptive `-n` and first line
  are worth a moment.
- **Two fetches in one repo at once fail** with `cannot lock ref`. agy-task queues its own fetch
  and `worktree add` behind a per-repo lock. Another `git fetch` at the same moment can still collide.
- **Do not edit the script in place while a job runs.** Bash reads a running script as it goes.
  Write the new version beside it and `mv` it over (a new inode; running jobs keep the old one).

## Choosing a model

The Models & Usage panel in Antigravity shows separate quota pots. On a Google AI Pro plan in
November 2026 there were three: **Gemini 4 Argon** (a weekly limit of its own), **Claude and GPT**
(weekly plus five-hour), and **other Gemini models** — Hydrogen and Flash — (weekly plus
five-hour). agy-task records the pot of each run in its ledger. Your plan may differ.

One set of measurements, same two tasks on three models (yours will vary):

| Task | Model | Result | Time | Tokens |
|---|---|---|---|---|
| Read: audit 11 serverless functions for privileged-key use | Argon | 11/11 | 158 s | 94k |
| | Hydrogen | 11/11 | 47 s | 61k |
| | Flash 3.8 | 11/11 | 161 s | 99k |
| Write: a Rust + Kotlin FFI change with generated bindings | Argon | correct, three rounds | 467 s (first) | 115k (first) |
| | Hydrogen | same code; the bindings step was denied (a `&&`, see the traps) | 156 s | 61k |

That is why the default is Hydrogen: as accurate on these, faster, cheaper, and off Argon's
weekly limit. Argon used roughly 2 % of its week per 100k tokens on that account; Claude models
drained their pot several times faster per token. Pick Argon (`-m argon`) when a job has already
failed on Hydrogen or needs more judgement.

## Claude Code skill

`skills/agy-task/SKILL.md` teaches Claude when to offload (mechanical work with a checkable
answer: searching, reading N files with one question, running a test suite, a bounded mechanical
change) and when not to (secrets, production data, design decisions), and to treat git rather
than the agent's answer as the receipt. Read it before installing the plugin — it is short.

## License

MIT
