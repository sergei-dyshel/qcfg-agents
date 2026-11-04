---
name: learn-from-corrections
description: 'Extract durable lessons from the user''s corrections to agent-generated code and propose them as instruction edits. Use when asked "what did you learn from my corrections?", "I fixed your code, what can you learn?" or similar.'
---

# Learn from the user's corrections

When the user edits code the agent just generated (rewording a comment, renaming a symbol, inlining a helper, reworking a control-flow branch, or leaving a `REVIEW:` note on it), each change encodes a preference the agent got wrong. This skill is the procedure for reading those corrections and distilling the general lesson behind each one.

**This skill only extracts and proposes lessons. It never applies them.** Exactly like [`update-instructions`](../update-instructions/SKILL.md), recording knowledge is a two-step process: the agent proposes, the user approves the specific wording, and only then is anything written. Being asked what you learned is a request for the answer, not permission to write it.

**The project's own instructions refine this procedure.** A project may say where its instruction files live or how to read a diff in its repository layout. Read them first, and where they contradict this skill, the project wins.

## Relationship to `update-instructions`

Both skills learn from the session and record what they learn through the **same** procedure. They differ only in **what they learn from**:

- **This skill** learns from the user's corrections to the agent's code: the diff between what the agent wrote and what the user changed it to, plus any `REVIEW:` notes left on that code.
- [`update-instructions`](../update-instructions/SKILL.md) learns from the session as a whole: every user prompt and the agent's development experience, such as build, lint and test failures, misleading command output and unwritten exceptions to a rule.

The shared downstream half (routing a candidate lesson to a file, wording the proposal, presenting it, and applying it once approved) lives in one place, the [`proposing-instruction-edits`](../proposing-instruction-edits/SKILL.md) skill, which both skills reference so they cannot drift. This skill owns only the correction-specific **extraction** steps below.

## When this skill applies

Trigger this skill whenever the user asks:

- "what lessons can you learn from my corrections?"
- "I made corrections to your changes, what can you learn?"
- "which instructions can you derive from my edits?"
- or any equivalent phrasing that points at edits the user made to code the agent generated (including `REVIEW:` comments they left on it) and asks for the takeaways.

Do **not** trigger this skill for:

- **Addressing `REVIEW:` comments.** Executing the pending instructions in `REVIEW:` comments is the [`review-comments`](../review-comments/SKILL.md) skill. This skill instead _learns_ from them, treating each as evidence of a preference, and is usually invoked after they have been addressed, or on corrections that carry no `REVIEW:` at all.
- **A general "anything you learned?" question** not anchored to the user's corrections. That is [`update-instructions`](../update-instructions/SKILL.md), which surveys the whole session, not just a diff.
- **A specific instruction edit the user dictated.** If the user says what to write and where, just write it.

## What this skill does

At a high level, this skill:

1. Reads the exact corrections: the diff between what the agent wrote and what the user changed it to, plus any `REVIEW:` comments the user left on the generated code.
2. Classifies each correction as an **applied** edit (the user changed the code) or a **pending** `REVIEW:` note (an instruction not yet executed). Both reveal a preference. Only applied edits show the preferred end state.
3. Distills the **general lesson** behind each correction: the reusable principle, not the one-off fix.
4. Filters hard, keeping only lessons that are durable, general and not already recorded.
5. Hands the survivors to [`proposing-instruction-edits`](../proposing-instruction-edits/SKILL.md) (route, word, present and wait, apply on approval) and follows it to the letter.

## Procedure

### 1. Read the corrections precisely

**Read the actual diff. Do not reconstruct the corrections from memory of what was written.** The generated code lives in dirty (uncommitted) working-tree changes. Run `git` from the repository that holds the files, which may be a nested repository such as a submodule rather than the one at the root:

```sh
git --no-pager diff -- <files the agent generated>
```

Read both halves of every hunk. The **removed** lines are what the agent wrote. The **added** lines are what the user preferred. A correction that only reworded a comment is as informative as one that changed logic.

Also collect the `REVIEW:` comments the user left on the generated code (the same `git diff` shows them as added lines). These are pending instructions, not applied edits, but each one names a preference just as clearly.

### 2. Classify each correction

Separate the corrections into two kinds, because they carry different weight:

- **Applied edit**: the user changed the code. This shows both the preference _and_ its concrete end state, so the lesson can be stated precisely and verified against the result.
- **Pending `REVIEW:` note**: an instruction the user left but did not execute (e.g. "inline this function, it is short and used only once"). It reveals the preference but not a worked example. When the same `REVIEW:` preference recurs across several spots (the user tagged six single-use helpers with the same note), the repetition is itself the signal that the underlying rule is worth recording, not just the individual sites.

### 3. Distill the general lesson

For each correction, ask what reusable principle it teaches: the rule a future agent would follow to avoid the same correction. State the rule, not the instance:

- A renamed symbol → the naming convention it reveals (matching surrounding vocabulary), not "rename X to Y".
- An inlined helper → the structural rule (inline short, single-use helpers), not "inline this one function".
- A reworded comment or option description → the phrasing convention, not the specific sentence.
- A control-flow rework → the invariant or ordering rule behind it.

A correction whose lesson is purely local (a fixed typo, a value that was simply wrong for _this_ command) has no general rule to teach. List it as considered and rejected when presenting the proposals, rather than forcing a rule out of it.

### 4. Filter hard

Most corrections do not become instructions. Discard a candidate lesson if any of these hold:

- **The repository already records it.** Re-read the relevant instruction files and knowledge files before claiming a lesson is undocumented. A correction often just re-applies an existing rule the agent overlooked. When something is already covered, the fix is to _follow_ the existing rule, not to add a second one.
- **The toolchain already catches it.** A correction that fixes something the build, the linter or a test would have flagged needs no rule.
- **It is task-specific.** A domain fact true only of this one command or feature (e.g. which host a particular process runs on) is not a general instruction. It may still belong in that code's doc comment, which is ordinary documentation work, not an instruction proposal.
- **It is an enumeration.** State the rule and point at the authoritative source, per `Examples in prose` in [`writing-style`](../writing-style/SKILL.md), rather than listing the symbols the correction happened to touch.

### 5. Route, propose and stop

The surviving lessons are now ordinary candidate instruction edits. Hand them to [`proposing-instruction-edits`](../proposing-instruction-edits/SKILL.md) and follow it to the letter. Do not duplicate its mechanics here. Two of its requirements matter most for corrections:

- **Each proposal references its destination instruction file and, where one exists, a link to the code location of the correction that led to it**: the source file and line the user changed, or the `REVIEW:` comment they left, so the evidence behind the rule is one click away.
- **Proposals are presented as numbered, clearly readable paragraphs, never a table**, so the user can approve, reject or reword each by number.

Then **stop**. Propose only, and apply nothing until the user approves the specific wording.

If the user's reply is itself a correction to a proposed lesson (rewording it, narrowing its scope, rejecting it as too specific), that reply is a fresh correction. Treat it the same way: it refines what the recorded rule should say.

## Anti-patterns

Do not:

- Turn a one-off, task-specific correction into a general rule. If it teaches nothing reusable, list it as considered and rejected.
- Record a lesson the repository already documents. Follow the existing rule instead of adding a duplicate.
- Propose a rule for a correction that the build, the linter or the test suite already catches.
- Reconstruct the corrections from memory instead of reading the actual diff. The removed lines are the lesson.
- Apply any instruction edit because the user asked what you learned. Proposal and approval are two separate steps.
- Present proposals as a table, or omit the link to the code correction that motivates each one.
- Duplicate the routing, proposal and apply mechanics here. They live in [`proposing-instruction-edits`](../proposing-instruction-edits/SKILL.md) so the skills cannot drift.
- Write the lessons to any out-of-repo memory store.
