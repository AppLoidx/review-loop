## Context
Repo: <REPO_ROOT>, review range <RANGE>.
Conventions:
<conventions>

Changed files in this range (review ONLY these and regressions they introduce):
<changed_files>

## Task
Review the diff <RANGE> read-only. Identify all findings that should be fixed
before merging — bugs, security issues, performance problems, quality issues,
and style deviations from the conventions above. Concentrate on the changed
files listed; do NOT audit untouched code outside this range. This is a
read-only review: do not edit or commit any files.

## Constraints
- Read-only: do not edit or commit any files.
- Scope: changed lines in <RANGE> and any regressions those changes introduce.
  Out-of-scope findings in pre-existing untouched code waste a round — skip them.
- Output ONLY in this exact format, one block per finding:
    file:line | severity | problem | suggested fix
  where severity is one of: bug | security | perf | quality | style
  and suggested fix must be concrete (not "consider refactoring").
- If there are no findings, output exactly the single line: NO FINDINGS
- No prose, no diffs, no explanations outside the finding blocks.

## Success criteria
Output is parsable by /codex-fix: every finding line matches
`file:line | severity | problem | suggested fix`, or the output is exactly `NO FINDINGS`.
