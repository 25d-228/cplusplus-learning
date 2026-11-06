# Repository working rules

This is a project-based technical book and its learner workspace.

- `main` holds the concise course README/index and this guidance.
- `gh-pages` holds the canonical book and generated website, not learner source.
- Implement the course on a persistent `work/strata` branch from `main`.
  Keep related work together; no branch-per-function ceremony is needed.

Read the relevant complete chapter and prerequisite code before implementing.
Explain, review, and diagnose by default. Implement only the scope explicitly
delegated by the user. Passing a check or publishing a lesson does not itself
permit closing its issue; ask before closing a whole issue. Do not add an approval
ceremony between functions when the implementation scope is already delegated.

Preserve user edits and existing branch history. Do not merge gh-pages into code
branches to acquire documentation. Read the site, issues, or fetch the book source
separately. Documentation updates on main may be brought into code branches with
a normal merge only after inspecting local changes; never reset the learner's work.

The textbook, not private chat context, defines the lesson. Point out missing
requirements rather than invent them silently. Do not publish private notes,
credentials, logs with secrets, or the authoring package itself.

Strata's index, disk format, recovery, compaction, spill processing, and bounded
runtime are learner-implemented mechanisms. Do not replace them with a database
or executor dependency. Preserve explicit ownership, resource limits, and failure
semantics when suggesting changes. A correctness check is not a performance
result; a process-stop test is not a physical power-loss test.
