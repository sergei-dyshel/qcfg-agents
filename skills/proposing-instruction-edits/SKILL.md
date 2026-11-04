---
name: proposing-instruction-edits
description: 'Route a candidate lesson to the right instruction file, word it, present it for approval and apply only what the user approves. Use from update-instructions or learn-from-corrections, or when asked to "propose an instruction edit" or similar.'
---

# Proposing instruction edits

This skill is the shared procedure for turning a **candidate lesson**, a durable and general fact worth recording, into an edit to an instruction file tracked in the repository. Two skills feed it:

- [`update-instructions`](../update-instructions/SKILL.md): lessons surveyed from the session as a whole (build, lint and test failures, misleading command output, unwritten exceptions, prompt history).
- [`learn-from-corrections`](../learn-from-corrections/SKILL.md): lessons extracted from the user's corrections to agent-generated code.

The two skills differ only in **how a candidate lesson is discovered**. Everything downstream of that (routing it to a file, wording it, presenting it, and applying it once approved) is identical and lives here, so the two cannot drift. Each skill references this one rather than restating the procedure.

Recording knowledge is always a **two-step** process: the agent proposes, the user decides. Never edit an instruction file (`AGENTS.md`, `CLAUDE.md`, a knowledge file written for agents, a skill) without explicit approval of the specific wording.

**The project's own instructions refine this procedure.** A project may name its instruction files, say which one owns which topic, or add steps for applying an edit (a formatter to run, symlinks to avoid). Read them before routing, and where they contradict this skill, the project wins.

## When this skill applies

Trigger this skill when:

- [`update-instructions`](../update-instructions/SKILL.md), [`learn-from-corrections`](../learn-from-corrections/SKILL.md) or another skill hands over candidate lessons
- the user asks to "propose an instruction edit" / "suggest where this rule should go"
- or any equivalent phrasing that asks for a lesson already identified to be routed, worded and proposed.

Do **not** trigger this skill for:

- **Finding the lessons.** Surveying a session is [`update-instructions`](../update-instructions/SKILL.md). Reading the user's corrections is [`learn-from-corrections`](../learn-from-corrections/SKILL.md).
- **A specific instruction edit the user dictated.** If the user says what to write and where, just write it.

## 1. Choose the destination

**A defect is not a rule.** When the surprise turns out to be a defect in tooling the project owns, for example a task that only runs on machines with a newer shell, the output is a fix to that tooling, not an entry telling the next agent to work around it. Propose the fix. Do not conclude "nothing to record" because the failure did not reproduce for the user: a defect only some environments hit is still a defect. Likewise, when the finding is that a documented recipe is outdated rather than wrong, the doc edit alone leaves every existing call site on the old idiom. Propose the migration of those call sites in the same list, and check whether the replacement API needs a wrapper first.

**The tree may have moved since the knowledge was gained.** A long-lived session's finding can point at a file that has since been renamed, split or deleted: the reference CLI was a single `src/cmd/x.ts` when the lesson surfaced and is now a `src/cmd/x/main.ts` folder. Re-locate the current owner of the mechanism before routing (grep for the symbol, not the old path). If the code the finding described no longer exists, the finding is stale. Drop it rather than proposing wording that cites a vanished file.

**A rewording is a lesson about style, not a cosmetic fix.** When the user rewrites prose the agent produced (a comment, a docstring, a help text or setting description, an instruction file entry, the wording of a proposal from this very procedure), read both halves of the diff and work out what changed: the term chosen, the mood or tense, the order of rule and reason, what was cut as unnecessary, what was moved elsewhere. Then state the preference as a general rule ("name the failure the rule prevents", not "this sentence should read …") and route it to the project's writing-style file. A one-word change is evidence. The same kind of rewording appearing twice in one session is a rule the guide is missing. Check the guide and the [`writing-style`](../writing-style/SKILL.md) skill first: a rewording that merely re-applies a rule already there is a reason to follow that rule, not to add a second one.

Route each candidate by what it is about:

- **Cross-cutting, applies to the whole repository** → the root instruction file (`AGENTS.md` or its equivalent).
- **How prose is worded** → the project's writing-style file, in the section for that kind of text. Only the words, sentence shape and punctuation go there. Where the text goes and what shape it takes stays with the file that owns the shape (e.g. the rule on docstring form). A project with no such file gets the rule in its root instruction file.
- **A constraint of an API the project owns** → the doc comment on that API, not a bullet in an instruction file. A doc comment is read at the point of use. A bullet is read only by an agent that thinks to look. Reserve the bullet for a fact with no single API to attach to. Both are warranted when they carry different content: the doc comment stating the constraint and what to do about it, the bullet adding the build-workflow fact of which task surfaces the violation. A doc comment is not an instruction file, so it falls outside the permission rule and is applied as part of the coding task rather than proposed. When the survey (or the correction) rather than the coding task surfaces the constraint (the change is already committed, or the API lives in code this task never touched), there is no coding task left to fold it into: list it with the other proposals and apply it on approval.
- **Owned by one component** (a package, a module, a subproject with its own instruction file) → that component's instruction file. Pick the component that **owns the mechanism**, not the one that happened to be edited when the lesson surfaced. Ownership follows the topic, not the implementation: a rule about a practice belongs to whichever component documents that practice, which may sit below the component implementing the helper the rule names. Search the existing knowledge files for one already covering the topic before proposing a new section.
- **A long procedure with its own trigger phrasings** → a new skill, with a short pointer from the relevant instruction file. When the lesson instead refines a procedure some skill already owns, amend that skill rather than adding a rule to an instruction file.
- **A knowledge file for a specific tool or framework** → a doc beside the component that owns the tool.

If the destination is a doc that no instruction file links to yet, the proposal must include adding that link. An unlinked doc is invisible.

**When the failure was that the agent never opened a doc the instructions already link, adding emphasis to the pointer does not fix it.** The next agent skips it the same way. Relocate the single load-bearing fact into the file the agent was already reading, and leave the rest behind the pointer. Copying more than that one fact makes the two files drift.

**A request that names one instruction file scopes the reply, not the routing.** Route every candidate to the file that owns it as usual, and present the ones landing elsewhere alongside the in-scope ones, each labelled with its destination. A candidate forced into the named file is misfiled, and one dropped for being out of scope is lost.

## 2. Prefer amending an existing rule over adding a new one

Most real discoveries are refinements: an exception, a caveat, a collision case. Attach them to the rule they qualify, in place. A fresh bullet that overlaps or quietly contradicts an existing one is worse than no bullet, because the next reader has to guess which wins.

## 3. Write each proposal so it can be pasted

Every proposal needs, in a clearly readable paragraph (never a table):

1. **The destination file, and the exact location**: a line number, or the exact existing sentence the new text attaches to.
2. **A link to the code correction or failure that motivates it, where one exists**: the source file and line the user changed (or the `REVIEW:` comment they left), so the reader can see the concrete evidence behind the rule. Omit it only when the lesson has no single code location (e.g. a session-wide build-workflow surprise).
3. **The exact wording**, ready to paste. Not a description of what to say. Present it as a Markdown **quote**, never a code block: a quote renders the wording the way it will read in the file, with its inline code, links and emphasis intact.
4. **One sentence on the concrete failure, surprise or correction that motivates it**, including the error code or command output where there was one. This is what lets the user judge whether the rule is worth its space, and lets a future reader confirm the rule is still real. When the surprise is a third-party bug rather than a project convention, the proposal also names the upstream issue, or says none was filed, so the entry can be retired once upstream fixes it. Keep that motivation in the proposal message rather than in the wording itself (see `Lead with the rule, then the reason` under `Instruction files` in [`writing-style`](../writing-style/SKILL.md)).

Word the proposal as the file it lands in is worded. The [`writing-style`](../writing-style/SKILL.md) skill applies in full, above all its `Instruction files` and `Examples in prose` sections, and so does the project's own writing-style file. Keep lines within the project's print width.

## 4. Present the list and stop

Present the proposals as clearly readable numbered paragraphs, **not a table**, ranked by value, highest first. Then also list the candidates that were **considered and rejected**, each with a one-line reason: the user may disagree with a filtering call, and cannot if the candidate was never mentioned. When the reason is that the repository already records it, name the file and the symbol or section that records it. "Captured in a code comment" is not a reason.

**One fact per candidate, numbered in one flat sequence**: proposals, rejected candidates and side notes alike. A candidate bundling several facts cannot be answered with one verdict, and a numbering that restarts under headings makes a reply of "2." ambiguous. Expect a reply that keeps some, drops others, rewrites a third and adds one of its own. When it answers with a question instead of a verdict, answer the question first. If a reply's referent is ambiguous, ask. Do not pick the reading that lets you keep the entry.

Then wait. Do not apply anything. Being asked what you learned is a request for the answer, not permission to write it.

## 5. Apply only what was approved

Once the user picks:

- **Apply exactly the approved items, with the approved wording.** Do not slip in a rejected candidate or a newly thought-of extra.
- **If the user removes or rewrites text you applied, that is a decision, not an oversight.** Do not restore it in a shorter form, in a different file or as a code comment, and do not re-propose it in a later round.
- **An item the reply passes over in silence stays open.** Note it once as still pending in the closing message and leave it there. Silence is neither approval nor rejection, so do not apply it and do not raise it again in a later round.
- **After a term substitution, re-read the whole sentence.** Replacing a command or name in place can leave the surrounding claim contradicting itself, and the diff looks clean because only the intended words changed. Report the collateral wording, and any occurrence the user's scope word excluded, rather than widening the edit unasked.
- **Edit the real file, never a symlink to it.** Instruction files are often reachable under several names (a `CLAUDE.md` linked to `AGENTS.md`, a skills directory linked into a host's config directory). Resolve the link and edit its target once. Never delete or replace a linked directory itself.
- **Run the project's formatter over the edited files**, as its instructions prescribe.

## Migrating pre-existing out-of-repo memory

If out-of-repo memory entries containing project knowledge are discovered, migrate their content into the appropriate instruction file via the proposal flow above, then delete the out-of-repo entries.

## Anti-patterns

Do not:

- Apply an instruction edit because the user asked what you learned. The question is not the permission.
- Propose a rule for a mistake that the build, the linter or the test suite already catches.
- Add a new bullet that restates or contradicts an existing rule instead of amending that rule in place.
- Propose wording that cites a symbol, task or error code without checking it exists in the current tree.
- Enumerate symbols or filenames to gesture at a category.
- Wave off the user's rewording of a comment, docstring or instruction as cosmetic instead of inferring the style rule behind it.
- Record the lesson in the component you were editing when it belongs to the component that owns the mechanism.
- Present proposals as a table instead of numbered, readable paragraphs.
- Silently drop a candidate you decided against. List it with its reason.
- Write project knowledge to any out-of-repo memory store.
