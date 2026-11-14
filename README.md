<h1 align="center">agy-task</h1>

<p align="center">
  <strong>Claude plans. Gemini does the legwork.</strong>
</p>

<p align="center">
  agy-task lets Claude Code hand tasks to Google's <a href="https://antigravity.google.com">Antigravity</a> models.<br>
  Two AI providers in one workflow, and git shows what changed at every handover.
</p>

<p align="center">
  <img alt="License: MIT" src="https://img.shields.io/badge/license-MIT-blue">
  <img alt="Tested with agy 1.5.9" src="https://img.shields.io/badge/tested%20with-agy%201.5.9-4285F4">
  <img alt="Claude Code plugin" src="https://img.shields.io/badge/Claude%20Code-plugin-D97757">
  <img alt="One file of bash" src="https://img.shields.io/badge/one%20file-bash-2ea44f">
</p>

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/workflow-dark.png">
    <img src="assets/workflow-light.png" width="838" alt="A line map of the workflow. Claude Code's line runs straight from plan to commit. Gemini's line leaves it at each handover, does a task, and comes back where Claude checks the result. In the two tasks shown, Gemini Hydrogen audits 11 functions in 47 seconds, and fixes and tests a bug in 20 seconds.">
  </picture>
</p>
<p align="center">
  <sub>Both tasks on the map are real runs, timed on Gemini 4 Hydrogen.</sub>
</p>

## Why two AI providers in one workflow

**Each model does what it is good at.** Claude Code keeps the thread. It plans the change, writes
the handover, checks what comes back and commits. Gemini takes the legwork: searching a codebase,
asking one question of fifty files, running the test suite, a small bounded fix. The two tasks on
the map took it 47 and 20 seconds.

**The legwork runs on a second quota.** Antigravity's models have a quota of their own, so searches
and test runs stop spending the tokens you want Claude to think with. Over one day of use, Gemini
Hydrogen cost about 0.25 % of its weekly quota per 100k tokens.

**You get a second opinion from another lab.** A different model misses different things. Claude
wrote agy-task, and Gemini then reviewed it through agy-task. Fifteen of the findings held up when
checked against the code, and all fifteen are fixed.

**git keeps both honest.** An agent's "done" is a claim, whichever lab trained it. This is a real
answer from `agy -p`, after a run that used 129k tokens (other fields left out):

```json
{
  "status": "SUCCESS",
  "response": "",
  "denied_actions": [{"action": "read_file", "display_name": "ReadFile"}]
}
```

A tool call was denied and the response is empty, yet the status says `SUCCESS`. agy-task exits 4
on that, names what was denied, and prints what git says changed. That is the one account you can
check.

**The agent stays in its lane.** Every task gets a fresh worktree at the ref you name. A read task
denies every file write. Shell commands are denied unless you allowed them by prefix, and that
includes `git commit` and `git push`.

**You can read it before you run it.** One file, about 500 lines of bash, built on git, jq and
standard Linux tools.

> [!WARNING]
> **Your code is sent to Google.** Everything the agent reads goes to the model you pick. Do not
> point it at secrets, `.env` files, production data or anything you would not paste into a chat.
> A fresh worktree has no untracked files, which helps. It does not make an unsuitable repository
> suitable.

## Try it in a minute

You need `agy` installed and signed in.

### From Claude Code

```
/plugin marketplace add kanylbullen/agy-task
/plugin install agy-task@agy-task
```

Then say "offload this to agy", or let Claude decide when a job fits. The plugin ships the script
and a short [skill](skills/agy-task/SKILL.md) that teaches Claude:

- **when to hand over:** mechanical work with an answer that can be checked, such as a codebase
  search, the same question asked of many files, a test run, or a bounded fix with a test command
- **when not to:** secrets, production data, access control, payments, and design decisions
- **how to hand over:** the agent starts cold, so the prompt carries the goal, the files already
  found, the trap, the test command and what done looks like
- **what to trust:** the exit code and the git receipt, read before anything is reported as done

The skill is about 100 lines. Read it before you install the plugin.

### From your shell

```sh
curl -fsSL https://raw.githubusercontent.com/kanylbullen/agy-task/main/bin/agy-task \
  -o ~/.local/bin/agy-task
chmod +x ~/.local/bin/agy-task
```

**Start with a question.** In any git repository:

```sh
agy-task -C . -p "Which source files have no tests? List them, most important first."
```

This is a read task: every file write is denied and only read-only commands are allowed, so the
worst outcome is a wrong answer.

**Then let it change something:**

```sh
agy-task -r ~/src/myapp -b origin/main -w -a "npm test" -n fix-pagination \
  -p "paginate() returns the wrong page for page 1. Fix it and run npm test."
```

That fetches `origin`, creates the branch `agy/fix-pagination` in `~/worktrees/agy-fix-pagination`,
and lets the agent edit files there and run `npm test`, but no other command. The work happens in
the worktree, not in your checkout, and nothing is committed. You read the diff, then commit and
push it yourself.

## What a handover looks like

<p align="center">
  <img src="assets/demo.svg" width="840" alt="Terminal: agy-task fixes a one-line bug in a git worktree in 20 seconds. The git receipt shows one file changed, nothing was denied, and the exit code is 0.">
</p>
<p align="center">
  <sub>The second task on the map, as it ran: bug found, fixed and tested in 20 seconds. Paths shortened, answer trimmed.</sub>
</p>

## When something goes wrong, it says so

A read task was asked to list the exit codes from this README and to report the output of
`python3 --version`, a command that was not on its allow list. The real output, trimmed:

```
-- git (the receipt, not the agent's word) --
no changes
-- denied --
missing command: python3 --version
-- answer --
0: done, nothing denied
...
MISSING COMMAND: python3 --version

agy-task: FAILED: denied: missing command: python3 --version — the answer may be incomplete
```

The exit code was 4. The agent did the part it could and named the command it lacked, instead of
making up what the command would have printed. Models do make that up when nobody tells them not
to (see [the traps](#ten-things-agy--p-will-not-tell-you)).

Exit code `0` is the only finished outcome:

| Code | Meaning |
|---|---|
| 0 | done, nothing denied |
| 2 | bad invocation, a setup failure (fetch, worktree, project) or a pot over its budget; the agent never ran |
| 3 | agy error, or no JSON answer |
| 4 | something was denied, or the agent reported a `MISSING COMMAND`; the answer may be incomplete |
| 5 | empty answer despite `"status":"SUCCESS"` |
| 6 | a read task changed files (the worktree is kept) |
| 7 | the agent was not in the worktree |
| 124 | time limit; any answer is partial |

## The workflow, stop by stop

The map at the top is also how agy-task is built.

1. **Plan.** Claude, or you, writes the task. The agent on the other side starts cold, so the
   handover carries the goal, the files already found, the trap, the test command and what done
   looks like.
2. **Hand over.** agy-task makes a worktree of `REPO` at `REF` (`-r REPO -b REF`). The remote is
   fetched first, so the agent never builds on a stale local branch. A write task gets a branch
   `agy/NAME`. A read task gets a detached worktree that is removed afterwards. `-C DIR` runs in a
   directory as it is. Then the permissions are locked down: read tasks deny every file write, and
   shell commands are denied unless they start with a prefix you allowed with `-a`, plus a few
   read-only defaults (`pwd`, `git status/log/diff/show`, `ls`, `wc`, `rg`). A short preamble tells
   the agent where it is, what it may run, and to write `MISSING COMMAND: …` instead of inventing
   output for a command it was not allowed to run.
3. **Gemini works.** `agy -p` runs with a time limit and JSON output. By default the run can be
   followed live on antigravity.google.com, filed under a project named after the repository.
4. **Check.** agy-task hashes the whole working tree before and after the run, untracked files
   included, and prints the commits and files that changed in between, beside the agent's answer.
   The exit code is non-zero whenever something was denied, missing or wrong.
5. **Commit.** The agent commits nothing. Claude, or you, reads the diff and commits it. Each run
   is also logged to a ledger, so `agy-task -u 7` shows what the week cost per model.

## Numbers

One set of measurements, with the same two tasks on three models. Yours will vary.

| Task | Model | Result | Time | Tokens |
|---|---|---|---|---|
| Read: audit 11 serverless functions for privileged-key use | Argon | 11/11 | 158 s | 94k |
| | **Hydrogen** | 11/11 | **47 s** | **61k** |
| | Flash 3.8 | 11/11 | 161 s | 99k |
| Write: a Rust + Kotlin FFI change with generated bindings | Argon | correct, three rounds | 467 s (first) | 115k (first) |
| | **Hydrogen** | same code; the bindings step was denied (a `&&`, see the traps) | **156 s** | **61k** |

That is why the default model is Hydrogen. It was as accurate as Argon on these tasks, about three
times faster, and it stays off Argon's weekly limit. Use `-m argon` when a job has already failed
on Hydrogen.

**agy-task reviewed itself.** Version 0.1.1 came out of two read tasks run on this repository.
Hydrogen took 105 s and 55k tokens and reported 9 findings. Argon took 499 s and 80k tokens and
reported 22. Fifteen findings held up when checked against the code, and all fifteen are fixed.

### Keep the scarce model for the jobs that need it

The Models & Usage panel in Antigravity shows separate quota pots, and they empty at very different
speeds. On a Google AI Pro plan in November 2026 there were three: Gemini 4 Argon (weekly), Claude
and GPT (weekly plus five-hour), and the other Gemini models, Hydrogen and Flash (weekly plus
five-hour). Measured over one day on that account:

- **Argon** used about 2.8 % of its week per 100k tokens. A single 42-minute write task (683k
  tokens) took about a fifth of the week.
- **Hydrogen and Flash** used about 0.25 % of their week per 100k tokens, roughly a tenth of that.

The panel rounds to whole percent, so these are rough, and a larger plan will have other limits.

Hydrogen is the default, so the bulk of the work already lands on the cheap pot. To keep it that
way, whoever or whatever is calling, give the scarce pot a weekly budget:

```sh
mkdir -p ~/.config/agy-task
echo 'AGY_TASK_BUDGET="argon=700k"' >> ~/.config/agy-task/config
```

Once agy-task has spent that many tokens from a pot in seven days, it refuses further runs on that
pot with exit code 2, before anything is created. `-B` runs one anyway, and `agy-task -u 7` shows
where each budget stands.

**Sizing it.** The budget is the share of the pot you let agy-task spend, not the size of the pot.
Set it to the whole pot and agy-task never refuses anything before Antigravity does. On that AI Pro
account the whole Argon week came to roughly 3.6M tokens, so 700k is about a fifth of it and 1.4M
about two fifths. On a larger plan, scale the number with your limit. The budget counts agy-task's
own runs, not other clients.

## Ten things `agy -p` will not tell you

Each of these cost real time to find. Measured on **agy 1.5.9, November 2026**. agy moves fast, so
if one of them stops being true, please open an issue.

1. **`"status":"SUCCESS"` even when nothing was done.** A denied command gives an empty `response`
   and one entry in `denied_actions`. The status tells you nothing.
2. **The default mode writes files without asking, and `--mode plan` runs nothing.** Plan mode
   writes a plan and waits for an approval that headless never gives. A read-only task therefore
   needs a deny rule, `write_file(*)`, which also stops the edit tools. A hit on that rule does
   **not** appear in `denied_actions`. Only git shows it.
3. **Allow rules are prefixes.** `command(npm test)` allows `npm test -- foo`. `command(npm*)`
   matches nothing, silently.
4. **A command with a shell operator is always denied headless**, whatever rule matches.
   `&&`, `;`, `|` and redirections trigger agy's `unquoted_shell_operators` check, which needs a
   confirmation nobody is there to give. `denied_actions` does not say why. The reason is only in
   `~/.gemini/antigravity-cli/log/cli-<time>.log`. agy-task refuses such `-a` values and tells the
   agent to use `env -C DIR CMD` instead of `cd DIR && CMD`.
5. **Models fabricate output rather than say no.** With `python3` not allowed, one model computed
   `1+1` itself and reported the result as the command's output, with exit code 0. agy-task asks
   for a `MISSING COMMAND:` line instead and exits 4 on it. Give the right `-a` up front.
6. **Half-done work stays.** A file change lands even if the test command after it is denied.
7. **A `--project` name that does not exist falls back silently to "CLI Project"**, and
   `--new-project` names the project after the directory and ignores the name. agy-task writes the
   project file itself in `~/.gemini/config/projects/` and passes the id.
8. **A project must not point at a directory.** The project's folder goes first in the
   conversation's working directories, and `run_command` has no `cwd` of its own. So the agent ran
   commands in the project's folder, the main checkout, instead of the worktree. agy-task creates
   projects with empty `projectResources`, refuses to use one that has folders, and has the agent
   run `pwd` first (exit 7 if it is wrong). A resumed conversation (`-c`) keeps its original
   directories, so the same applies there.
9. **The remote-control daemon reads projects only when it starts.** A project created later shows
   on the page, but its conversations land under "CLI Project" until the daemon restarts. agy-task
   restarts it after creating a project, but only when no other print-mode run is going, because a
   restart cuts off runs that are being followed live.
10. **The remote page names a conversation after the start of its first message.** agy-task opens
    the prompt with `<job name> — <first line of the task>`, so a descriptive `-n` and first line
    are worth a moment.

And two that are not agy's fault:

- **Two fetches in one repo at once fail** with `cannot lock ref`. agy-task queues its own fetch
  and `worktree add` behind a per-repo lock. Another `git fetch` at the same moment can still
  collide.
- **Do not edit the script in place while a job runs.** Bash reads a running script as it goes.
  Write the new version beside it and `mv` it over. That gives the file a new inode, and running
  jobs keep reading the old one.

## Reference

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
  -B               run although the model's pot is over its weekly budget
```

`agy-task -h` has the rest.

**Requirements.** `bash`, git 2.17 or later, `jq`, `flock`, GNU coreutils 8.28 or later (for
`timeout -k`, and for the `env -C` the agent is told to use), `pgrep` (procps) and `agy`, signed
in. Tested on Linux (Debian) with agy 1.5.9. BusyBox systems such as Alpine lack `env -C`. macOS
is untested; you will at least need `brew install flock coreutils`.

**Settings.** From the environment, or as `KEY=VALUE` lines in `~/.config/agy-task/config`, so
that every caller on the machine gets the same ones. The environment wins.

| Variable | Default | |
|---|---|---|
| `AGY_TASK_MODEL` | `hydrogen` | model when `-m` is not given |
| `AGY_TASK_BUDGET` | none | weekly tokens per quota pot, e.g. `argon=700k claude+gpt=300k` |
| `AGY_TASK_WORKTREES` | `~/worktrees` | where worktrees go |
| `AGY_TASK_HOME` | `$XDG_CACHE_HOME/agy-task`, else `~/.cache/agy-task` | logs, ledger, model list |
| `AGY_TASK_DAEMON_UNIT` | `antigravity-cli-daemon` | systemd user unit of the remote-control daemon |

**Logs.** Each run leaves `prompt.txt`, `result.json`, `stdout`, `stderr` and `git.txt` in
`~/.cache/agy-task/<run>/`.

**Following runs in the browser.** `agy remote-control start` registers a daemon that shows your
conversations on antigravity.google.com. agy-task passes `--remote-control` to every run. Turn it
off with `-L`.

**After a write task.** Read the diff in the worktree, then commit, push and open the PR yourself.
Clean up with:

```sh
git -C REPO worktree remove ~/worktrees/agy-NAME && git -C REPO branch -D agy/NAME
```

## FAQ

**Does it commit or push?** No. `git commit` and `git push` are not on the allow list, so the
agent's attempts are denied. Add `-a "git commit"` if you want the agent to commit.

**Can a read task change my files?** Every file write is denied, and the default commands only
read. If something changes anyway, the receipt shows it, the exit code is 6 and the worktree is
kept for you to inspect.

**Which model should I use?** Hydrogen, the default, for nearly everything. Switch with `-m`,
using a short name such as `argon` or `flash-high`, or any id from `agy models`. Argon is for a
job that has already failed on Hydrogen. Its quota is small, so
[give it a budget](#keep-the-scarce-model-for-the-jobs-that-need-it).

**Does it work on macOS?** Untested. The script relies on `flock` and GNU `timeout`, which need
Homebrew there. Reports are welcome.

**Why bash?** So there is nothing to install and nothing to trust that you cannot read in ten
minutes.

## License

MIT. Not affiliated with or endorsed by Google.
