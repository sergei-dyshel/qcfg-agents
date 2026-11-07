---
name: commit-changes
description: 'Split the uncommitted changes into commits, one per action, staged through the index and described in the style of the log. Use when asked to "commit changes", "commit this", "split this into commits" or similar.'
---

# Commit changes

Turn the uncommitted changes of a working tree into a short series of commits: one per action, each staged in the index without touching the working tree, each with a message in the style of the project's log.

**The project's own instructions refine this procedure.** A project may name its main branch, the validation to run before committing, its commit message format (a subject prefix, a wrap width) and how to push. Read them first, and where they contradict this skill, the project wins.

## When this skill applies

Trigger this skill whenever the user asks to:

- "commit changes" / "commit this" / "commit everything"
- "split this into commits" / "re-split the last commits"
- or any equivalent phrasing about recording the working tree's changes as commits.

**Commit only when the user asks.** Finishing a task is not a request to commit it.

Do **not** trigger this skill for a commit the user fully specifies, such as the files and the message, or an amend of the last commit. Make it as asked, after the `REVIEW:` check of `Before committing or finalizing` in [`review-comments`](../review-comments/SKILL.md).

## What this skill does

1. Reads every change and checks it before anything is staged.
2. Groups the changes into actions, one commit each, ordered so that each commit stands on its own.
3. Stages each commit in the index from patches, never by editing the working tree.
4. Writes each message in the style of the log and commits.
5. Confirms that the tree is clean, reports the commits and pushes only when asked.

## Procedure

### 1. Read every change

- **Read the whole change set before planning a single commit**: `git status --porcelain --untracked-files=all`, `git diff` and every untracked file in full. Without `--untracked-files=all`, a new directory shows up as one entry and the files inside it go unread.
- **Run the `REVIEW:` check** of `Before committing or finalizing` in [`review-comments`](../review-comments/SKILL.md), and follow that skill on any tag it finds. A tag never reaches a commit.
- **Run the project's validation** (the build, linter or tests its instructions mandate) before the first commit. Fix what fails, or report it and ask before committing.
- **Commit every change in the working tree unless the user names a subset.** Ask before committing a file that looks unrelated to the work, such as an editor setting or a scratch file.

### 2. Group the changes into actions

- **Make as few commits as you can: one per action.** An action is one specific thing the changes do, such as retrying failed requests. Put every edit that serves it into its commit, whatever file the edit is in: the code, its test, its docs and the instruction file that mentions it.
- **Test each commit by its title.** The title names the action and may list its parts: "Retry failed requests with backoff, jitter and a cap" is one commit. A title that joins two actions, "Retry failed requests and rename the config loader", is two commits. Every edit in the commit falls under the action, though the title need not list each one. A broad verb over a broad object, such as "Improve the client", does not turn two actions into one.
- **Order the commits so that each one refers only to what exists by then.** A commit that links a new file comes after the commit that adds it. When one line in the working tree serves two commits, stage an interim wording of it in the earlier one.

### 3. Stage each commit in the index

**Shape the commits in the index, never in the working tree.** The user may go on editing the same files while you commit. An edit you make to the working tree to shape a commit, such as a revert, a stash or a temporary deletion, can lose their work or mix it into the wrong commit.

- **Build each commit from patches.** `git add -p` needs a terminal. Write the chosen hunks of `git diff -U1 <file>` to a patch and stage it with `git apply --cached`. `-U1` splits nearby edits into separate hunks. Pick the hunks by editing the patch, or with `filterdiff --hunks=<n>` where patchutils is installed.
- **To take part of a hunk, drop the other change's `+` lines, turn its `-` lines into context** (a space in place of the `-`) and apply with `--recount`.
- **To take part of an untracked file, start from `git diff --no-index -U1 /dev/null <file>`.** It prints the file as a new-file patch that `git apply --cached` stages. Drop the `+` lines of the part you leave out and apply with `--recount`. The command exits with 1 when it prints a difference, so write it as `git diff --no-index ... > <patch> || [ $? -eq 1 ]` in a script that stops on errors.
- **Stage a whole file with `git add -- <file>`** once every change left in it belongs to this commit.
- **Run the staging and the commit as one `bash -euo pipefail` script**, so that a failed `git apply` stops the script before `git commit` records half a change. `git apply --cached --check <patch>` tests a patch without staging it.
- **Check the staged change before each commit**: `git diff --cached --stat`, and `git diff --cached` in full wherever you edited a patch by hand.

### 4. Write the messages

- **Write each message as the log does.** Read `git log` first for its subject form (imperative mood, a scope prefix, the length) and its body (the wrap width, prose or bullets), and match it. Where the project sets no format, write an imperative subject and a body wrapped at 72 columns.
- **The body says what changed and why**, following `Commit message bodies` in [`writing-style`](../writing-style/SKILL.md).
- **When the diff does not show why a change was made, ask the user instead of guessing.** Collect these questions for all the commits and ask them together, before the first commit.
- **Write each message to a file and commit with `git commit -F <file>`.** It keeps the message's quotes and backticks away from the shell. `awk 'length > 72' <file>` prints every line over a 72-column wrap.

### 5. Finish

- **End with a clean tree**: `git status --porcelain --untracked-files=all` lists only what the user kept out of the commits.
- **To re-split commits that are not pushed yet, `git reset --mixed` to their base first**, which keeps every change in the working tree, then follow this procedure from step 1. `git log @{upstream}..HEAD` lists the commits not pushed yet. Never re-split a pushed commit. Afterwards `git diff <old tip> HEAD` prints nothing.
- **Report the commits**: each one's hash, subject and what it holds, what was left uncommitted and why, and whether anything was pushed.
- **Push only when asked**, the branch to its upstream or as the project's instructions say. Confirm with `git ls-remote <remote> refs/heads/<branch>`, which prints the hash of `HEAD`. When the push fails on authentication, ask the user to renew their credentials, then retry.

## Anti-patterns

Do not:

- Commit, push or re-split commits when the user did not ask.
- Shape a commit by editing, stashing or restoring files in the working tree.
- Join two actions in one commit because they touch the same file, or split one action across commits because it touches several.
- Commit a `REVIEW:` tag, or a file the user did not mean to commit.
- Guess why a change was made when writing its message.
- Re-split a commit that is already pushed.
