# Agentic UI Flow Critique Evaluation Suite

## Purpose

This is a reusable evaluation suite for an **agentic UI flow/state critique prompt** — a tool used in day-to-day UX/UI design work to systematically review agent-assisted workflows and identify UX gaps.

The prompt helps designers and product teams:
- Identify missing progress/status indicators in multi-step agent tasks
- Catch confirmation gaps before destructive actions
- Spot user control/interrupt issues in streaming or long-running operations
- Ensure error states provide recovery guidance
- Verify agent reasoning and decision-making is transparent to users

**Weekly usage frequency**: ~5–7x per week (daily design review cycle)

## Problem It Solves

When designing agentic UX, it's easy to miss critical patterns:
- A multi-step task with no progress indicator leaves users confused
- An auto-executing destructive action violates user control principles
- A streaming response with no cancel button traps users
- A failed handoff with no error message leaves users stuck

This prompt provides a **structured, repeatable critique** that catches these issues early, grounded in agentic UX heuristics (progress visibility, user control, error recovery, confirmation patterns, transparency).

## Setup

### Prerequisites
- **Node.js** (v16+)
- **Ollama** installed locally with two models pulled:
  - `llama3.2` — used for generation (critiquing the flows)
  - `llama3.1:8b` — used for grading (`llm-rubric` assertions); pulled on Day 3 after the smaller `llama3.2` proved unreliable as a grader (see Evaluation Rubric section)

### Installation

1. Clone or download this repo to `/Users/mario.miqueles/Documents/improving-certification-stage-3`
2. Install dependencies:
   ```bash
   npm install
   ```
3. Verify Ollama is running and check your model:
   ```bash
   ollama list
   ```
   If your model is not `llama3.1`, update `promptfooconfig.yaml` under `providers` with your model tag (e.g., `ollama:chat:mistral`).

## Running Evaluations

### Evaluate v1 (baseline) only:
```bash
npm run eval:v1
```

### Evaluate v1 and v2 side-by-side:
```bash
npm run eval
```

### View results in web UI:
```bash
npm run view
```
This opens a local web interface showing scores, outputs, and side-by-side comparisons.

## Project Structure

```
.
├── README.md                          # This file
├── package.json                       # npm scripts and dependencies
├── promptfooconfig.yaml               # Promptfoo config (providers, tests, assertions)
├── progress.txt                       # Working notes (iteration log)
├── prompts/
│   ├── v1-flow-critique.txt          # Baseline/naive prompt (under-specified)
│   ├── v2-flow-critique.txt          # Improved prompt (structured, heuristic-grounded) - recommended
│   └── v3-flow-critique.txt          # Day 5 experiment (fabrication fix attempt) - not an improvement, kept as documented result
├── testcases/
│   └── fixtures.yaml                 # 8 agentic UI flow scenarios
└── results/                           # Canonical exported eval results per day (committed, not generated-only)
    ├── day1-results.json
    ├── day3-results.json
    ├── day4-results.json
    ├── day5-results.json
    ├── day6-results.json
    └── day7-results.json             # Final result for certification (no Day 2 file - reused Day 1's eval)
```

## Evaluation Rubric

Each test case is scored on five criteria (`promptfooconfig.yaml` → `defaultTest.assert`). Every assertion carries a `metric` name so `promptfoo view` shows a distinct, labeled pass/fail (and score) per criterion instead of one lumped number:

1. **Actionability — Specificity** (pass/fail, `llm-rubric`, metric `actionability-specificity`): Do the critique's fix suggestions reference specific, concrete UI states/interactions from the flow (a specific button, message, timing, or step), rather than describing the problem only in the abstract?

2. **Actionability — Concreteness** (pass/fail, `llm-rubric`, metric `actionability-concreteness`): Does the critique provide at least 3 fix suggestions that are concrete and implementable as stated (not generic advice like "improve the experience")?
   - Example concrete: "Add a progress bar showing 'Step 2/5: Cleaning data' with estimated time remaining"
   - Example generic: "Improve the user experience"
   - Note (Day 3): this was originally a single combined "actionability" criterion. It was split into these two binary sub-criteria so the rubric can distinguish *why* a critique fails to be actionable (ungrounded in the flow's specifics vs. suggestions being vague) rather than reporting one opaque pass/fail. Kept binary rather than a 1-5 graded score, since Day 1 found the local 2GB `llama3.2` grading model unreliable at numeric/counting judgments.

3. **Coverage of Agentic UX Heuristics** (pass/fail, deterministic keyword count via `javascript` assertion, metric `heuristic-coverage` — requires ≥2 of 5 to pass):
   - Progress/status visibility
   - User control and override capability
   - Error recovery guidance
   - Confirmation before destructive actions
   - Transparency of agent reasoning

   Note: this was originally an `llm-rubric` ("count how many topics are mentioned"), but the local grading model (2GB `llama3.2`) proved unreliable at counting/enumeration tasks. It's now a deterministic keyword match against the fixed, closed set of heuristic names the v2 prompt requires, which is both more accurate and doesn't depend on the local model's reasoning.

4. **Structure/Consistency** (binary, metrics `structure-issue-marker` / `structure-severity-marker` / `structure-fix-marker`): Does the output contain the literal `**Issue**`/`## Issues`, `**Severity**`, and `**Fix Suggestion**`/`**Fix**` markdown markers (all three required)?
   - Checked via 3 separate `regex` assertions. Requiring all three (rather than an OR of loose words like "issue"/"fix") avoids false passes from v1's free-form prose incidentally containing those words.

5. **Conciseness** (binary, metric `filler-phrase-check`): No filler phrases
   - Checked via multiple `not-icontains` assertions for phrases that are unambiguously generic in any context (e.g. "improve the experience"). Single words like "user-friendly" were deliberately excluded from this list since they can appear inside an otherwise concrete, specific suggestion.

**Grading provider**: all model-graded (`llm-rubric`) assertions run against a local Ollama model, set via `defaultTest.options.provider`, so the whole suite runs offline with no `OPENAI_API_KEY` required. Generation still uses `ollama:chat:llama3.2` (2GB, unchanged since Day 1, kept for comparability across days), but grading was moved to `ollama:chat:llama3.1:8b` (4.9GB) on Day 3 — see caveat below. Both providers use `temperature: 0` for reproducible results.

**Day 3 grading-reliability investigation**: splitting actionability into two binary sub-rubrics initially surfaced cases where the 2GB `llama3.2` grading model's stated `reason` text contradicted its own `pass` verdict for a single, narrow yes/no question (e.g., reasoning that reads as clearly affirmative while `pass: false`). Three mitigations were tried, in order:
1. `temperature: 0` — eliminated *flakiness* (verified via `--repeat 2`: identical verdicts every time for the same input), but did not fix the underlying contradictions, which turned out to be reproducible/systematic rather than random.
2. A stronger rubric prompt requiring the grader to state evidence for/against and forbidding a reason/pass mismatch — measurably reduced (but did not eliminate) contradictions.
3. Switching the **grading provider only** (not generation) to `llama3.1:8b` — resolved the remaining contradictions in testing (verified via `--repeat 2`, 0 inconsistencies, and manual reading of `reason` text against `pass` values).

This is why generation and grading now use two different local models: the 2GB model is good enough to critique a UI flow, but not reliable enough to grade its own kind of critique against a binary rubric.

## Quantified Improvement Evidence

### Final numbers (as of Day 7)

Canonical, reproducible export: **[`results/day7-results.json`](results/day7-results.json)** — `npm run eval -- --no-cache --output results/day7-results.json` (eval ID `eval-yPR-2026-08-16T12:57:39`), 8 fixtures × 3 prompt versions = 24 test cases, graded against the current 5-criteria rubric above. Identical to Day 6's result (`results/day6-results.json`, eval `eval-8ua-2026-08-15T14:29:24`) — reproduced exactly with no changes to prompts/config in between, confirming the suite is stable.

Full result history is in `results/`: `day1-results.json` (Day 1), `day3-results.json` (Day 3), `day4-results.json` (Day 4), `day5-results.json` (Day 5), `day6-results.json` (Day 6), `day7-results.json` (Day 7, final). There's no separate Day 2 file since Day 2 reused Day 1's eval rather than running a new one.

| Version | Pass rate | Takeaway |
|---|---|---|
| **v1** (baseline) | 0/8 (0%) | Fails structure every time (never instructed to use an Issue/Severity/Fix format); leans on generic framing even when individual suggestions are reasonable. |
| **v2** (improved) | 8/8 (100%) | **Recommended prompt.** Consistently structured, cites specific UI states, labels a heuristic per issue — but see the caveat below: it also fabricates issues on well-designed flows, which this rubric can't yet detect. |
| **v3** (Day 5 experiment) | 4/8 (50%) | Attempted fix for v2's fabrication problem; did not work and is *not* an improvement over v2 — kept as a documented negative result. |

**Important caveat**: pass/fail is intentionally strict (requires every criterion simultaneously), and a "pass" measures *structure + specificity + non-genericness* — it does **not** verify that flagged issues are real. Fixture 8 ("well-designed export flow," which has no genuine UX problems) still causes v2 to score 100% by fabricating plausible-sounding issues. Treat these percentages as directional evidence of prompt-structure quality for this suite, not a universal correctness score. Full details on this gap are in the Day 4/5 log entries below.

### Investigation Log (Day 1 → Day 7)

<details>
<summary>Click to expand the day-by-day narrative behind the numbers above</summary>

**Day 1 baseline** — Measured via `npm run eval` on 2026-08-10 (5 fixtures, v1 vs v2, original 4-criteria rubric): v1 = 0/5 (0%), v2 = 5/5 (100%). v1 fails the structure check every time and leans on generic framing ("...to improve the user experience") even when some individual suggestions are reasonable; v2 consistently follows the required structure, cites specific UI states, and labels each issue with a specific heuristic. **Improvement**: +100 percentage points in pass rate (0% → 100%) on this rubric and fixture set. Full per-assertion results: **[`results/day1-results.json`](results/day1-results.json)** (this file existed locally since Day 1 but was gitignored and never actually committed until the Day 6 cleanup below — now it's real, committed evidence).

**Day 3 update**: after splitting actionability into two binary sub-criteria, adding a 6th fixture (legal research agent), and fixing a grading-reliability issue (moved grading to `llama3.1:8b`, `temperature: 0` — see Evaluation Rubric section above), the final Day 3 run (**[`results/day3-results.json`](results/day3-results.json)**, `eval-cuX-2026-08-12T17:48:25`, 6 fixtures × 2 prompts) showed v1 = 0/6 (0%), v2 = 6/6 (100%) — a cleaner, more trustworthy result than the intermediate runs during that investigation (which used the unreliable 2GB grading model and produced a misleadingly low 1/6 pass rate for v2). Verified stable across 2 repeated runs (0 inconsistencies).

**Day 4 update — stress-test with a bad flow and a well-designed flow**: added fixtures 7 ("Compounding failures") and 8 ("Well-designed export flow") specifically to check whether the rubric behaves sensibly at the extremes. `npm run eval --no-cache` (**[`results/day4-results.json`](results/day4-results.json)**, `eval-CIY-2026-08-13T13:26:24`, 8 fixtures × 2 prompts): v1 = 0/8 (0%), v2 = 8/8 (100%) — same pattern as Day 3.

The compounding-failures fixture worked as intended (v2 correctly flagged the irreversible-submission issue as top severity among several). **The well-designed fixture did not** — v2 still passed all 6 criteria by fabricating 5 issues on a flow that has no real problems (e.g., flagging "inadequate confirmation before destructive actions" on a flow that explicitly has a confirmation dialog). v1 was comparatively more honest (it opened with a "Strengths" section) but still manufactured some generic critique.

This is a known, currently-unresolved gap, not a hidden one: neither prompt has permission to report "no significant issues found," and the rubric's requirement for ≥3 concrete fixes structurally rewards inventing problems on a clean design with a full pass. v2's 100% pass rate should be read as "v2 always produces a well-structured, heuristic-labeled, non-generic critique" — not as "v2 always correctly identifies real problems." Fixing this was queued for Day 5's prompt iteration. See `progress.txt` Day 4 notes for the full critique text and reasoning.

**Day 5 update — v3 attempted to fix the fabrication gap, and did not succeed**: `prompts/v3-flow-critique.txt` adds explicit permission to respond "No significant issues found" plus a "do not fabricate/nitpick cosmetics" rule, and `promptfooconfig.yaml`'s rubric was updated (OR-clauses on `structure-severity-marker`/`structure-fix-marker`/`heuristic-coverage`, updated `actionability-*` wording) so a correct no-issues response wouldn't be automatically penalized. `npm run eval --no-cache` (**[`results/day5-results.json`](results/day5-results.json)**, `eval-iuL-2026-08-14T14:30:26`, 8 fixtures × 3 prompts) result: v1 = 0/8, v2 = 8/8, v3 = 4/8 — v3 scored worse than v2.

v3 still fabricated 5 issues on the well-designed flow (same defect as v2), and introduced two new artifacts: a contradictory trailing "No significant issues found." line appended *after* a correct 5-issue critique on the compounding-failures fixture, and a spurious echoed `## Constraints` section in several outputs that confused the grader. A simplified single-template rewrite was tried as a second attempt (`eval-ELw-2026-08-14T15:46:18`, v3 only) but regressed further — 0/8 pass, and the model now stops after finding just 1 issue on flows with multiple genuine problems.

**Conclusion**: `prompts/v3-flow-critique.txt` in this repo is attempt 1 (the better of the two, kept as an honest documented result) — not a working fix. **v2 remains the best-performing, recommended prompt.** The fabrication gap appears to need more than prompt wording changes on this 2GB generation model to resolve; candidates for a future attempt include a bigger local generation model (mirroring the Day 3 grading-model fix) or a concrete few-shot example of a correct "no issues" response. See `progress.txt` Day 5 notes for full critique text and root-cause analysis.

**Day 6 correction note**: while preparing the final `results/day6-results.json` export, discovered that a Day 5 file edit meant to revert `prompts/v3-flow-critique.txt` back to "attempt 1" had silently failed to persist — the file committed at the end of Day 5 was actually "attempt 2" (the worse, simplified rewrite), even though Day 5's write-up described attempt 1 as the kept version. This was caught because Day 6's fresh eval run produced v3 = 0/8 instead of the previously-reported 4/8. Fixed by rewriting the file directly via shell and re-verifying with a fresh `--no-cache` run, which reproduced the original 4/8 result exactly, confirming the fix. The `results/day6-results.json` file linked above reflects the corrected, verified `v3-flow-critique.txt`.

**Day 7 — final verification and wrap-up**: backfilled `results/day3-results.json`, `day4-results.json`, and `day5-results.json` from local promptfoo history (they were documented by eval ID throughout this log but never actually exported until today), so every day's claimed numbers now have a real, inspectable file backing them, not just narrative. Ran one final `npm run eval --no-cache` (`results/day7-results.json`, `eval-yPR-2026-08-16T12:57:39`) — reproduced Day 6's exact numbers (v1=0/8, v2=8/8, v3=4/8), confirming the suite is stable and nothing regressed since the Day 6 fix. This is the final state of the suite for certification review.

</details>

## Version History

- **v1** (baseline): Naive, under-specified prompt with no output structure or persona. Serves as the "before" state.
- **v2** (improved): Structured prompt with defined persona (senior agentic-UX reviewer), explicit output format (Issues / Severity / Fix), constraints (max 5 issues, cite specific states, no fluff), and grounding in agentic UX heuristics. **Currently the best-performing, recommended prompt.**
- **v3** (Day 5 experiment, not an improvement): Attempted to add permission to report "No significant issues found" on well-designed flows, to fix the fabrication gap found on Day 4. Did not work on this 2GB generation model — still fabricated issues on the well-designed flow, and introduced new formatting/completeness regressions. Kept in the repo as a documented negative result; see the Day 5 update above and `progress.txt` for full details.

See `git log --oneline` for iteration history.

## Test Cases

The suite includes 8 agentic UI flow scenarios:

1. **Multi-step processing with no progress indicator** — User uploads CSV, agent processes across 5 steps with only "Processing..." message; no cancel, no error recovery.
2. **Destructive action without confirmation** — Chat agent auto-approves all expenses without summary or confirmation dialog.
3. **Streaming response with no interrupt** — Agent streams long response; user cannot cancel mid-stream if agent goes off-track.
4. **Silent handoff failure** — Agent handoff to human specialist fails silently; no error message, retry, or fallback contact info.
5. **Opaque agent reasoning** — Agent recommends a product with no explanation of reasoning, data sources, or confidence; user cannot override.
6. **Hidden confidence and provenance gap** *(Day 3)* — Legal research agent's contract risk analysis has no inline citations, confidence scores, or source attribution.
7. **Compounding failures** *(Day 4)* — Financial reconciliation agent stacks multiple violations at once: irreversible auto-submission with no confirmation, no progress, no cancel, and a blank error screen on partial failure.
8. **Well-designed export flow** *(Day 4)* — Deliberately has *no* real UX issues (progress bar, cancel button, confirmation dialog, detailed error+retry, explained/overridable defaults); added to stress-test whether the rubric can distinguish a genuinely good flow from a bad one. See the Day 4 finding below — it currently can't.

## Daily Testing Cadence (Week 1)

This is a **7-day iteration loop**. Each day you run the eval, review results, and (on key days) adjust the prompt/rubric based on what you observe.

### Day 1 — Baseline
1. Confirm Ollama model: `ollama list`
2. Run `npm run eval:v1`
3. Run `npm run view` — review v1 outputs and rubric scores
4. Judgment: Does v1 actually miss things you'd flag as a designer? Jot notes in `progress.txt`

### Day 2 — Introduce v2, compare
1. Run `npm run eval` (v1 + v2 side-by-side)
2. In `promptfoo view`, compare scores per rubric criterion
3. Judgment: Does v2 feel more useful/actionable? If the rubric disagrees with your read, note it for tuning

### Day 3 — Add real-world case
1. Think of an actual agentic UI flow you're designing/reviewing
2. Add it as a new fixture in `testcases/fixtures.yaml`
3. Re-run `npm run eval` — check if v2 gives actionable critique

### Day 4 — Stress-test rubric
1. Add two fixtures: one low-effort/bad flow, one well-designed flow
2. Run eval — confirm rubric scores them differently (not always passing)
3. Adjust assertion thresholds if needed

### Day 5 — Prompt iteration (v3)
1. Based on 4 days of review, decide on 1–2 concrete prompt changes
2. Draft `prompts/v3-flow-critique.txt`
3. Run `npm run eval` with all 3 versions, compare in `promptfoo view`

### Day 6 — Record metrics ✅ done
1. Export results: `npx promptfoo eval --output results/day6-results.json`
2. Compute pass-rate / avg score per version for this README

### Day 7 — Finalize & ship ✅ done
1. Final `npm run eval` run — `results/day7-results.json`, reproduces Day 6 exactly (v1=0/8, v2=8/8, v3=4/8)
2. Reviewed/finalized this README (final numbers table, Investigation Log, Project Structure, this cadence section)
3. Confirmed `git log --oneline` shows one commit per day of version/rubric iteration (see below)
4. Repo already exists and has been pushed continuously since Day 2 (see Push to GitHub note below)

## Git Workflow

### Initial setup (Day 1)
```bash
git init
git add .
git commit -m "Initial commit: v1 baseline prompt, test fixtures, rubric, promptfoo config"
```

### After v2 improvements (Day 2)
```bash
git add prompts/v2-flow-critique.txt promptfooconfig.yaml
git commit -m "Add v2 improved prompt with structured output and agentic UX heuristics"
```

### After v3 iteration (Day 5)
```bash
git add prompts/v3-flow-critique.txt
git commit -m "Add v3 prompt iteration based on 4-day review cycle"
```

### Push to GitHub (Day 7)
Already done — this repo has a `origin` remote configured and has been pushed to after every day's commit since Day 2. `git remote -v` / `git log --oneline` confirm this. (The commands below are kept for reference/reproducibility if setting this up from scratch elsewhere.)
```bash
git remote add origin https://github.com/YOUR_USERNAME/agentic-ui-flow-critique-eval.git
git branch -M main
git push -u origin main
```

## Notes

- All prompt templates are in `/prompts` with version headers for clarity
- Test fixtures are in `testcases/fixtures.yaml` as YAML vars consumed by promptfoo
- Promptfoo's internal run cache/history lives in `.promptfoo/` (gitignored) — that's separate from `results/`, which holds the canonical exported JSON snapshot(s) and *is* committed (as of Day 6) so the numbers in this README are backed by a real, inspectable file
- Each iteration (v1 → v2 → v3) is a separate git commit, showing the version history required for certification

---

**Questions?** Review the daily testing guide above or check `progress.txt` for iteration notes.
