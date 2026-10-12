- When naming, prefer verbose and clear over concise and unclear
- When naming, think about how you would want to talk about the code and the feature to stakeholders
- When naming, reuse concepts where it makes sense, don't invent something new that would make names inconsistent
- my goal is to not touch the existing code so much, I want my changes to look and feel additive, this makes it easier to review correctness



Keep comments and docstrings simple, concise, informative: the important why, not the what the code already shows. Calibrate to the existing code (e.g. videoUtil.ts):
- Usually one line. A paragraph only for a genuinely subtle subsystem (cf. the seekToSafe coalescer).
- Explain the non-obvious why (a host/browser quirk, a subtle invariant) and point at the external reason.
- Natural prose. No buzzwords ("keystone", "by construction", "storm-proof", "pixel-locked"), no stacked parentheticals or "(see X)" chains.
- Don't re-derive the code's logic in prose.
- No em dashes (anywhere, comments or prose). Use a colon, comma, parentheses, or a second sentence.



## Commit message style (mirror the git history)

Conventional Commits: `type(scope): subject`.
- Types seen: `feat`, `fix`, `refactor`, `perf`, `docs`, `tests`, `chore`, `ci`, `build`, `deps`/`mdeps`. Breaking changes add `!` (e.g. `refactor(clipper)!: ...`). Version bumps are just `version: X.Y.Z` (no type/scope).
- Scopes: `clipper`, `markup`, or `clipper+markup`; for docs/ci scope the target (`docs(changelog)`, `docs(readme)`) or omit it (`ci:`). An optional secondary qualifier can follow the scope (`fix(markup): yt: ...`). Multi-part series tag the subject `[p0]`, `[p1]`, ...
- Subject: lowercase imperative, no trailing period. Make it carry the full meaning — long, specific subjects are normal and preferred over adding a body. Join two related changes with ` + `. Reference flags/keys literally (`--cookiefile`, `shift+e`).
- Body: OMIT BY DEFAULT — ~95% of commits are subject-only. Add a body only for genuinely notable mechanism/why/trade-offs, several distinct sub-changes, or breaking changes. When present, keep it terse: a 1-2 sentence prose note, or lowercase `-` bullets describing the mechanism, or a `BREAKING CHANGE:` block listing `old -> new`.
- Do not add `Co-Authored-By: Claude ...` trailer
- focus on what was fixed in the summary line, remember these get used in the auto release notes so should be user friendly. technical details can go in the body if needed.
- fix commits should generally be worder as "feature should do x"
- feature commits should concisely describe what and why feature added.


Tests should be consolidated while maintaining good coverage. Avoid brittle tests. Use parameterization and test helpers to keep test code maintainable and readable.


When tracing where a symbol is defined or finding all references to
it, use LSP (goToDefinition, findReferences, hover) instead of Grep.
LSP gives exact results; Grep gives text matches.

Use Grep/Glob for discovery (finding files, searching patterns). Use
LSP for understanding (definitions, references, type info).

After locating a file with Grep/Glob, use LSP to navigate within it
rather than reading the whole file.


prefer concision in documentation and commit messages
