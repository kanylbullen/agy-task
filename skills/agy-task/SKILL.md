---
name: agy-task
description: Offload a bounded, mechanical coding task to the Antigravity CLI (agy), headless in its own git worktree, and read the result from git. Use when the work has an answer you can check and would otherwise spend this session's tokens — searching a codebase, asking one question of many files, reading logs or CI output, running a test suite and reporting, a small mechanical change with a test command. Not for secrets, production data, security-critical decisions or design choices.
---

# Offloading work to Antigravity with agy-task

`agy-task` runs one task through `agy -p` in a fresh git worktree and prints git's receipt
beside the agent's answer. The work runs on the user's Antigravity quota, not this session's.

Run it as `${CLAUDE_PLUGIN_ROOT}/bin/agy-task`. `${CLAUDE_PLUGIN_ROOT}/bin/agy-task -h` lists
every option.

## When to offload

Work with an answer you can check, where the checking is cheaper than the doing:

- searching a codebase, or asking the same question of N files
- reading and summarising logs, CI output or a long file
- running a test suite and reporting what failed
- a bounded mechanical change with a command that proves it (a rename, porting a finished change
  to a sibling, applying a pattern you have already decided on)

## When not to

- **Anything that must not leave the machine.** The code the agent reads is sent to Google.
  No secrets, `.env` files, credentials, customer or production data. If you are not sure the
  repository may be sent, ask the user before the first run.
- Changes to access control, permissions, payments or migrations: the failures there are
  silent, and "the code looks reasonable" is not a check.
- Design decisions. A task with three reasonable approaches is a choice for the user, not
  something to offload.
- Anything whose result you need to weigh with this conversation's context.
- A write task on a repository whose content you do not trust. A write task runs outside agy's
  sandbox, so the agent's edits are not confined to the worktree, and the receipt covers the
  worktree only.

## Writing the task

The prompt is the whole handover. The agent starts cold and sees nothing of this session.
Include:

- **The goal, and what is out of scope.**
- **The files you already found**, with paths — otherwise it redoes the search.
- **The trap**: what you know that makes a reasonable-looking solution wrong.
- **How to test, verbatim**, and pass the same commands with `-a`.
- **Done when**, in terms you can check from git.

Write the first line as a summary of the task: the remote page names the conversation after it.

## Running it

```sh
# Read task: detached worktree at the remote's branch, removed afterwards
${CLAUDE_PLUGIN_ROOT}/bin/agy-task -r /abs/path/to/repo -b origin/main -n short-name -p "<task>"

# Write task: branch agy/<name> in ~/worktrees/agy-<name>, kept for review
${CLAUDE_PLUGIN_ROOT}/bin/agy-task -r /abs/path/to/repo -b origin/main -w -a "npm test" -n short-name -p "<task>"
```

- `-b` is required with `-r`. Use the remote ref (`origin/main`), not a local branch, which may lag.
- `-a PREFIX` allows shell commands starting with PREFIX. **No shell operators**: `&&`, `;`, `|`,
  `<`, `>` are always denied headless. For another directory use `-a "env -C web npm test"`,
  never `cd web && npm test`.
- A read task may always run `pwd`, `ls`, `wc`, `grep` and `git status`. `git log`, `git diff`,
  `git show` and `rg` need `-a`, and each is a way out: `--output=FILE` writes a file wherever the
  sandbox can reach, and `rg --pre` runs a program. Add them only when the task needs them, on the
  user's own repositories.
- A long prompt goes in a file with `-f FILE`, or on stdin. Past about 120 KB it is refused: agy
  takes the task as a single command-line argument.
- Runs take from ten seconds to several minutes. Run anything longer than a quick read in the
  background and wait for it to finish rather than polling.
- **Stay on the default model, Hydrogen.** Argon draws on a small weekly quota: one long write
  task can take a fifth of it. Use `-m argon` only to retry a job that has already failed on
  Hydrogen, never for a long write task and never as a second opinion.
- **Respect the budget.** If agy-task refuses a run because a quota pot is over its weekly budget
  (exit 2), do not pass `-B` on your own. Run it on the default model, do the work yourself, or
  ask the user.

## Reading the result

**Exit code 0 is the only finished outcome.**

| Code | Meaning | What to do |
|---|---|---|
| 0 | done | check the receipt |
| 2 | bad invocation, setup failure, or a pot over its budget | fix the flags or the repo, or change model; the agent never ran |
| 3 | agy error / no JSON | read `stderr` in the log directory |
| 4 | denied, or `MISSING COMMAND` | the answer may be incomplete; rerun with the right `-a` |
| 5 | empty answer | rerun or rephrase |
| 6 | read task changed files | inspect the kept worktree |
| 7 | agent not in the worktree | start a new run without `-c` |
| 124 | time limit | raise `-t` or split the task |

**The receipt is git, not the agent's answer.** For a write task, read the diff in the worktree
before reporting anything as done. The agent does not commit unless an `-a` prefix covers
`git commit` (`-a "git"` does); committing, pushing and opening a PR are yours to do after review,
and only if the user wants them.

A half-done change stays in the worktree even when the run failed — check before rerunning on
the same branch. Clean up with the command on the receipt's `clean up:` line.

`${CLAUDE_PLUGIN_ROOT}/bin/agy-task -u 7` shows the last week's usage per quota pot.
