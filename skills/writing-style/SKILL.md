---
name: writing-style
description: 'Write English prose in code comments, docstrings, help text, commit message bodies, docs, agent instruction files and replies. Use before writing or editing any of these, or when asked to "check the writing style", "review the wording" or similar.'
---

# Writing style

How English is written in everything that is read rather than executed: code comments and docstrings (JSDoc, Python docstrings, Go doc comments and the like), CLI help text and setting descriptions, commit message bodies, user-facing docs, agent instruction files (`AGENTS.md`, `CLAUDE.md`, skills, knowledge files) and the agent's replies to the user. Read it before writing or editing any of these.

This skill governs sentences and word choice. Where a kind of text goes and what shape it takes (the form of a docstring, the format of a commit subject, the layout of a help screen) is owned by the project's own instructions. When a project states a rule that contradicts this skill, the project wins.

**Nothing here applies inside a fenced block or quoted output.** Code, log lines, command output and error messages stay verbatim, down to their punctuation, so the reader can grep for them.

**Apply these rules to text you write and to text you rewrite.** A sentence you touch for another reason is brought into line as part of that edit. Text you do not touch stays as it is: do not reflow a file to satisfy this skill. The one exception is the em dash rule below, which forbids new ones outright and leaves old ones alone.

## When this skill applies

Trigger this skill whenever you are about to write or edit any of the kinds of text named above. Also trigger it when the user asks to:

- "check the writing style" / "review the wording"
- "does this read well?" / "fix the prose"
- or any equivalent phrasing about how a text is worded rather than what it says.

Do **not** trigger this skill for the structural side of a text (where a docstring goes, which docstring form to use, how a commit subject is scoped). That is project-specific and lives in the project's instructions.

## Principles

- **Write for every reader.** Prose attached to code (a comment, a docstring, a description) documents the code for human and agent alike. Never word it as an instruction to the agent (e.g. "ask the user before changing this"). State the fact about the code and let the reader draw their own conclusion. Agent-directed guidance belongs in an instruction file instead.
- **Describe behavior, not code.** Say what something does or holds in proper English, in terms of the project's own concepts, not by listing the code that implements it or the symbols it touches. Use project-specific terms freely. Put a symbol name in backticks only when it adds clarity.
- **Derive language and terminology from the surrounding code.** Names, comments and messages reuse the vocabulary already established in the file, the module and the codebase: the existing term for a concept, never a synonym for it. Before naming something or describing it, check how the neighboring code and related symbols already refer to it, and match that. Similar things are expressed in similar language, so that identical concepts read identically everywhere.
- **One term per concept.** Pick one name for a thing and repeat it. Do not rotate synonyms for variety: a reader who meets a second name for the same thing has to stop and check whether it is the same thing.
- **Project terms are technical names.** A term the code or the tooling defines is used exactly as the code spells it, never simplified or paraphrased. When the reader may not know it, define it in one clause on first use, then use it bare.
- **Be concrete.** Prefer the specific to the general: the error code over "an error", the task name over "the build", the measured count over "often". "10 of 15 bodies wrap at 72" can be checked and acted on. "Bodies are often long" cannot.
- **State what was observed, not what is impossible.** Exhausting your own attempts does not show that a framework cannot express something. Write what was tried and what happened ("not achievable with the four groupings tried") so the text survives someone finding the fifth. An impossibility recorded this way in one session was disproved in the next turn by a shape that differed only in branch order.
- **Public content is self-contained.** Text that is published (a public repository, a package's docs) never mentions a private repository or anything inside it: no names, paths, commands, symbols or code samples. Describe a cross-project convention generically and take its examples from public code.

## Sentences and words

- **One idea per sentence. Split a sentence carrying three or more facts.** A long sentence that stacks a rule, its exception and its reason is three sentences, or a bullet per fact. Short active sentences are read once. A sentence that has to be re-read to find its subject is rewritten.
- **State the fact. Do not announce it.** Do not open a sentence with "Critically,", "Notably,", "Importantly," or "Essentially,". If a fact matters most, put it first.
- **State what holds rather than what is absent.** A positive statement hands the reader the fact itself: the label "Only in r5" says where a package is, while "Not in r4" leaves the reader to work that out. The rule covers names too, such as a field that records the one revision a package is in rather than the one it is missing from.
- **No inflated vocabulary.** Write `use`, not `leverage` or `utilize`. Drop `robust`, `seamless`, `comprehensive`, `delve`, `holistic`, `pivotal` and `crucial` unless quoting.
- **Do not hedge.** `Perhaps`, `possibly`, `potentially`, `might` and `probably` go only where the uncertainty is real, and then say what it depends on.
- **No dramatic framing.** Not "killed the run" or "pure collateral": state what the output shows and what it means. No metaphor or simile where the plain term exists. A project's fixed terms for failure modes (a list that _rots_, two copies that _drift_, a check that passes _silently_) are technical names under `One term per concept`, not drama.
- **Write words out.** `do not`, `is not`, `cannot` (one word), not `don't`, `isn't`, `can't`. A contraction stays only inside a quoted phrasing (`"don't close this as transient"`) or a `//` comment, where the register is looser.

## Examples in prose

- **Give at most one example.** When illustrating a category, pick the single most meaningful instance instead of listing several. Interchangeable names add no information beyond the first one, and each extra name is one more thing that can rot.
- **Mark it as an example** with `e.g.`, `for example`, or `such as`, so the reader treats it as one instance out of many rather than a closed set.
- **Never enumerate symbols, filenames, packages or tasks to gesture at a category.** Such lists go stale silently: nothing fails when a symbol is renamed or a module added, so they quietly become wrong and misleading. Instead state the rule that generates the list ("modules are named after their topic") and point at the authoritative source: the directory to list, the exports map, the file itself.
- **Exception: normative lists.** A list is not an example when it _is_ the content: a fixed vocabulary, an exhaustive catalog of the available helpers, a routing table, or a set where every entry teaches something the others do not. Keep those complete.
- **Deciding which kind a list is.** If the items are things _named in the code_ (symbols, filenames, modules, services, subsystems, constants), the list rots silently. Cut it. If they are plain-English descriptions of _behavior_ ("opening, refreshing, and discarding reviews"), or a set fixed outside the project (git's rebase instructions, JSON's value types), it is not an example. Leave it complete.

When you come across a stale enumeration while working on something else, replace it with the rule plus one example rather than extending it.

## Spelling, punctuation and typography

- **American spelling**: `behavior`, `color`, `capitalize`, `neighboring`.
- **Product and tool names as the project already spells them.** Check the existing text before choosing between variants such as `VSCode` and `VS Code`, and use the one that dominates. A name that is lowercase by convention stays lowercase mid-sentence. Do not open a docstring summary with one when the project's formatter capitalizes the first word.
- **Backticks** mark anything the reader could type, grep or click: an identifier, a file path, a task name, a command, a flag, an error code, a config key, and a section title when referring to a section (`Examples in prose`).
- **Double quotes** mark literal text a user sees (a message, a UI label, a line of command output) and a word used as a word.
- **Never write an em dash (`—`, U+2014) or an en dash (`–`, U+2013).** This is a hard rule with no exceptions for new text, in every kind of prose this skill covers. Set off an aside with commas or parentheses, introduce a definition or an explanation with a colon, or write two sentences. A spaced hyphen (`-`) is acceptable where a project has made a dash its established notation, such as a `term - definition` list format. Existing em dashes in a project are left as they are unless its instructions say otherwise, and their presence in a file is not a precedent for adding more. Before finishing a change, check your own additions: `git diff --word-diff=porcelain | grep '^+' | grep -v '^+++' | grep -c '—'` must print `0` in every repository you touched. The line-based `git diff | grep -c '^+.*—'` also counts a dash that was already on a line you changed, such as a Markdown paragraph you appended to.
- **No semicolons joining clauses.** Two independent clauses are two sentences. A semicolon stays only as a list separator inside a sentence whose items themselves contain commas.
- **Arrows are notation, not prose.** `→` and `->` are fine inside a table, a routing list (`Node-only → the Node package`) or an ordering (`types → variables → functions`). Do not chain them to narrate a sequence of events in a sentence. A rename in a commit subject is `rename a to b`, not `a => b`.
- **Bold the rule and the operative word. Italicize a contrast.** The sentence that states a rule, usually the first of a bullet, is bold. Inside a sentence, bold the one word a skimming reader must not miss (`do **not** use the inline form`). Italics mark a term contrasted with another (`what it _is_ versus what it does`). Bold is never decoration: one operative word per sentence at most.
- **Headings are sentence-cased**, with no trailing period.
- **A complete sentence is capitalized and ends with a period**, whether it stands alone or in a bullet. A fragment (a term followed by a colon and its definition, a one-line option description) is capitalized but ends without one. Do not mix the two forms within one list.

## Comments and docstrings

The shape of a docstring (one-line or multi-line form, summary and body, which tags it carries) is the project's rule. This section is about the words.

- **A function summary says what the function does, in the imperative**: "Run the callback with its stdout replaced by the given stream.", not "Runs …" or "This function runs …". Not a noun phrase naming the result either ("Target of a run …", "The `kill` command: …"), which reads as a value's summary. A command handler's summary is "Run the `<cmd>` command: …". A value, type or property gets a noun phrase naming what it holds or means: "Options for …", "Whether …", "Name of …".
- **Every summary ends with a period**, even a noun phrase: the summary reads as a sentence. Existing summaries without one are not retrofitted on their own. Add the period only when editing that comment as part of another change.
- **A body elaborates on rationale, edge cases and references** in the same register: full sentences, project terms, symbols only where they add clarity.
- **A comment that justifies a design choice says what the simpler alternative would break.** State the constraint, then the concrete failure of the obvious design, then the choice. "With module variables, the copy that sets its formatter last would map only its own files" can be checked against the code. "Because every bundle has its own copy" leaves the reader to work out why that matters. When an instruction file already documents the constraint, the comment states the rule as it applies at that spot, in one or two sentences, and names the section that documents it. It does not repeat the mechanism, and it does not say that the code departs from a rule: the reader needs what to do here and the one-clause reason, and the mechanism would drift from the section that owns it.
- **Reference code by symbol, never by line number.** A `file.ts:162` in a comment or a doc rots the moment the file moves. Name the function, method or property, with the documentation system's link form where the project allows it. Line anchors belong only in the agent's own output (see `Agent replies`).
- **Name a test or demo by its file, not by the command that runs it**: "run `src/test/logging-test.ts`", not the full command line. How to run each kind of test belongs in one place in the project's instructions, and a copy of the command in a comment drifts from it.
- **A `//` comment explains what the code alone does not show**: the reason for a step, a non-obvious consequence, an upstream reference. It is a fragment: lowercase start, no trailing period. When it runs to several sentences, write them as full sentences.

## File-level comments

Applies to a comment that describes a whole file or module (a `@file` JSDoc, a Python module docstring, a Go package comment).

- **The summary line says what the file is for**, self-contained enough to stand alone in a file map or an index.
- **The body describes the functionality the file exposes, in proper English**, not literal symbol names or implementation details. Use project-specific terms freely.
- **Do not enumerate the file's parts**: the subsystems it initializes, the constants it defines, the services it wraps. Name the area of functionality instead, and at most one concrete instance (see `Examples in prose` above). A file summary is often copied into a file map, so an enumeration here goes stale in two places at once.
- **The goal is human-readable prose, not an export listing.** A reader should learn what the file is about without recognizing any of its symbols.

## Help text and descriptions

Applies to CLI help (a command's brief and description, an option's or argument's description) and to the description of a configuration property.

- **Describe the effect for the user, not the implementation.** "Modify the input file in place" says what happens. Which function performs it does not belong here.
- **A command's brief is a minimal imperative fragment** ("Run a test session") with no trailing period.
- **Option and command descriptions are consistent within one CLI: either every one ends with a period or none does.** A description that is a proper sentence or several sentences ends with a period. So when a CLI has even one such description, every description in it, one-line fragments included, ends with a period. Only a CLI whose descriptions are all one-line fragments leaves the period off.
- **Every promise in a description is a test.** "Describe it", "exit with the binary's exit code", "without wrapping long lines" each name a behavior. Before finishing, find the line of code that delivers each one. A description that cannot be matched to code is either wrong or describing a bug.
- **Name the literal the user types the way the framework renders it.** When the CLI framework has its own way to mark a flag, a value or a command name in a description (a template interpolation, a markup tag), use it rather than typing quotes into the text, so the rendered help styles the literal once and consistently.
- **Do not state a configuration property's default value in its description** when the schema or the framework already records it. A sentence such as "Off by default." only repeats the schema and drifts from it when the default changes.

## Commit message bodies

The format of a commit message (subject shape, scope prefix, wrapping) is the project's rule. Within that format, the body is the principles above applied: describe the nature of the change in proper English rather than listing the code that changed, use project terms freely, put a symbol in backticks only when it adds clarity, and make every bullet a full sentence. A commit message names a function or a test, never a `file:line`, and carries no link.

## Instruction files

Applies to every agent instruction file: `AGENTS.md`, `CLAUDE.md`, skills, knowledge files written for agents.

- **Address the agent directly, in the imperative**: "Read …", "Do not guess". The human is "the user". The AI is "the agent".
- **Lead with the rule, then the reason.** The first sentence of a bullet states what to do, in bold. What follows explains only as much as the agent's next action depends on. An instruction earns its space by prescribing. An explanation of the underlying mechanism belongs in the file only when knowing it changes what the agent does.
- **State the rule, not the instance.** A renamed symbol teaches the naming convention it reveals, not "rename X to Y". An inlined helper teaches "inline short, single-use helpers", not "inline this one function". Write the principle a future reader applies, and leave the one-off fix in the commit.
- **Do not stamp a fact with the dependency version it was checked against** (e.g. "Verified against version 1.3.6."). The stamp goes stale with the next upgrade and does not change what the reader does. Name a version only where it tells the reader when to act, such as the release that fixes an upstream bug a rule works around.
- **Name the failure the rule prevents** (the error code, the misleading command output, the task that fails) so a reader can recognize the situation and later confirm the rule is still real. Leave out the story of how it was discovered. That belongs in the proposal and the commit message.
- **Name only what exists.** Every symbol, task, flag, file and error code mentioned must exist in the current tree. A rule that cites something stale is worse than no rule. Check before writing it down.
- **Refer to a section by its title in backticks and link the file**: "see `Examples in prose` in [writing-style.md](writing-style.md)". A heading quoted this way survives the section moving within the file. A line number does not, so a git-tracked file never carries one (see `Reference code by symbol` above).

## User-facing documentation

Applies to a `README.md`, user guides and changelogs: anything written for the person using the tool rather than for the person maintaining it.

- **Address the end user and describe what the tool does for them**, in terms of their tasks. Implementation details and agent-directed guidance stay out. The latter belongs in an instruction file.
- **Published as-is.** Public docs must read as self-contained (see `Public content is self-contained` above).

## Agent replies

The agent's messages to the user, and the temporary documents it writes for one reading (a plan, a design, a review report), follow the same principles, above all `Be concrete` and `State what was observed`: the error line as printed rather than a paraphrase of it, the count rather than "several", what was tried and what happened rather than "impossible".

**This is the one place a line number belongs.** A reply or a plan is read once, against the tree as it is now, so a link to a file and line is current by construction and lets the user jump to the code. Form it as a Markdown link with a `#L<line>` anchor (`[renderLog](src/cmd/render.ts#L373)`), with the path relative to the repository root. Only that anchor form is clickable in most hosts. A `:<line>` suffix renders as plain text. Keep square brackets out of the link text, since the text ends at the first `]`. Everything git-tracked (code, comments, instruction files, docs) names the symbol instead.
