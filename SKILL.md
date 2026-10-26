---
name: review-loop
description: Use when you want to iteratively review-and-fix the current branch with two independent cross-model reviewers (a Claude reviewer and Codex) until findings stop, before merging — runs in its own worktree, commits fixes, never pushes or opens a PR. Triggers on /review-loop, "loop the review", "review and fix until clean", "double review with codex".
---

# review-loop

## Overview

A self-contained review→fix loop. **This skill owns all three roles** — it does NOT call the
cwd-bound `/code-review` or `/codex-fix` slash skills (they can't target a worktree). Everything is
`-C`-based so it works in any repo/worktree:

- **Reviewer A** — a primed Claude `general-purpose` subagent (`git -C`).
- **Reviewer B** — Codex, read-only (`codex exec -s read-only -C`).
- **Fixer** — Codex, write (`codex exec -s workspace-write -C`); the skill writes no fix code itself.

The two reviewers are independent and **cross-model** (Anthropic + gpt-5.5) — never merged into one
pass. They run **concurrently** (Codex in a background Bash while the subagent reviews); findings are
**deduped** before any fix. The point of the loop: each reviewer finds bugs the other (and the
fixer) missed → dedup → fix → re-review until nothing new. Default budget: 4 rounds (Step C).
Fix real bugs in the intended case; decline the rest with reasons. Keep the diff at the size the
user's goal needs, and hand disputes to the user after the first actionable re-raise.

The whole loop runs in its **own git worktree** off the target branch's committed tip — it never
touches the user's checkout, so several agents can work the same repo without colliding. Fix commits
land on a dedicated `review-loop/<branch>-<sha>` branch; the user merges from there. Never pushes,
never opens a PR.

Termination is **identity-based**, not count-based: a round makes progress only if it surfaces a
*new* actionable finding. No actionable findings this round and no open actionable ids leads to
CLEAN after the conditional backstop and Final verify; nits remain follow-ups. Open actionable ids
with nothing new actionable means NO-PROGRESS. Budget exhaustion means CAP; disputes go to the user.

## Preconditions (fail fast, in order)

1. `git rev-parse --show-toplevel` from the target repo succeeds → `MAIN_ROOT`. Not a repo → stop.
2. `TARGET_BRANCH = git -C "$MAIN_ROOT" branch --show-current`. Equals the default branch (`origin/HEAD`, else `main`/`master`) → stop, ask user to branch first.
3. **Isolate in a dedicated worktree — ALWAYS.** The loop checks out a branch, commits, and needs a clean tree; doing that in the user's checkout collides with other agents on the same repo. So:
   - `LOOP_BRANCH="review-loop/${TARGET_BRANCH//\//-}-$(git -C "$MAIN_ROOT" rev-parse --short HEAD)"` (ties the branch to the exact reviewed commit; collision-safe across agents). `WT="$(dirname "$MAIN_ROOT")/.review-loop-$(basename "$MAIN_ROOT")-${LOOP_BRANCH##*/}"`.
   - Off the **committed** tip: `git -C "$MAIN_ROOT" worktree add "$WT" -b "$LOOP_BRANCH" HEAD`. Branch already exists → **resume**: `git -C "$MAIN_ROOT" worktree add "$WT" "$LOOP_BRANCH"`; if a live worktree holds it, append `-2`. Uncommitted WIP in `MAIN_ROOT` is NOT included — commit it first if it should be reviewed.
   - **Set `REPO_ROOT="$WT"` for every step below.** Because everything below is `-C`-based, no slash tools are involved, so the worktree-vs-launch-dir mismatch never matters.
4. `codex --version` succeeds AND a model probe passes — `codex exec -m gpt-5.5 -s read-only -C "$REPO_ROOT" - <<<'reply with OK'` returns without a model-rejection error. Codex missing/unauthed → stop. Model `gpt-5.5` rejected → STOP and ask which model; never silently substitute.
5. `BASE=$(git -C "$REPO_ROOT" merge-base "$(git -C "$REPO_ROOT" rev-parse --abbrev-ref origin/HEAD 2>/dev/null | sed 's@^origin/@@' || echo main)" HEAD)`
6. `STATE_DIR="$(git -C "$REPO_ROOT" rev-parse --git-path review-loop)"` — `mkdir -p`, `rm -f "$STATE_DIR"/*` stale scratch. Use `--git-path` (NOT `$REPO_ROOT/.git/...`): a linked worktree's `.git` is a FILE. Scratch lives in the git dir → never in the working-tree review, never inflates findings.
7. Read `knowledge-base.md` (this skill dir) if present → `KB_CHECKS`. Do NOT inject all — **select the top ~8 by `Seen (N)` plus any matching the changed files' area**. See **Knowledge base**.

## The three roles (this skill owns them)

### Reviewer A — Claude subagent

Dispatch `general-purpose` subagent(s) (full model, NOT `cavecrew-reviewer` — its token-compression
produced a whole false-positive cohort). Give each: `git -C "$REPO_ROOT"`, the round's range, the
distilled `<conventions>` + selected `KB_CHECKS`, an explicit file group, and the **Reviewer-A
brief** below. For a large diff (≳15 changed files) fan out several reviewers over disjoint file
groups — one under-covers. Each returns findings as `file:line | severity | problem | suggested fix`
(or `NO FINDINGS`). No 4-cap (subagents return everything), so round 1 needs no "drain".

**Reviewer-A brief** (give to every reviewer-A subagent): cite a real `file:line` for every finding
— no memory assertions. **Before** flagging, VERIFY: read `package.json` for the framework version
before calling syntax "invalid" (e.g. Tailwind v4 `(--var)`, `**:`, `in-*` are valid); check the
repo's sibling convention before calling a missing directive a bug; read the design doc before
asserting a token is wrong. Restrict strictly to changed lines. A finding you could not verify →
label it "unverified", do not assert it.
Severity: `blocker` = the intended case is wrong; `should` = a real bug callers will hit, within
this change's intent; `nit` = everything else. Only blocker/should are actionable. Another path,
a rare case, small perf cost, extra compatibility, or tests/guards the intent does not need = nit.
Exception: a regression this change introduces affecting security, auth/permissions, or data loss
is never downgraded to nit for another path or rarity. Read declined ids/reasons before reporting;
a re-raised id must address the written reason.

### Reviewer B — Codex read-only

Fill `codex-review-prompt.md` (this skill dir) and run `codex exec -m gpt-5.5 -s read-only -C
"$REPO_ROOT"` (details in Step A). Independent of Reviewer A; same output format.

### Fixer — Codex write (inlined `codex-fix`)

The skill never hand-writes fixes (one exception: ≤2 trivial mechanical edits where spinning Codex
isn't worth it — a direct `Edit` is fine). Otherwise Codex authors them:
1. Write the round's deduped actionable findings to `$STATE_DIR/rR-fix.md` (the `file:line | …`
   format), **batched by file** (all findings for one file together).
2. `codex exec -m gpt-5.5 -s workspace-write -C "$REPO_ROOT" -o "$STATE_DIR/rR-fix-out.md" - < <fix-prompt>`
   where the fix prompt = the findings + `<conventions>` + "touch ONLY these files, match
   surrounding style, no marker comments, no scope creep, do not commit. Smallest fix; prefer
   deleting or narrowing code over a new lock, cache, helper or option. Fix the case that happened."
   For a large file-disjoint set you may run several `codex exec` in parallel (background Bash) — but COMMIT sequentially
   (clean tree between).
3. **Verify before commit**: the diff is non-empty and addresses each finding; run the repo
   typecheck/lint on it (empty diff or new errors → re-run Codex once with the failure appended;
   still failing → fix that batch inline). Apply the Step D hunk check before committing.
   Then `git -C "$REPO_ROOT" commit` on `LOOP_BRANCH`.

## Round Loop

For R in 1..LIMIT (`LIMIT = 4` initially; Step C allows one extra), each round is:
**parallel review → dedup → terminate-check → fix (if continuing)**.

### Review range per round (converge in fewer, cheaper rounds)

Track `PREV` = HEAD at the end of the previous round (`PREV = BASE` before round 1).
Before round 1, record intent/out-of-scope in 1-2 sentences and the baseline shortstat:
`git -C "$REPO_ROOT" diff --shortstat BASE..HEAD` (substitute the saved `BASE` SHA).

- **Round 1 — FULL.** Range = `BASE..HEAD`. Review everything, deeply (medium/normal Codex effort).
- **Rounds ≥2 — INCREMENTAL.** Range = `PREV..HEAD` (only the last round's fix commits) + re-verify
  the findings just fixed. Far less to read, and a fix's regressions live in exactly those lines.
  Small delta → drop Codex effort (`-c model_reasoning_effort=low`) and tell the subagent it's a
  quick regression check.
- **Final backstop (conditional, Codex-only).** Guards the one incremental gap — a fix that
  regresses a file NOT in its own delta:
  - **No fix applied all loop** → **skip** (nothing to regress) → Final verify.
  - **Any fix applied** → ONE **Codex-only** read-only review scoped to the **union of fix-touched
    files** at `BASE..HEAD`. Codex alone — the reliable reviewer; the subagent backstop only added
    false-positive noise. Merge backstop findings into this round's findings, then reclassify in
    Step B using the pre-round SEEN set. Mark the backstop done for this HEAD before re-checking
    Step C; never repeat it on the same HEAD. A fix changes HEAD and requires a fresh backstop.

Substitute the round's range as `<RANGE>` and `git -C "$REPO_ROOT" diff --name-only <RANGE>` as
`<changed_files>` in both reviewers' inputs. Include the intent/out-of-scope brief and, in later
rounds (including the backstop), declined ids with their written reasons for both reviewers.

### Step A — two reviews, concurrently

1. Fill `codex-review-prompt.md`: substitute `<RANGE>`, `<changed_files>`, `<conventions>` (5-15
   lines distilled from the repo's CLAUDE.md/AGENTS.md; omit if none) **plus the selected
   `KB_CHECKS`**, `<intent_and_scope>`, and `<declined_findings>` (ids/reasons, or none). The scoped range + changed-file list keep Codex on the diff (faster, fewer
   out-of-scope FPs). Write to `$STATE_DIR/rR-prompt.md`.
2. **Launch Codex (Reviewer B) in the background** (Bash `run_in_background`, timeout 600000):
   ```
   codex exec -m gpt-5.5 -s read-only -C "$REPO_ROOT" -o "$STATE_DIR/rR-codex.md" - < "$STATE_DIR/rR-prompt.md"
   ```
   Incremental rounds (≥2): append `-c model_reasoning_effort=low`.
3. **While Codex runs**, dispatch **Reviewer A** (general-purpose subagent(s), `git -C "$REPO_ROOT"`,
   the range, conventions + `KB_CHECKS`, the Reviewer-A brief; fan out for a large diff). Collect
   findings_A. `NO FINDINGS` → findings_A = 0. Also scan the diff for the selected `KB_CHECKS`.
4. Collect Codex output → normalize → findings_B. Malformed/none parsable → findings_B = 0 (optionally
   re-prompt once). Hard failure (non-zero exit / timeout) → findings_B = 0, note it, **increment a
   consecutive-B-failure counter**, CONTINUE.

> Why parallel: both reviews are read-only and independent, so running them together halves review
> latency and lets identical findings dedup. Cost: B no longer sees A's fixes within the round —
> regressions a fix introduces are caught as NEW findings next round (the SEEN set preserves the
> cross-check, shifted one round).

### Step B — dedup + classify

Merge findings_A + findings_B. **key** = relative file path + normalized problem theme (lowercase
first ~6 significant words; ignore line-number drift).

- Re-grade EVERY finding yourself using the Reviewer-A severity rubric, whatever label it came
  with: blocker/should only are **actionable**; everything else is a **declined nit**. Another path,
  rare case, small perf cost, extra compatibility, or unneeded tests/guards = nit, subject to the
  rubric's security/auth/permissions/data-loss regression exception.
- False positive disproved with a file:line / command-output citation = **refuted**; cite it.
  Record every declined nit/out-of-scope finding with a one-line reason and list it as a follow-up.
- `SEEN` = keys from ALL prior rounds (actionable, refuted AND declined). Use these keys as ids.
  `NEW_R` = actionable findings whose
  key ∉ `SEEN`. Add every key this round to `SEEN`. `T_R` = actionable count this round (report).
- Same key from both reviewers → collapse to one, fix once.
- An actionable id is **resolved** when its fix commit passed Fixer step-3 verify and the next
  review of that range does not re-raise it. Until that review, mark it **fixed, pending review**.
  **Open** = actionable and not fixed (including failed fixes, CAP residuals, and disputed ids).
  A fixed id re-raised as actionable (after your re-grade) is open again; do not re-fix repeats
  automatically. A nit re-raise of a fixed id resolves it and goes to follow-ups.
- The FIRST re-raise after a written decline/refutation goes under **Disputed** for the user;
  a re-raise the reviewer labels nit is agreement, not a dispute. Keep disputed ids open and
  separate from other NEW_R findings. Fix those other findings first if budget allows, then stop
  with both sides reported; otherwise list them as open too. Never agree just to end the loop.

### Step C — terminate-check (first match wins)

At the default limit, allow exactly one extra round (`LIMIT = 5`) only if an open blocker/should
concerns security, permissions/auth, data loss, or state/concurrency **with evidence**, or the user
asks. Record the reason and evidence; never extend again. Budget exhaustion is not approval.

| Condition | Action |
|---|---|
| disputed ids remain | if `NEW_R > 0` and `R < LIMIT`, Step D fixes only the other new actionable ids, then stop **NO-PROGRESS** for the user; otherwise exit **NO-PROGRESS** and list all open items. Flag fixes made on this path as unreviewed in the Report |
| `T_R == 0` AND no open actionable ids | run the conditional backstop if fixes were applied and it is not done for this HEAD; merge its findings into this round and re-check via Step B/C once. With zero actionable findings and no open ids after backstop (or skip), **Final verify**, then **CLEAN**; nits stay follow-ups |
| open actionable ids remain AND `NEW_R == 0` | exit **NO-PROGRESS**; do NOT fix repeats |
| `R == LIMIT` AND `NEW_R > 0` | exit **CAP**; report all open residuals, do NOT fix (it would be unreviewed) |
| `NEW_R > 0` AND `R < LIMIT` | go to Step D |

### Step D — fix the new findings

Run the **Fixer** (above) on `NEW_R` (actionable, deduped, batched by file): `codex exec -s
workspace-write -C "$REPO_ROOT"`, verify, commit on `LOOP_BRANCH`. Then next round, unless Step C
requires a dispute handoff; report those fixes as pending review when stopping.

Before committing on EVERY fix round, include new files with `git -C "$REPO_ROOT" add -N .`.
Measure `git -C "$REPO_ROOT" diff --shortstat BASE` (saved SHA) and inspect every added hunk in
`git -C "$REPO_ROOT" diff HEAD`, regardless of net growth. Revert hunks touching code no finding
named (scope creep), then verify retained fixes before commit. Shortstat deltas are report-only.
After each fix round, record `git -C "$REPO_ROOT" diff --shortstat BASE..HEAD` again (saved SHA).

### Final verify (gates CLEAN only)

Before declaring CLEAN, run the repo's whole-tree checks once (per-batch fix verification is not a
full-tree check): derive from CLAUDE.md/AGENTS.md / `package.json` (e.g. `pnpm typecheck && pnpm
lint`, else `npx tsc --noEmit` + the lint script). None → note "no typecheck configured".
- Pass → CLEAN. Fail → **VERIFY-FAILED**: surface the errors as residual findings; do NOT claim CLEAN.
- Stale env noise is not a regression: missing-dependency errors (`Cannot find module`) → run
  `pnpm install --frozen-lockfile` (also confirms the lock is canonical) and re-check; generated
  artifacts outside the diff (e.g. `.next/types`) are pre-existing, note and ignore.

## Knowledge base (self-improving)

`knowledge-base.md` (this skill dir) holds the user's recurring corrections:

- **Load + select** at start (precondition 7) → `KB_CHECKS` = top ~8 **Generic process lessons** by
  `Seen (N)` plus any matching the changed files' area. Not the whole file (it feeds every prompt).
- **Feed both reviewers**: in the Reviewer-A brief and appended to the Codex prompt conventions.
- **Self-improve (after the run)**: matched a known theme → bump `Seen (N)`; a NEW recurring issue
  (or a user correction) → append a one-line row + a Changelog entry. Prune `Seen 1`/dupes when it bloats.

If `knowledge-base.md` is absent, skip all KB steps (the loop still runs).

## Report (then stop — no push, no PR)

1. **Where** — `WT` and `LOOP_BRANCH` the fixes are committed on. The user merges `LOOP_BRANCH` into
   the target branch (or you do, only if explicitly asked). Cleanup after merge:
   `git -C "$MAIN_ROOT" worktree remove "$WT"` — branch and commits survive.
2. **What changed** — per-round counts (`T_R`, `NEW_R`, A/B split w/ severities), files touched,
   `git -C "$REPO_ROOT" log --oneline BASE..HEAD`; diff size before/after (shortstat).
3. **Why** — findings that drove each fix, grouped by round/file; refuted findings with their disproof.
4. **Context** — code/diff excerpts for notable fixes.
5. **Outcome** — CLEAN / NO-PROGRESS / CAP / VERIFY-FAILED + remaining unfixed findings; rounds
   used and the extra-round reason/evidence, if any. Open items stay open at the budget.
   **Follow-ups** — every declined id and its one-line reason.
   **Disputed** — for each first actionable re-raise after a written decline/refutation, both
   sides in 2-3 lines each, with evidence; the user decides.
6. **Reviewer health** — Codex hard-failed ≥2 rounds → label outcome **SINGLE-REVIEWER (Codex
   unavailable)**; a CLEAN reached with one reviewer is NOT the full double review.
7. **Verification caveat** — Final-verify result (or "not configured"); full test suite NOT run. Not
   pushed, no PR — the user merges; CI is the safety net.

## Edge Cases

- Round-1 zero actionable findings and no open ids: Final verify, exit CLEAN, still report follow-ups.
- Codex emits malformed/no parsable findings: findings_B = 0, note; optionally re-prompt once.
- `codex exec` hard failure: findings_B = 0, note, bump the counter, CONTINUE. Fails every round →
  one reviewer → SINGLE-REVIEWER outcome.
- `codex-review-prompt.md` missing from this skill dir: stop — it ships with the skill; restore it.
- Only refuted and/or nit findings: Step C's zero-actionable path; no empty fix round. Cite
  refutations and list declined nits as follow-ups; open actionable ids still prevent CLEAN.
- Fixer produces an empty diff for a batch: treat as a failure — re-run Codex once, else fix inline.

## Red Flags — STOP

- About to push or open a PR → never. The user merges.
- Running the loop in the user's checkout instead of a dedicated worktree → no; always isolate
  (precondition 3) so concurrent agents on the same repo don't collide.
- Hand-writing a non-trivial fix → no; Codex authors fixes (`codex exec -s workspace-write`). Only a
  handful of trivial mechanical edits may be a direct `Edit`.
- Writing loop scratch (`*-state-*.md`, prompts) into the working tree → no; it lives in `$STATE_DIR`
  (the worktree's git dir, via `git rev-parse --git-path`) so it can't be reviewed as a finding.
- Using `cavecrew-reviewer` as Reviewer A → no; it's token-compressed and produced a false-positive
  cohort. Reviewer A is a full-model `general-purpose` subagent.
- Only ran one reviewer this round when both were available → a round needs BOTH reviewers.
- Loop with no round counter / no cap → default 4, at most 5 under Step C.
- Terminating on raw counts: NO-PROGRESS needs open actionable ids and no new actionable findings
  (or a dispute handoff); zero actionable findings with no open ids takes the CLEAN path.
- Applying a fix at the active cap → no, it would be unreviewed.
- Claiming CLEAN with open actionable ids or fixes pending review: require zero actionable findings,
  confirmed resolution, the conditional backstop, and passing Final verify.
- Reporting CLEAN as a full double review when Codex failed every round → label it SINGLE-REVIEWER.
- Fixing nits or accepting a reviewer label without re-grading: decline with a reason (Step B).
- Skipping a hunk check because the diff shrank: check EVERY added hunk, including new files.
- Agreeing to end an argument: report the first actionable re-raise after a written decline; the user decides.
