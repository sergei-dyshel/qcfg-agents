---
name: code-review
description: 'Review code or prose against the project''s instruction files and report every deviation with the rule it breaks, marking the trivially fixable ones. Use when asked to "review your changes", "check this against the instructions" or similar.'
---

# Review code against the instructions

Code and prose an agent produces do not always follow the instructions that were in context when they were written: a rule read at the start is forgotten by the end, a component's instruction file was never opened, a wording convention slipped. This skill is a separate pass that reads the result against the project's instruction files and reports every gap. It reviews against the **instructions** first (the project's `AGENTS.md` or equivalent at every level, its writing-style rules, its knowledge files, its skills). A finding names the rule it breaks. A bug or a broken functional convention found along the way is a finding too, and so is a design smell the reviewer judges worth raising, at the lowest tier.

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

- **The scope**, in one of four forms:
  - **Specific code**: a list of files, a directory, a module. Every line of it is in scope, old or new.
  - **Uncommitted changes**: everything in the working tree that differs from the last commit, staged and unstaged alike, `git diff HEAD`.
  - **Unstaged changes only**: what is not yet staged, `git diff`.
  - **Changes since a fixed point**: a commit, branch or tag the brief names, `git diff <ref>...HEAD` (three dots, so the comparison is against the merge base) together with the commit messages of `git log <ref>..HEAD`, which state the intent of the change and are prose in scope like any other.

  "Uncommitted" and "unstaged" are the two words to tell apart. **Do not start a review until its scope is exact**: every repository, and in each the files or the diff form. "Your changes" or "the diff" without a qualifier is a question for the user unless the conversation already settled it. In a repository with submodules each submodule has its own index and diff, so the scope lists the repositories together with the files or diff form in each.

- **The focus**, if any. "Review for writing style" or "check the docstrings" names which instruction file leads the review and is judged against first. It does not narrow what is read or what is reported: a deviation from another rule found on the way is still a finding.
- **A description of the change**, in the caller's words: what was built and why, in a few sentences. It is the reviewer's only view of the intent behind the diff when the brief names no task. It is not the conversation that produced the change: a reviewer handed the session's history inherits the author's blind spots, which the fresh context exists to avoid (see `Review in a fresh context`).
- **The task**, if any: the issue, spec or user request the change implements, by path or quoted in the brief. The task is then an instruction file of the review: a requirement it states that the change leaves out or does in part, a behavior the change adds that it did not ask for, and a requirement the change implements wrongly are findings, each quoting the task's line as its rule. The task states what the change must do, not every input, environment or condition it meets. Where it is silent, judge by what a user of the software would expect: a missing case a user would count on is a finding whose rule is that expectation, tiered by its effect on the user, not by whether the task names it. A defect in the task itself, such as two requirements that contradict each other or one the code shows cannot be met as stated, is reported under `Scope notes`, since it is the caller's to resolve with the user, not a finding against the change. When the brief names no task and the commit messages or the conversation point at one, ask for it before reviewing rather than guessing.
- **The instruction files to load**, beyond what the reviewer finds on its own (see step 2 below).

**The reviewer reviews exactly the scope it was given.** Which files a review covers is the caller's decision: other dirty files in the same repository may belong to another session or another task, so the reviewer does not open them to look for more, and it does not drop part of the scope either.

**Do not review with context missing.** When the brief lacks something the review depends on, e.g. the intent of the change when it names neither a task nor a description, ask the caller for it before reading the scope, rather than reviewing with what can be inferred. A reviewer that cannot ask, such as a subagent, returns a report that holds only `Scope notes`, naming what is missing.

**A wide scope may be split into several reviews, each with its own brief and focus, run in separate subagents in parallel.** Split by instruction file, by component, or by one of these focuses, each a reading the others tend to skip: the comments and docstrings against the code they describe, the tests against the behavior they claim to cover, the error handling for errors caught and dropped or hidden by a fallback, the new types for the invariants they state and enforce, the change against the task it implements, and the code against the project's guidelines in general. Each reviewer still reports every deviation it finds on the way (see `The focus` above). Before spawning the reviewers, confirm the scope resolves and is not empty: `git rev-parse <ref>` for a fixed point, a non-empty `git diff` for a diff form, an existing path for each named file. A bad ref fails once in the caller, not in every subagent. The caller merges the reports into one flat numbered list, with one entry for a finding several reviewers report. A reviewer may spawn such subagents itself to hunt one kind of problem across its scope, and merges their findings into its own report the same way.

## Reviewer procedure

### Constraints

- **Report, never edit.** Not the code under review, not an instruction file, not a scratch file. The output is the report. Applying a finding is the caller's flow, in the context that knows what else the task changed.
- **Read-only git.** `git diff`, `git status`, `git show` and `git log` are fine. Nothing that changes the working tree or the index, and no clone, checkout or worktree of another revision without the user's confirmation.
- **Do not build the change or run its tests, linters or formatters.** The reviewer reads the code and does not fix it: building and testing the change is the job of its author and of the project's CI.

### Approach

1. **Read the brief.** Note the scope, the focus, the task and the instruction files it names, and ask for what is missing before going on (see `The brief`).
2. **Load the instruction files in full.** Start at the project's root instruction file and its writing-style rules, then the instruction file of every component in scope and of every component it builds on (a dependency's rules apply to code written on top of it), then the knowledge files that cover something in scope (e.g. the project's notes on a CLI framework when a CLI is in scope), then the task when the brief names one. Read whole files, not sections. When the brief names a focus file, read it first and judge primarily against it. Load every file the project's instructions tell a reviewer to load, in addition to these.
3. **Read the scope in full.** For a file scope, read every named file end to end. For a diff scope, read the whole diff in each repository, not the hunks that look interesting. Read beyond the scope what a rule needs for context: the callers of a changed function, the subclasses of a changed base class, the tests that exercise the changed code, a wrapper module to judge an import path.
4. **Read the neighbors of a change before judging it**: the nearest functions in the same class or module, the other tests in the same file, the existing helper for the same concern. A convention the surrounding code keeps is a rule of the project even when no instruction file states it. A change that diverges from it is a finding that names the neighbor it diverges from, and a new test or helper weaker than the existing ones for the same concern is a finding too.
5. **Check what a change obliges**, beyond its changed lines: whatever the project's instructions say must change together with it, such as a file-level comment that no longer matches the file, a file map whose entry was not updated, a generated file that was not regenerated, a moved symbol that stayed exported with no consumers left, a new third-party import that bypasses the project's wrapper for it, a changed behavior whose test was not updated.
6. **Read every comment and docstring in scope against the code it describes.** A parameter or return the signature no longer has, a behavior the body does not do, a symbol that no longer exists, an example that no longer matches, a `TODO` already done: a claim the code contradicts is a finding like a bug, with no rule needed. Whether a comment that only restates its code is a finding is for the project's writing rules to say.
7. **For every deviation, find the rule.** Quote the instruction file and the section title (e.g. `Examples in prose` in the writing-style rules), or the task's line when the task is the rule. Skip a deviation the project's formatter, linter or compiler will fix or reject: the change's author and the CI run them, and a report that repeats them is noise. A bug is a finding without a rule: a wrong result, an error caught and dropped or replaced by a fallback that hides it, a resource not released on an error path, a test that cannot fail because it asserts a tautology or mocks the code under test. So is a broken functional convention the instructions document. A finding with no rule, no bug and no smell behind it is the reviewer's own judgement: it may stay, as a `Suggestion` that says so.
8. **Name the design smells the change introduces, as `Suggestion` entries.** A smell is a heuristic, never a violation, so it is named as "possible <smell>" with the hunk quoted and the fix it points at. A project rule that endorses the construct suppresses the smell. The smells agent-written code produces most:
   - **Duplicated code**: the same logic shape in more than one hunk or file of the change. Extract it and call it from both.
   - **Speculative generality**: an abstraction, parameter or hook for a need the task does not have. Delete it and inline until a real need shows.
   - **Mysterious name**: a function, variable or type whose name does not say what it does or holds. Rename it, and when no honest name comes, the design is unclear.
   - **Data clumps**: the same few fields or parameters travelling together. Bundle them into one type.
   - **Primitive obsession**: a primitive or string standing in for a domain concept. Give the concept its own small type.
   - **Middle man**: a class or function that mostly delegates onward. Cut it and call the target directly.
   - **Message chains**: a long `a.b().c().d()` walk the caller should not depend on. Hide it behind one method on the first object.
   - **Shotgun surgery**: one logical change forcing scattered edits across many files of the diff. Gather what changes together into one module.
   - **Divergent change**: one file or module edited for several unrelated reasons. Split it so that each module changes for one reason.
   - **Repeated switches**: the same `switch` or `if` cascade on the same type recurring across the change. Replace it with polymorphism or one shared map.
   - **Feature envy**: a method that reaches into another object's data more than its own. Move it onto the data it uses.
   - **Refused bequest**: a subclass or implementer that ignores or overrides most of what it inherits. Drop the inheritance for composition.
9. **Let the scope decide what is reviewed, not whether a line is new.** For a diff scope, focus on the lines in the hunks and read the pre-existing lines around them as context for the change: a convention the change breaks shows against them, and a bug they hold is reported (step 12), but their style is not re-reviewed on its own. For a named file or directory, every line is in scope and an old deviation is a finding like any other: the "do not retrofit" clauses of a project's rules do not apply when the user asked for the whole file to be brought up to the instructions.
10. **Give a test-only, refactor, cleanup or doc-only change the same reading as a feature.** A test that asserts the wrong thing gives false confidence, a refactor changes behavior through ordering or error paths, a cleanup removes code that was load-bearing, and a stale example in a doc is copied into new code. Read for what the change could break, not for what its label says it does.
11. **Label each finding `Trivial` or `Complex`, by whether it is trivially fixable.** A trivial fix is right at a glance, mechanical and reversible in a single hunk: the wording of a comment or a help text, a missing period, formatting or symbol ordering, an un-export or a rename whose every site a reference search confirms. Anything that changes behavior or an API, needs a decision between alternatives or is hard to undo is not trivial. When unsure whether something is a deviation at all, mark it not trivial and say why. The project's flow decides what happens to each kind, e.g. applying the trivial ones at once and proposing the rest. A project's own definition of trivial replaces this one.
12. **Report pre-existing logic bugs and broken functional conventions in the surrounding code**, whatever the scope form, and label them `Outstanding`. They are not the change's to fix, but a reviewer who read the code and stayed silent about them did not report what was observed.
13. **Check every occurrence.** A rule found broken once is checked across the whole scope, and each site is reported, so the caller fixes all of them and not one. When one rule is broken at many sites, one entry may cover them, as long as it names every site and gives the replacement for each trivial one.

## Output format

Open with one line stating the scope as read: each repository, the diff form or the files, the count of files read, and the task when the brief named one. A misread scope shows there before any finding does.

**Group the findings by severity, most severe first**: a `Critical`, an `Important` and a `Suggestion` section, in that order, each left out when empty. The confidence does not set the order. Any other sectioning is the reviewer's judgement, and a project's instructions or another skill may prescribe a layout or add a section, such as a verdict. Number every entry in one flat sequence across the whole report (1, 2, 3 and so on, continuing from section to section) so a reply of "apply 4 and 7" is unambiguous. A report with no findings says `None.`. The report lists problems only: no praise, no note on what was done well.

A finding states what was read, not what the code "might" or "probably" does. Where the evidence is incomplete, say what was checked and what remains unverified, so the caller knows what to confirm. A bug entry names the input or event that triggers it and what then happens, so the caller can confirm it without re-deriving it.

### Severity

**The tier is the severity of the finding: what it costs if it stays**, not whether an instruction file states a rule it breaks. Rules differ in what they protect: a broken rule on an invariant can be `Critical`, while a broken rule on the wording of a comment is a `Suggestion`. A project's instructions may set the tier of a rule they state.

- `Critical`: a bug, lost data, a security hole, or a broken invariant or functional convention.
- `Important`: a problem that breaks no behavior yet but makes the code harder to keep correct, e.g. a changed behavior that no test covers.
- `Suggestion`: a cost to readability or structure only: style, naming, code organization, a design smell (step 8), a recommendation among alternatives, or the reviewer's own judgement.

**Keep every `Suggestion` when there are few. When there are many, e.g. more than 20, keep the ones that matter most and drop the rest**, and give the count dropped under `Scope notes`: a long list of minor points hides the findings that matter. Never drop a `Critical` or an `Important` finding.

### Each finding

Each finding gives the file and line, the rule it breaks when one does (the instruction file and the section title, or the task's line), and what is wrong in one sentence. Then, by its label:

- `Trivial`: the **exact replacement**, the current text and the new text, both verbatim, so the caller can apply it without judgement.
- `Complex`: the alternatives with a recommendation, written so the finding can be pasted into a proposal list as it is.

A finding in the pre-existing code around the change also carries the label `Outstanding` (step 12). It covers logic and functional conventions only, not style.

**Each finding ends with its tier, its labels and its confidence**, as `[Critical, Complex, 95]`. The confidence, 0 to 100, says how sure the reviewer is that the finding is real, given what it read. A low number comes with the reason, as step 11 asks.

### Declined

A low-severity section after the findings: every observation the reviewer considered and set aside, one line each with the reason: a construct that looked like a deviation until a project rule endorsed it, a smell suppressed by a neighbor's convention, a gap the task leaves open on purpose. The caller rules on each line. Nothing is dropped silently.

### Scope notes

Anything about the scope itself: a file in the brief that does not exist, a repository with no changes, an instruction file the brief named that could not be found, a brief that omitted the focus, the description or the diff form, a task that is missing or defective. Then the instruction files that were read in full, so the caller can see what the scope was judged against: only files read in this session, never ones considered and skipped.

Write every entry as `Agent replies` in [`writing-style`](../writing-style/SKILL.md) prescribes: the line as it reads, not a paraphrase; the count, not "several"; what was observed, not what is impossible.

## Anti-patterns

Do not:

- Review from memory of what the instructions say. Load the files, or reload the leading one when reviewing in the main context.
- Review with the scope or the context missing. Ask for it first.
- Set a tier by whether a rule names the finding rather than by what the finding costs, e.g. a broken style rule above `Suggestion`.
- Report every minor suggestion when there are many of them.
- Narrow the given scope, or spill past it into files the brief did not name.
- Read the hunks that look interesting and skip the rest of the diff.
- Re-review the style of pre-existing lines around a hunk when the scope is a diff, or leave an old deviation alone when the scope is a named file.
- Drop a pre-existing bug because it is not the change's to fix. It is a finding, labeled `Outstanding`.
- Report one occurrence of a deviation and leave the others in the scope unchecked.
- Manufacture a finding so that the report is not empty. A clean scope produces a report that says `None.`.
- Praise what was done well. The report lists problems.
- Edit anything, or run a command that changes the working tree, the index or the build state.


## Improving the review

A review's misses and the findings the user declines are the evidence for improving this skill, a project's review agents and the instruction files they judge against. When the session that held the review, or received its report, goes on to fix the defects, the defects the user fixed that the report did not list and the findings the user declined as wrong are each a lesson. That analysis runs when the user asks to update the instructions, through [`update-instructions`](../update-instructions/SKILL.md), which names the review as a survey source. It is not part of the review itself, and the reviewer does not run it unprompted.

## Reference

This skill owns the reading: what a review loads, what it reads, what counts as a finding and how the report is shaped. The project's flow owns the rest: when a review runs, what the brief contains, who applies which findings and what is verified afterwards. Its entries follow the [`writing-style`](../writing-style/SKILL.md) skill, and the rules it checks against live in the project's instruction files, which it never edits.
