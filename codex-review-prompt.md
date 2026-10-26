## Context
Repo: <REPO_ROOT>, review range <RANGE>.
Conventions:
<conventions>
Intent / out of scope: <intent_and_scope>
Declined ids and reasons (none in round 1): <declined_findings>

Changed files in this range (review ONLY these and regressions they introduce):
<changed_files>

## Task
Review the diff <RANGE> read-only. Identify all findings that should be fixed
before merging within the change's intent. Find real bugs in the intended case;
suggest the smallest fix. Concentrate on the changed files listed; do NOT audit untouched code outside this range. This is a
read-only review: do not edit or commit any files.

## Constraints
- Read-only: do not edit or commit any files.
- Scope: changed lines in <RANGE> and any regressions those changes introduce.
  Out-of-scope findings in pre-existing untouched code waste a round — skip them.
- Output ONLY in this exact format, one block per finding:
    file:line | severity | problem | suggested fix
  suggested fix must be concrete (not "consider refactoring").
  Severity is one of: blocker | should | nit
  blocker = the intended case is wrong.
  should = a real bug callers will hit, within the change's intent.
  nit = everything else; only blocker/should are actionable.
- Another path, a rare case, small perf cost, extra compatibility, or tests/guards
  the intent does not need = nit. Flag unneeded code rather than expanding it.
- A regression this change introduces affecting security, auth/permissions, or data loss
  is never downgraded to nit for another path or rarity.
- Read declined ids/reasons before reporting; a re-raised id must address the written reason.
- If there are no findings, output exactly the single line: NO FINDINGS
- No prose, no diffs, no explanations outside the finding blocks.

## Success criteria
Output is parsable by /codex-fix: every finding line matches
`file:line | severity | problem | suggested fix`, or the output is exactly `NO FINDINGS`.
