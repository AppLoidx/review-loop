# review-loop knowledge base
> Your recurring review corrections, distilled from your own sessions. review-loop consults this
> before/while reviewing and APPENDS new recurring corrections after runs. Self-improving.
>
> This ships EMPTY on purpose — it fills up from YOUR reviews so the checks match the mistakes
> YOU actually make. The skill runs fine with an empty table; it just has nothing extra to look
> for yet. Delete this file entirely to disable the KB feature.

## How review-loop uses this
- At loop start, read this file. Fold the top ~8 generic rules (by `Seen (N)`) plus any matching
  the changed files' area into BOTH reviewers: append them to the Codex review-prompt conventions
  and to the Claude reviewer brief.
- After a run: if you corrected something not listed here, append it under the right section (one
  line); if a listed theme recurred, bump its `Seen (N)`. This is how the skill improves itself.
- Prune `Seen 1`/duplicate rows when the file bloats (it feeds every prompt — keep it lean).

## Generic process lessons

| Lesson | Why it matters | Review check | Seen (N) |
|--------|---------------|--------------|----------|
| _(empty — populated from your runs)_ | | | |

## Project-specific
<!-- One subsection per repo you use this in, e.g. `### my-app`. Rows here only load when the
     changed files match that project's area. Keep company/internal specifics in YOUR fork, not
     upstream. -->
