---
name: review-comments
description: 'Address `REVIEW:` tags the user has left in dirty (uncommitted) working-tree files and in-progress Markdown docs — inline instructions, or questions about a change. Use when the user asks to "address review comments", "address review", "resolve REVIEWs", "check REVIEW tags", or any equivalent phrasing. Also use before committing (commit, amend, fixup), before creating or updating a PR/CR, and before declaring a plan or design doc final, to check that no `REVIEW:` tags leak through.'
---

# Address `REVIEW:` tags

`REVIEW:` tags are **transient instructions or questions** the user leaves inline in source code and Markdown documents (plans, design docs, notes) for the agent to act on. They live only on dirty (uncommitted) changes in the working tree or in documents still being written — they must never be committed, and must not survive into a finalized document. This skill is the canonical procedure for finding, executing, and removing them.

## When this skill applies

Trigger this skill whenever the user asks to:

- "address review comments"
- "address review"
- "address the REVIEW comments"
- "resolve REVIEWs" / "check REVIEW tags"
- or any equivalent phrasing (in any language) that refers to `REVIEW:`-prefixed tags in the working tree or in a document.

Also, before every commit, before creating or updating a PR/CR, and before a plan or design doc is declared final, run the check in "Before committing or finalizing" below.

Do **not** trigger this skill for:

- **GitHub / GitLab / Code Review platform PR review comments** — those go through the platform-specific skill (e.g. `address-pr-comments`).
- **Any other comment tag** (e.g. `TODO:`, `FIXME:`) — those are long-lived annotations that survive across commits, not transient instructions.
- **Comments in already-committed code** — `REVIEW:` should never appear in clean history; if it does, it's a stale leftover from a previous task and out of scope for the current request.

## What a `REVIEW:` tag looks like

### In code

A comment prefixed with `REVIEW:`, followed by a short instruction or question:

```ts
// REVIEW: rename to parseConfigFile
// REVIEW: drop the cache — the map is rebuilt on every call anyway
// REVIEW: why is the retry needed here? the caller already retries
```

### In Markdown documents

A parenthesized sentence starting with `REVIEW:`. Its position determines what it refers to:

- **On its own line** (a standalone paragraph) — refers to the block **after** it: the next paragraph, list, table, code block, or section.
- **Inline** (mid-paragraph, mid-list-item, mid-sentence) — refers to the **surrounding text**: the sentence or list item it sits in.

```markdown
## Storage layer

(REVIEW: split this section into read and write paths)

- Reads go through `CacheReader`.
- Writes go through `JournalWriter`.

The importer processes files in parallel (REVIEW: is parallel safe here? order may matter) and reports progress per file.
```

## What this skill does

At a high level, this skill:

1. Locates every `REVIEW:` tag across dirty (uncommitted) paths, plus any file the user points at, using **exactly two commands**: `git status` to list dirty files, then a single `grep` invocation that explicitly enumerates those files as arguments.
2. For each tag, performs the requested edit — or, for a question, investigates and answers it — and **deletes the tag in the same change**. The tag _is_ the work item; removing it is how the work is marked done.
3. Builds and tests code changes, then re-scans for `REVIEW:` tags the user may have added **while the agent was working** (often as a reaction to in-flight edits) — and loops until a fresh scan returns zero matches.
4. Reports a summary of how each tag was resolved.

The detailed step-by-step procedure follows below.

## Procedure

Follow these steps in order. Do not skip the re-scan or the summary at the end.

**Hard rule for the scan (steps 1 and 2):** use only the two commands documented below — `git status` (with any flags you like) to list dirty files, then a single `grep` invocation that lists filenames explicitly as arguments: the dirty files, plus any file the user pointed at. **Do not write custom bash scripts, `for` loops, `find`/`xargs` pipelines, `while read` loops, or ad-hoc shell logic** to walk directories, iterate submodules, or discover files. If the scan needs several `git status` invocations (e.g. one per submodule — see below), run each one as its own separate command and manually collect the filenames.

### 1. Find every dirty file

`REVIEW:` tags should only exist on dirty (uncommitted) changes. Get the list of dirty files with `git status`:

```sh
# From the repository root — lists modified/added/untracked paths.
git status --porcelain --untracked-files=all
```

Always pass `--untracked-files=all`. Without it, a new untracked directory shows up as a single `?? dir/` entry instead of its files, so every file inside it — often exactly the new code carrying `REVIEW:` tags — gets skipped in step 2.

**Git submodules.** If the repository is a superproject with git submodules, `git status --porcelain` at the root only reports each dirty submodule as a single ` M <submodule>` entry and does **not** list the individual files inside it. That ` M` also marks a submodule whose checked-out commit merely differs from the recorded one, with nothing inside to scan. `git status --porcelain=v2` at the root tells the two apart: a submodule entry's `S<c><m><u>` field has `M` in the third position for tracked changes and `U` in the fourth for untracked files (e.g. `SCM.`), while `SC..` means only its commit moved. To see the actual file paths, run the same `git status` command **once inside each submodule with `M` or `U` in that field**: <!-- markdownlint-disable-line MD038 -->

```sh
# For each submodule with M or U in its S<c><m><u> field at the root:
cd <submodule> && git status --porcelain --untracked-files=all
```

Run these as separate commands (one per submodule) — do not wrap them in a shell loop. Then manually collect the file paths and prefix them with the submodule directory when passing them to `grep` in step 2.

**Files the user points at.** If the user names a specific file, or a file open in the IDE is obviously the target, add it to the `grep` list even when `git status` doesn't show it. Markdown plans and design docs are often git-ignored (e.g. `doc/*-plan.md`), and `git status --porcelain` never lists ignored files. If the request is clearly about that one file only, scan just that file.

**Do not scan clean, fully-committed code.** If a `REVIEW:` tag ever appears in committed code, it's a stale leftover from a previous task and is _not_ in scope for the current request — leave it alone unless the user explicitly says otherwise.

### 2. Locate every `REVIEW:` tag

Run a **single** `grep` command that lists every file from step 1 explicitly as an argument:

```sh
# Explicitly enumerate the files — no wildcards, no directory scanning.
grep -Hn "REVIEW:" path/to/file1.ts path/to/file2.ts doc/plan.md
```

Rules for this step:

- **List filenames explicitly.** No `grep -r`, no glob expansion over directories, no `find`, no shell loops. The argument list must be the exact set of files from step 1.
- **Skip non-file entries** from `git status` output (deleted files, submodule pointer changes without file paths). Only pass real, existing files to `grep`.
- **One `grep` invocation** is enough even when files come from multiple submodules — just include the full paths for each.

The same pattern catches both the code form and the Markdown `(REVIEW: ...)` form. In code, the token can appear inside any comment syntax the file's language supports — line comments (e.g. `// REVIEW: ...`), block comments, JSDoc-style. Treat all of them the same way. Filter out obvious false positives (e.g. an identifier such as `IS_REVIEW:`) by reading the surrounding code before acting. In particular, when an instruction file is dirty (this skill's own `SKILL.md`, or an `AGENTS.md`/`CLAUDE.md` section that points here), its prose and examples _describing_ the `REVIEW:` mechanism match `grep` too — those are documentation, not work items, and must be skipped.

### 3. Address each tag

**Read all tags before editing anything.** Go through every tag from step 2 in context first. Tags in the same change often relate to each other — one may depend on, overlap with, or contradict another — and handling them as a set avoids redoing edits. Collect the clarifying questions you have for _all_ tags and ask them together, before you start editing, rather than interrupting the work one at a time.

Then, for every `REVIEW:` tag found:

1. **Read the tag in context.** The tag alone rarely carries enough context — `Read` the file around the tag's line to see what it points at.
   - **Code**: read the function, class, or call site the tag sits on.
   - **Markdown**: first check whether the tag is **standalone** or **inline** (see "In Markdown documents" above). A standalone tag refers to the block immediately after it — the preceding block is usually irrelevant. An inline tag refers to the sentence or list item it sits in — read that unit, not neighboring blocks.
2. **Act on it according to its kind:**
   - **Instruction** (`REVIEW: rename X to Y`, `REVIEW: drop the cache`) — perform the requested edit. Apply the same rigor as any other code change (types, formatting, existing conventions, related call sites).
   - **Question** (`REVIEW: why is the retry needed?`, `REVIEW: is this safe?`) — the user is questioning a particular edit or change. Investigate the question and try to answer it: read the related code, callers, tests, history, and docs. If the investigation shows there is a problem and a fix is needed, make the fix. If there is no problem, just remove the tag. Either way, a question usually means the change isn't self-explanatory, so in most cases add a short comment (in Markdown, a sentence) that explains it. Give the answer in the summary (step 6).
3. **Ask for clarification** if an instruction is ambiguous, or if investigating a question can't settle it — up front with the rest of your questions when you can (e.g. it comes down to a decision only the user can make). Follow the "Handling ambiguity and uncertainty" rules in [`context/handling-ambiguity.md`](../../context/handling-ambiguity.md) — do not guess.
4. **Remove the tag** as part of the same edit. The tag is the work item; removing it is how the work is marked done.
   - **Code**: delete the comment. If the resolution needs a lasting explanation, replace the tag with a normal comment explaining _why_ — only when the reason isn't obvious from the code; otherwise remove the tag with no replacement. Never leave the tag behind "for reference", never leave breadcrumbs like `// resolved REVIEW: ...`, and never rewrite it into a `TODO:`.
   - **Markdown**: if the tag is standalone, delete the entire `(REVIEW: ...)` paragraph plus its trailing blank line. If it's inline, delete only the `(REVIEW: ...)` fragment and the space in front of it, so the surrounding sentence stays grammatical. If the resolution changed the surrounding prose, update the prose in the same edit. Never leave breadcrumbs like `(RESOLVED: ...)` — the document should read as if the tag never existed.

If addressing one tag naturally subsumes another nearby tag (e.g. a rename that also satisfies a related note), remove both — but only if you're confident both requests are satisfied.

### 4. Build and test (code only)

Renames, signature changes, and changes to the shape of types or structs usually touch several files. Once all code tags from this pass are addressed — not after each one — run the project's build or type check, and re-run the tests that cover the changed code, to catch call sites you missed and confirm nothing regressed. Markdown-only resolutions don't need a build.

### 5. Re-scan after finishing

The user often adds **new** `REVIEW:` tags **while the agent is working**, typically as a reaction to in-flight edits. After you believe you've addressed every tag from step 2:

1. Repeat step 1 (re-run `git status --porcelain --untracked-files=all` at the root and inside each submodule with `M` or `U` in its `S<c><m><u>` field) — the set of dirty files may have grown. Keep any file the user pointed at in the list.
2. Repeat step 2 with the updated file list (single `grep` invocation, filenames listed explicitly).
3. If any `REVIEW:` matches remain, go back to step 3 and address them (and step 4 if code changed).
4. Loop until a fresh `grep` returns **zero** matches.

Only then move on to the summary.

### 6. Report a summary

The summary is the deliverable of this flow — it's what the user reads to answer "what did you change?". Do **not** skip it, and do **not** collapse it into a one-liner like "done" or "all REVIEWs resolved". Write a Markdown section with one bullet per tag that was in scope:

- Each bullet starts with a **clickable Markdown link** to the exact location of the resolved content, with the path relative to the workspace root: `[file.ts:42](src/file.ts#L42)`. Use the **new**, post-edit line number — the line the resolved content sits on now, not the line the tag used to be on.
- When one resolution fanned out to several locations (e.g. a rename touching several files), list all of them as links on the same bullet.
- Follow the links with a one-sentence description of what changed and why. For a **question**, give the answer the investigation found, and say whether it led to a fix, an explanatory comment, or neither.
- If a tag was **not** applied (the user deferred it, dropped it, or chose to keep it), still list it with its disposition rather than omitting it.
- Keep it compact — it's a report, not a re-rendering of the diff.

Example:

```markdown
### REVIEW resolutions

- [config.ts:42](src/config.ts#L42), [cli.ts:17](src/cli.ts#L17) — renamed `loadCfg` to `parseConfigFile` and updated its caller.
- [fetch.ts:88](src/fetch.ts#L88) — "why is the retry needed here?": it isn't; the caller already retries, so removed the inner retry loop.
- [plan.md:15](doc/plan.md#L15) — "is parallel safe here?": yes, each file is imported independently; added a sentence saying so.
- `REVIEW: reconsider the retry cap` — **deferred** by the user; tag left in place.
```

## Leaving tags alone during unrelated work

Never address or remove a `REVIEW:` tag on your own initiative — not while working on an unrelated request, not during a drive-by cleanup, not as part of a `simplify` / `code-review` pass. A tag can hold a decision the user hasn't made yet; resolving it silently throws that decision away. Only address tags when the user explicitly asks (see "When this skill applies").

If an unrelated edit (e.g. "rename X" or "extract Y") moves or rewrites code that carries a pending tag, keep the tag attached to the code it refers to — move it along with that code — and mention it in your reply so the user knows it's still pending. If the code it refers to is removed entirely, don't silently delete the tag; point it out and ask.

## Before committing or finalizing

These checks are separate from the scan in steps 1–2: they look at what's about to be committed or handed off, so piping `git diff` into `grep` is fine here. Apply the same false-positive filtering as in step 2.

### Code

Before **every** `git commit` — including amends, `git commit --fixup`, and commits made during a rebase — scan the staged changes:

```sh
git diff --cached | grep -E '^\+.*REVIEW:'
```

For `git commit -a`, check `git diff HEAD` instead, since unstaged changes get committed too.

Before creating or updating a PR/CR, run the same check over the whole branch to catch tags that slipped into earlier commits:

```sh
git diff <base>..HEAD | grep -E '^\+.*REVIEW:'
```

Where `<base>` is the branch's fork point from the main branch, or the previous revision's base when updating.

### Markdown documents

When a plan or design doc goes from in-progress to ready to hand off (the user asks to share or publish it, attach it to a PR/CR, or says it's done), scan the file:

```sh
grep -n 'REVIEW:' path/to/plan.md
```

### If any tag is found

- **Stop and surface it to the user.** Do not silently go ahead with the commit, PR/CR, or handoff.
- Report each tag as `file:line — REVIEW text`.
- Ask whether to address or drop it. Only commit or hand off with a tag still in place if the user explicitly says to.

## Anti-patterns

Do not:

- Commit code that still contains `REVIEW:` tags — run the check in "Before committing or finalizing" below first.
- Convert `REVIEW:` into another comment tag (e.g. `TODO:`), or leave breadcrumbs like `resolved REVIEW:` / `(RESOLVED: ...)` — the whole point is that the tag disappears once addressed.
- Address or remove tags on your own initiative while doing unrelated work.
- Remove a question tag without investigating it, or answer it only in chat when the code needs a fix or an explanatory comment.
- Scan the entire repository or clean/committed files for `REVIEW:` — restrict the scan to dirty paths per step 1, plus files the user points at.
- Batch-address every `REVIEW:` tag without reading its surrounding context — each one is a targeted instruction that depends on context.
- Skip the final re-scan (step 5). The user's follow-up tags are often the most important ones because they react to what you just did.
- Skip the summary (step 6) or reduce it to "done".
- Write custom shell scripts, `for`/`while` loops, `find`/`xargs` pipelines, or any ad-hoc bash logic to discover dirty files. The scan is always exactly `git status` (possibly once per submodule) followed by a single `grep "REVIEW:" <file> <file> ...` with filenames listed explicitly.
- Use `grep -r`, glob wildcards, or directory arguments to `grep`. Passing a directory (or `.`, or `**/*.ts`) is the same class of mistake as writing a shell loop — it scans files that were not in the `git status` output.

## Reference

This skill is the authoritative source for how `REVIEW:` tags are handled. Any `AGENTS.md`/`CLAUDE.md` that mentions `REVIEW:` tags should keep only a short pointer here — all detailed rules live in this file.
