# review-loop

A self-contained **review→fix loop** for [Claude Code](https://docs.claude.com/en/docs/claude-code).
Point it at your current branch and it reviews-and-fixes until nothing new turns up — then hands
you a branch to merge. It **never pushes and never opens a PR**; you do the merge.

## What makes it different

Two reviewers from **different models** — a Claude subagent and **Codex (gpt-5.5)** — read the diff
**independently and in parallel**. Because they're different models, each catches what the other
misses. Findings are merged and de-duplicated, then Codex writes the fixes. Repeat up to 4 rounds.

- **Isolated** — runs in its own git worktree, so it never touches your working copy and several
  agents can work the same repo at once. Fixes land on a dedicated `review-loop/<branch>-<sha>`
  branch.
- **Honest termination** — stops when a round surfaces no *genuinely new* finding (not just when
  the count drops), so it won't thrash on the same non-issue. Bounded at 4 rounds.
- **Self-improving** — keeps a small `knowledge-base.md` of your recurring corrections and feeds
  them into future reviews. Ships empty; fills up from your own runs.
- **Straight answers** — reports `CLEAN` / `NO-PROGRESS` / `CAP` / `VERIFY-FAILED`, and flags
  `SINGLE-REVIEWER` if Codex was unavailable, so a green result is never oversold.

## Requirements

- **Claude Code** (this is a Claude Code skill).
- **[Codex CLI](https://github.com/openai/codex)**, authenticated, with `gpt-5.5` available — it's
  the second reviewer and the fixer. Without it the loop degrades to a single reviewer.
- **git** with worktree support (2.5+), and a repo on a non-default branch to review.

## Install

### Option A — ask your agent (paste this into Claude Code)

```
Install the review-loop skill from https://github.com/AppLoidx/review-loop.
Create the directory ~/.claude/skills/review-loop/ and download these three files
from the repo's main branch into it, preserving names:
  - SKILL.md
  - codex-review-prompt.md
  - knowledge-base.md
Use the raw.githubusercontent.com URLs. Then list the directory to confirm all three
are present, and tell me the skill is ready to invoke with /review-loop.
```

### Option B — one-liner

```bash
mkdir -p ~/.claude/skills/review-loop && \
for f in SKILL.md codex-review-prompt.md knowledge-base.md; do \
  curl -fsSL "https://raw.githubusercontent.com/AppLoidx/review-loop/main/$f" \
    -o ~/.claude/skills/review-loop/"$f"; \
done && ls ~/.claude/skills/review-loop
```

### Option C — clone

```bash
git clone https://github.com/AppLoidx/review-loop ~/.claude/skills/review-loop
```

Restart Claude Code (or start a new session) so the skill is discovered.

## Usage

On a feature branch (not your default branch):

```
/review-loop
```

or just: *"loop the review"*, *"review and fix until clean"*, *"double review with codex"*.

When it finishes it reports the worktree path and the `review-loop/<branch>-<sha>` branch the
fixes are committed on. Review that branch, then merge it yourself. Clean up the worktree with:

```bash
git worktree remove <worktree-path>   # the branch and commits survive
```

## How it works (one round)

```
parallel review (Claude subagent ‖ Codex read-only)  →  dedup + classify
   →  terminate-check (CLEAN / NO-PROGRESS / CAP?)  →  fix new findings (Codex write)  →  repeat
```

- **Round 1** reviews the whole diff (`BASE..HEAD`) deeply.
- **Rounds ≥2** review only the last round's fix commits plus a regression re-check — cheaper and
  faster to converge.
- A final Codex-only backstop guards the one gap incremental review can't see (a fix that regresses
  a file outside its own delta).
- Before declaring `CLEAN` it runs the repo's typecheck/lint once.

Full behavior, preconditions, and edge cases live in [`SKILL.md`](./SKILL.md).

## Files

| File | Role |
|------|------|
| `SKILL.md` | The skill itself — all logic, preconditions, the round loop, the report format. |
| `codex-review-prompt.md` | Template for the Codex read-only reviewer (Reviewer B). |
| `knowledge-base.md` | Self-improving list of your recurring corrections. Ships empty; optional. |

## Notes & limits

- Only Codex writes the fixes — there's no cross-model check on the fix itself yet.
- The `CLEAN` gate runs typecheck/lint, **not** your full test suite — CI stays the final safety net.
- The 4-round cap is fixed; very large diffs may need a second pass.
- De-dup is theme/string based, so two findings worded very differently may not collapse.

## License

MIT
