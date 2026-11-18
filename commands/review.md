---
description: Review a pull request, branch or tree with agy-review (six lenses on Antigravity), then check every finding against the code
argument-hint: "[PR number | branch | nothing for the current branch] [extra agy-review options]"
allowed-tools: Bash, Read, Grep, Glob
---

Review with `${CLAUDE_PLUGIN_ROOT}/bin/agy-review`, following the agy-task skill. Target: `$ARGUMENTS`

1. **Work out what to review.** Use the git repository of the current directory unless the
   arguments name another path.
   - A number is a pull request: get its head and base with
     `gh pr view <n> --json headRefName,baseRefName,headRepositoryOwner,isCrossRepository`.
   - A branch name is that branch, against the repository's default branch.
   - No target is the current branch against the default branch.
   Use remote refs (`origin/<branch>`), never local branches, which may lag. Pass `-b` for the head
   and `-d` for the base. Any options in the arguments (`-l`, `-m`, `-M`, `-x`, `-a`) go through
   unchanged.
2. **Check it may be sent.** The code goes to Google. If the repository holds secrets, production
   data or anything the user would not paste into a chat, stop and ask.
3. **Pass what the lenses cannot know.** If this conversation knows what the change is for, a
   decision not to report or a trap you already suspect, write it to a file in the scratchpad and
   pass it with `-x FILE`.
4. **Run it in the background** with `-n review-<pr or branch>` and wait for it to finish; do not
   poll. Leave the model alone unless the user named one: the default is Hydrogen, and `-M claude`
   takes a large share of the Claude pot's five-hour limit.
5. **Check every finding** in the aggregated report against the code before you repeat it. Drop
   what does not hold and say so. Report the findings that held, most severe first, each with
   file:line, and the exit code. On exit 2 nothing ran: say why. On exit 3 name the task that
   failed; its report is in the directory printed at the end.

Do not fix anything and do not comment on the pull request unless the user asks.
