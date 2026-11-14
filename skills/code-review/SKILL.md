---
name: code-review
description: 'Review code or prose against the project''s instruction files and report every deviation with the rule it breaks, marking the trivially fixable ones. Use when asked to "review your changes", "check this against the instructions" or similar.'
---

# Review code against the instructions

Code and prose an agent produces do not always follow the instructions that were in context when they were written: a rule read at the start is forgotten by the end, a component's instruction file was never opened, a wording convention slipped. This skill is a separate pass that reads the result against the project's instruction files and reports every gap. It reviews against the **instructions** (the project's `AGENTS.md` or equivalent at every level, its writing-style rules, its knowledge files, its skills), not against taste. A finding names the rule it breaks. A bug or a broken functional convention found along the way is a finding too.

This skill is about **how to review**: what to load, what to read, what counts as a finding and how to report it. How a review is started, who applies the findings and what is verified afterwards is the project's flow.

**The project's own instructions refine this procedure.** A project may name the instruction files to load and their order, say what a change obliges beyond its lines, define what counts as a trivial fix, or prescribe the report's place in a summary. Read them first, and where they contradict this skill, the project wins.

**This skill only widens a review, never narrows it.** Every rule here sets a floor: a project rule that asks for more is followed, and nothing here is grounds to skip a file, a rule or a finding the project or the brief asks for.

## When this skill applies

Trigger this skill whenever the user asks to:

- "review your changes" / "review what you did" / "review the diff"
- "check this against the instructions" / "does this follow the conventions?"
- "review for writing style" / "check the comments" (a review focused on one instruction file)
- or any equivalent phrasing that asks for code or prose to be checked against how the project says it should be written.

Also trigger it when a project's flow or another agent hands over a review brief (see `The brief` below).

Do **not** trigger this skill for:

- **Addressing `REVIEW:` tags.** That is the [`review-comments`](../review-comments/SKILL.md) skill: executing instructions the user left, not checking the result against the rules.
- **Learning from the user's corrections.** That is [`learn-from-corrections`](../learn-from-corrections/SKILL.md): it reads what the user changed to derive a rule. This skill reads what was written to apply the rules.
- **Reviewing a plan document.** Plans are reviewed by the user with `REVIEW:` tags.

## Review in a fresh context

**When the agent that developed the changes is the one asked to review them, run the review in a new agent or subagent.** The author carries the context that produced the deviation (the rule it skipped, the shortcut it took), and a reviewer with the same context skips the same rule. A fresh agent has no access to the conversation, so it gets everything in a brief (see `The brief`). How the agent is started, and which model and tools it gets, is the project's or harness's business.

**When the review does run in the main context, reload the instruction files it judges against instead of reviewing from memory**, the leading file of a focus above all: a request to "review for writing style" is a request to reload the writing-style rules. Reload the whole file, not the section remembered.

## The brief

A review is defined by its brief, written by whoever starts it. It names:

- **The scope**, in one of three forms:
  - **Specific code**: a list of files, a directory, a module. Every line of it is in scope, old or new.
  - **Uncommitted changes**: everything in the working tree that differs from the last commit, staged and unstaged alike, `git diff HEAD`.
  - **Unstaged changes only**: what is not yet staged, `git diff`.

  "Uncommitted" and "unstaged" are the two words to tell apart. "Your changes" or "the diff" without a qualifier is a question for the user unless the conversation already settled it. In a repository with submodules each submodule has its own index and diff, so the scope lists the repositories together with the files or diff form in each.

- **The focus**, if any. "Review for writing style" or "check the docstrings" names which instruction file leads the review and is judged against first. It does not narrow what is read or what is reported: a deviation from another rule found on the way is still a finding.
- **The instruction files to load**, beyond what the reviewer finds on its own (see step 2 below).

**The reviewer reviews exactly the scope it was given.** Which files a review covers is the caller's decision: other dirty files in the same repository may belong to another session or another task, so the reviewer does not open them to look for more, and it does not drop part of the scope either. When something the brief should say is missing, the report says so at the top and proceeds with what can be inferred, rather than guessing silently.

**A wide scope may be split into several reviews, each with its own brief and focus, run in separate subagents in parallel.** Split by instruction file, by component, or by one of these focuses, each a reading the others tend to skip: the comments and docstrings against the code they describe, the tests against the behavior they claim to cover, the error handling for errors caught and dropped or hidden by a fallback, the new types for the invariants they state and enforce, and the code against the project's guidelines in general. Each reviewer still reports every deviation it finds on the way (see `The focus` above). The caller merges the reports into one flat numbered list, with one entry for a finding several reviewers report.

## Reviewer procedure

### Constraints

- **Report, never edit.** Not the code under review, not an instruction file, not a scratch file. The output is the report. Applying a finding is the caller's flow, in the context that knows what else the task changed.
- **Read-only git.** `git diff`, `git status`, `git show` and `git log` are fine. Nothing that changes the working tree or the index.
- **Do not run the project's build, lint, test or formatting tasks.** The caller runs them after applying the findings.

### Approach

1. **Read the brief.** Note the scope, the focus and the instruction files it names.
2. **Load the instruction files in full.** Start at the project's root instruction file and its writing-style rules, then the instruction file of every component in scope and of every component it builds on (a dependency's rules apply to code written on top of it), then the knowledge files that cover something in scope (e.g. the project's notes on a CLI framework when a CLI is in scope). Read whole files, not sections. When the brief names a focus file, read it first and judge primarily against it. Load every file the project's instructions tell a reviewer to load, in addition to these.
3. **Read the scope in full.** For a file scope, read every named file end to end. For a diff scope, read the whole diff in each repository, not the hunks that look interesting. Read beyond the scope what a rule needs for context: the callers of a changed function, the subclasses of a changed base class, the tests that exercise the changed code, a wrapper module to judge an import path.
4. **Read the neighbors of a change before judging it**: the nearest functions in the same class or module, the other tests in the same file, the existing helper for the same concern. A convention the surrounding code keeps is a rule of the project even when no instruction file states it. A change that diverges from it is a finding that names the neighbor it diverges from, and a new test or helper weaker than the existing ones for the same concern is a finding too.
5. **Check what a change obliges**, beyond its changed lines: whatever the project's instructions say must change together with it, such as a file-level comment that no longer matches the file, a file map whose entry was not updated, a generated file that was not regenerated, a moved symbol that stayed exported with no consumers left, a new third-party import that bypasses the project's wrapper for it, a changed behavior whose test was not updated.
6. **Read every comment and docstring in scope against the code it describes.** A parameter or return the signature no longer has, a behavior the body does not do, a symbol that no longer exists, an example that no longer matches, a `TODO` already done: a claim the code contradicts is a finding like a bug, with no rule needed. Whether a comment that only restates its code is a finding is for the project's writing rules to say.
7. **For every deviation, find the rule.** Quote the instruction file and the section title (e.g. `Examples in prose` in the writing-style rules). A finding that cannot be tied to a rule is an opinion: drop it. A bug is a finding without a rule: a wrong result, an error caught and dropped or replaced by a fallback that hides it, a resource not released on an error path, a test that cannot fail because it asserts a tautology or mocks the code under test. So is a broken functional convention the instructions document.
8. **Let the scope decide what is reviewed, not whether a line is new.** For a diff scope, focus on the lines in the hunks and read the pre-existing lines around them as context for the change: a convention the change breaks shows against them, and a bug they hold is reported (step 11), but their style is not re-reviewed on its own. For a named file or directory, every line is in scope and an old deviation is a finding like any other: the "do not retrofit" clauses of a project's rules do not apply when the user asked for the whole file to be brought up to the instructions.
9. **Give a test-only, refactor, cleanup or doc-only change the same reading as a feature.** A test that asserts the wrong thing gives false confidence, a refactor changes behavior through ordering or error paths, a cleanup removes code that was load-bearing, and a stale example in a doc is copied into new code. Read for what the change could break, not for what its label says it does.
10. **Mark each finding as trivially fixable or not.** A trivial fix is right at a glance, mechanical and reversible in a single hunk: the wording of a comment or a help text, a missing period, formatting or symbol ordering, an un-export or a rename whose every site a reference search confirms. Anything that changes behavior or an API, needs a decision between alternatives or is hard to undo is not trivial. When unsure whether something is a deviation at all, mark it not trivial and say why. The project's flow decides what happens to each kind, e.g. applying the trivial ones at once and proposing the rest. A project's own definition of trivial replaces this one.
11. **Report pre-existing logic bugs and broken functional conventions in the surrounding code**, whatever the scope form, under `Outstanding`. They are not the change's to fix, but a reviewer who read the code and stayed silent about them did not report what was observed.
12. **Check every occurrence.** A rule found broken once is checked across the whole scope, and each site is reported, so the caller fixes all of them and not one. When one rule is broken at many sites, one entry may cover them, as long as it names every site and gives the replacement for each trivial one.

## Output format

Open with one line stating the scope as read: each repository, the diff form or the files, and the count of files read. A misread scope shows there before any finding does.

Then return exactly these sections, in this order, each a numbered list in one flat sequence across the whole report (1, 2, 3 and so on, continuing from section to section) so a reply of "apply 4 and 7" is unambiguous. Say `None.` under a section that is empty rather than omitting it. The report lists problems only: no praise, no note on what was done well.

A finding states what was read, not what the code "might" or "probably" does. Where the evidence is incomplete, say what was checked and what remains unverified, so the caller knows what to confirm. A bug entry names the input or event that triggers it and what then happens, so the caller can confirm it without re-deriving it.

**Each entry ends with its tier and its confidence**, as `[Critical, 95]`:

- `Critical`: a bug, or a change that loses data, opens a security hole or breaks a functional convention the instructions document.
- `Important`: a rule an instruction file states, broken.
- `Suggestion`: a convention the neighbors keep that no file states, or a recommendation among alternatives.

The confidence, 0 to 100, says how sure the reviewer is that the finding is real, given what it read. A low number comes with the reason, as step 10 asks.

**Within each section, order the entries from most to least severe**, `Critical` first, unless the project's instructions or the brief prescribe another order. The confidence does not set the order.

### Trivial

For each: the file and line, the rule (file and section title), and the **exact replacement**: the current text and the new text, both verbatim, so the caller can apply it without judgement.

### Complex

For each: the file and line, the rule, what is wrong in one sentence, and the alternatives with a recommendation. Written so it can be pasted into a proposal list as it is.

### Outstanding

Pre-existing logic or functional-convention problems in the surrounding code, each with its file and line and the rule it breaks. Not style.

### Scope notes

Anything about the scope itself: a file in the brief that does not exist, a repository with no changes, an instruction file the brief named that could not be found, a brief that omitted the focus or the diff form. Then the instruction files that were read in full, so the caller can see what the scope was judged against: only files read in this session, never ones considered and skipped.

Write every entry as `Agent replies` in [`writing-style`](../writing-style/SKILL.md) prescribes: the line as it reads, not a paraphrase; the count, not "several"; what was observed, not what is impossible.

## Anti-patterns

Do not:

- Review from memory of what the instructions say. Load the files, or reload the leading one when reviewing in the main context.
- Report a finding that names no rule. That is an opinion.
- Narrow the given scope, or spill past it into files the brief did not name.
- Read the hunks that look interesting and skip the rest of the diff.
- Re-review the style of pre-existing lines around a hunk when the scope is a diff, or leave an old deviation alone when the scope is a named file.
- Drop a pre-existing bug because it is not the change's to fix. It goes under `Outstanding`.
- Report one occurrence of a deviation and leave the others in the scope unchecked.
- Manufacture a finding so that a section does not say `None.`. A clean scope produces an empty report.
- Praise what was done well. The report lists problems.
- Edit anything, or run a command that changes the working tree, the index or the build state.

## Reference

This skill owns the reading: what a review loads, what it reads, what counts as a finding and how the report is shaped. The project's flow owns the rest: when a review runs, what the brief contains, who applies which findings and what is verified afterwards. Its entries follow the [`writing-style`](../writing-style/SKILL.md) skill, and the rules it checks against live in the project's instruction files, which it never edits.
