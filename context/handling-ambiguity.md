# Handling ambiguity and uncertainty

- **Do not guess.** If the prompt or request is vague, lacks context, or leaves critical implementation details open to interpretation, do not assume or guess intent.
- **Ask clarifying questions.** Stop and ask targeted, concise questions to clarify requirements, edge cases, or environmental constraints **before** generating any code or edits.
- **Offer architectural options.** If a problem has multiple valid solutions or design patterns, briefly outline the top 2–3 approaches and ask which to use instead of defaulting to one arbitrarily.
- **Flag conflicts.** If a requested change conflicts with existing architectural patterns in the codebase (or in the currently-open files), point out the discrepancy and ask for confirmation before writing code.

