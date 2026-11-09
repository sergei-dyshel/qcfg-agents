---
name: update-instructions
description: 'Turn knowledge gained during a session into proposed edits to in-repo instruction files. Use when asked "anything you learned?", "any new knowledge?", "anything we can add to instructions?", "update instructions" or similar.'
---

# Update instructions from session experience

Everything an agent learns that outlives the conversation (a build quirk, an unwritten exception to a rule, a command whose output misleads) belongs in an instruction file tracked in the repository, so it is reviewable, versioned, and visible to other contributors and agents. Out-of-repo memory is invisible to all of them.

Recording that knowledge is always a **two-step** process: the agent proposes, the user decides. This skill is the procedure for finding what to propose. The [`proposing-instruction-edits`](../proposing-instruction-edits/SKILL.md) skill owns how a proposal is worded, presented and, once approved, applied.

**The project's own instructions refine this procedure.** A project may name its mandatory build task, its knowledge files or a repository layout that changes how `git` reports changes. Read them first, and where they contradict this skill, the project wins.

## When this skill applies

Trigger this skill whenever the user asks:

- "anything you learned in this session that can be added to instructions?"
- "any new knowledge?" / "anything we can add?"
- "update the instructions" / "document what you learned"
- or any equivalent phrasing about capturing session experience.

Also trigger it **unprompted** when a session produced a discovery that would clearly mislead the next agent if left unrecorded, but stop at proposing, exactly as if the user had asked.

Do **not** trigger this skill for:

- **A specific instruction edit the user dictated.** If the user says what to write and where, just write it. There is nothing to survey or propose.
- **Ordinary documentation work**: a doc comment, a file-level summary, a `README` section, or a mechanical map that mirrors the code (a file index, for example), describing what code does rather than recording a rule for using it. Those are part of the coding task itself. A prose knowledge file is not: it states rules rather than mirroring structure, so an insight belonging there goes through the proposal flow like any `AGENTS.md` edit, even when the file itself invites the addition.
- **Recording work items.** Follow-up work discovered mid-task becomes a comment in the relevant source spot, in the form the project uses for such comments, not an instruction edit.
- **Learning from the user's corrections to generated code.** That is [`learn-from-corrections`](../learn-from-corrections/SKILL.md), which reads the diff between what the agent wrote and what the user changed it to.

## What this skill does

At a high level, this skill:

1. Surveys the session for knowledge that is **durable** (true beyond this conversation) and **not already recorded** anywhere in the repository.
2. Filters that list hard. Most of what feels like a lesson is either already written down, already caught by the build, or specific to the conversation.
3. Verifies each survivor against the current tree.
4. Hands the survivors to [`proposing-instruction-edits`](../proposing-instruction-edits/SKILL.md): route to a file, word the proposal, present the numbered list and **wait**, apply only what the user approves.

Steps 1 to 3 are unique to session-wide learning and are detailed below. Step 4 lives in the shared skill.

## Procedure

### 1. Survey the session

Look back over the session for moments where reality differed from expectation. The richest sources, in rough order of value:

- **A build, lint or test failure whose cause was not covered by any existing rule.** These are the most valuable entries because they cost real time and will cost it again.
- **A skill or doc that let you down while you followed it**: a documented command that failed or needed a different spelling, an argument or output field that was missing or named differently, a default that changed, a claim the run contradicted. Treat the friction as a defect in the doc, not only in the run, and quote the command that failed and the one that worked.
- **A rule that turned out to have an unwritten exception**, typically revealed by the user overruling an action the rule seemed to mandate. The exception is as important as the rule.
- **A rule or correction the user stated during the session** ("when X, check Y") that no instruction file records yet.
- **A command whose output was misleading** rather than wrong, so it was believed at first, or that needed interpretation no doc gives: what an empty result means, which errors are transient and worth a retry.
- **A fact that had to be verified empirically** because no doc stated it, and a future agent would have to verify it again.
- **A documented entry that was not enough**: you had to run the command's `--help`, read its source or try twice. Propose a fuller entry for that command: a worked example with real-shaped arguments, the meaning of the result fields you had to interpret, the follow-up its result usually leads to.
- **A recipe in a knowledge file that upstream has since superseded.** When the repository keeps a verbatim snapshot of an upstream doc beside its own knowledge file, reading the snapshot end to end is a survey source in its own right. It surfaces recipes that still work but are no longer the supported way, which no build failure will ever flag. Verify each divergence against the pinned version before proposing it, and propose the code migration together with the doc edit.
- **A conflict between two instruction files**, or between an instruction file and the code.

### 2. Filter out what does not belong

Discard a candidate if any of these hold:

- **The repository already records it.** Code structure, symbol names, past fixes and history are in the code and in git. An instruction file that restates them goes stale silently.
- **An existing rule already covers it**, even loosely. Re-read the relevant instruction file sections before claiming something is undocumented. Check the rule against the **specific failure**, not just the topic: a rule stating one half of a two-sided constraint leaves the other half undocumented while making the area look covered.
- **The toolchain already catches it.** When the project mandates a build for every change, a mistake that fails compiling, linting or a test needs no rule. A rule earns its place by covering something that passes the build but is still wrong, or that costs a debugging detour.
- **It is conversation-specific**: what was decided next, which files were touched, what the user wants done later.
- **It is an enumeration** of symbols, filenames, packages or tasks. State the rule that generates the list and point at the authoritative source instead. See `Examples in prose` in [`writing-style`](../writing-style/SKILL.md).
- **It weakens a safety rule of the file it would land in**: a rule that forbids or gates an action, such as a write, a destructive command or a step that needs the user's approval. Do not propose loosening it. Report the observation behind it to the user as a fact (e.g. a step that needed a write the rule forbids), and leave the decision to them.

### 3. Verify each survivor

Before proposing anything, confirm it still holds in the working tree: re-run the command, re-read the file, check that every symbol, flag, task name and error code named in the proposed wording exists today. A rule that cites something stale is worse than no rule. Re-read the destination section itself rather than the copy already in context. The user may have rewritten it earlier in the same session, so an anchor quoted from memory can no longer exist.

**A convention must be checked against every instance that already exists, not just the one in front of you.** When the proposed wording says what new code should do, check every call site that already does it. If they disagree with each other, the finding is an inconsistency and the proposal is a question for the user, not a rule. In one repository, the CLIs built on one framework turned out to use four different completion configurations, so a rule naming any one of them would have been wrong about most. When the proposal simplifies a shape rather than forbidding one, grep for every occurrence and check each against its consumers before proposing. The count tells the user whether this is a one-line note or a sweep, and a shape that looks redundant can be load-bearing at some sites.

**Some claims can only be confirmed by constructing the failure.** Enable a rule to count its hits, hoist a value into a variable to watch the inferred type widen, add a file that deliberately violates the thing. A measured number in the proposal is worth far more than a hedge. Treat the experiment as disposable: restore every file touched and confirm with `git status` that nothing leaked before presenting the list. A probe file left in the source tree is picked up by the next build. When the user sanctions a scratch experiment inside a tracked file, that file stays dirty by design: name it and its revert command in the final message instead of restoring it.

**When the probe is a whole-tree command rather than a file edit, run it on a throwaway copy outside the repository.** A command that rewrites shared state (a lockfile, an installed dependency tree, a build cache) has no targeted reverse edit to undo it. Copy the inputs it reads into a scratch directory under `/tmp` and probe there, with whatever flag keeps it from reaching further (e.g. `--package-lock-only` for an `npm install` probe). Confirm the real tree is unchanged afterward all the same.

**Undo a probe with a targeted reverse edit, never a whole-file snapshot.** The user may edit the same file while the probe is in flight, and restoring it by overwriting from a copy taken beforehand silently discards that edit. Comparing the restored file against your own snapshot cannot detect the loss, because both predate it. Reverse the exact text the probe changed, then confirm the concurrent changes are still present. The same holds for every tracked file, not only probes: never restore one with `git checkout` or `git restore`, and never delete and recreate it.

**Confirm the probe actually landed before trusting its result.** An edit that silently fails to apply leaves output indistinguishable from a genuine pass, and the wrong conclusion is the comfortable one: a green suite reads as "the new test does not catch the bug". Assert the change is present before running (grep for the symbol you deleted, or check the diff), and only then draw the conclusion.

### 4. Route, propose and apply

The surviving findings are now candidate instruction edits. Route, word, present and (on approval) apply them via [`proposing-instruction-edits`](../proposing-instruction-edits/SKILL.md). Do not restate its mechanics here. It covers choosing the destination file, preferring to amend an existing rule, writing each proposal so it can be pasted, presenting the ranked numbered list and stopping, and applying only what the user approves.

## Anti-patterns

These are specific to surveying a session. [`proposing-instruction-edits`](../proposing-instruction-edits/SKILL.md) lists the anti-patterns of the proposal and apply half.

Do not:

- Apply an instruction edit because the user asked what you learned. The question is not the permission.
- Propose a rule for a mistake that the build, the linter or the test suite already catches.
- Add a new bullet that restates or contradicts an existing rule instead of amending that rule in place.
- Propose wording that cites a symbol, task or error code without checking it exists in the current tree.
- Record the lesson in the component you were editing when it belongs to the component that owns the mechanism.
- Silently drop a candidate you decided against. List it with its reason.
- Write project knowledge to any out-of-repo memory store.
