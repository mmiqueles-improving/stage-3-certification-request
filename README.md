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
- **Ollama** installed locally with a model pulled (default: `llama3.1`)

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
│   └── v2-flow-critique.txt          # Improved prompt (structured, heuristic-grounded)
├── testcases/
│   └── fixtures.yaml                 # 5 realistic agentic UI flow scenarios
└── results/                           # Evaluation outputs (generated)
```

## Evaluation Rubric

Each test case is scored on four criteria (`promptfooconfig.yaml` → `defaultTest.assert`):

1. **Actionability** (pass/fail, `llm-rubric` graded by the local Ollama model): Does the critique provide at least 3 concrete, implementable fix suggestions that reference specific UI states/interactions from the flow (not generic advice)?
   - Example concrete: "Add a progress bar showing 'Step 2/5: Cleaning data' with estimated time remaining"
   - Example generic: "Improve the user experience"

2. **Coverage of Agentic UX Heuristics** (pass/fail, deterministic keyword count via `javascript` assertion — requires ≥2 of 5 to pass):
   - Progress/status visibility
   - User control and override capability
   - Error recovery guidance
   - Confirmation before destructive actions
   - Transparency of agent reasoning

   Note: this was originally an `llm-rubric` ("count how many topics are mentioned"), but the local grading model (2GB `llama3.2`) proved unreliable at counting/enumeration tasks. It's now a deterministic keyword match against the fixed, closed set of heuristic names the v2 prompt requires, which is both more accurate and doesn't depend on the local model's reasoning.

3. **Structure/Consistency** (binary): Does the output contain the literal `**Issue**`/`## Issues`, `**Severity**`, and `**Fix Suggestion**`/`**Fix**` markdown markers (all three required)?
   - Checked via 3 separate `regex` assertions. Requiring all three (rather than an OR of loose words like "issue"/"fix") avoids false passes from v1's free-form prose incidentally containing those words.

4. **Conciseness** (binary): No filler phrases
   - Checked via multiple `not-icontains` assertions for phrases that are unambiguously generic in any context (e.g. "improve the experience"). Single words like "user-friendly" were deliberately excluded from this list since they can appear inside an otherwise concrete, specific suggestion.

**Grading provider**: all model-graded (`llm-rubric`) assertions run against the local Ollama model (`ollama:chat:llama3.2`), set via `defaultTest.options.provider`, so the whole suite runs offline with no `OPENAI_API_KEY` required.

## Quantified Improvement Evidence

Measured via `npm run eval` on 2026-08-10 (Ollama `llama3.2`, 5 real-world agentic UI flow fixtures, both prompts graded against the same 4-criteria rubric above):

### Baseline (v1)
- **Pass rate**: 0% (0/5 test cases meet all criteria)
- v1 fails the structure check every time (it was never instructed to use an Issue/Severity/Fix format), and its critiques tend to lean on generic framing ("...to improve the user experience") even when some individual suggestions are reasonable.

### Improved (v2)
- **Pass rate**: 100% (5/5 test cases meet all criteria)
- v2 consistently follows the required structure, cites specific UI states from each flow, and labels each issue with a specific heuristic.

**Improvement**: +100 percentage points in pass rate (0% → 100%) on this rubric and fixture set. Full per-assertion results are in `results/day1-results.json`.

Note: pass/fail here is intentionally strict — it requires meeting every criterion simultaneously (structure AND actionability AND heuristic coverage AND no filler). This is a demonstration rubric across 5 fixtures on a single local model; treat the percentages as directional evidence for this suite, not a universal quality score.

## Version History

- **v1** (baseline): Naive, under-specified prompt with no output structure or persona. Serves as the "before" state.
- **v2** (improved): Structured prompt with defined persona (senior agentic-UX reviewer), explicit output format (Issues / Severity / Fix), constraints (max 5 issues, cite specific states, no fluff), and grounding in agentic UX heuristics.

See `git log --oneline` for iteration history.

## Test Cases

The suite includes 5 realistic agentic UI flow scenarios:

1. **Multi-step processing with no progress indicator** — User uploads CSV, agent processes across 5 steps with only "Processing..." message; no cancel, no error recovery.
2. **Destructive action without confirmation** — Chat agent auto-approves all expenses without summary or confirmation dialog.
3. **Streaming response with no interrupt** — Agent streams long response; user cannot cancel mid-stream if agent goes off-track.
4. **Silent handoff failure** — Agent handoff to human specialist fails silently; no error message, retry, or fallback contact info.
5. **Opaque agent reasoning** — Agent recommends a product with no explanation of reasoning, data sources, or confidence; user cannot override.

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

### Day 6 — Record metrics
1. Export results: `npx promptfoo eval --output results/day6-results.json`
2. Compute pass-rate / avg score per version for this README

### Day 7 — Finalize & ship
1. Final `npm run eval` run
2. Review/finalize this README (purpose, problem, weekly frequency, before/after numbers)
3. Confirm `git log --oneline` shows version iterations
4. Create private GitHub repo, push (instructions below)
5. Email repo link to `Tim.Rayburn@improving.com`, subject: **"Stage 3 Certification Request"**

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
Once you've created a private GitHub repo:
```bash
git remote add origin https://github.com/YOUR_USERNAME/agentic-ui-flow-critique-eval.git
git branch -M main
git push -u origin main
```

Then email the repo link to `Tim.Rayburn@improving.com` with subject **"Stage 3 Certification Request"**.

## Notes

- All prompt templates are in `/prompts` with version headers for clarity
- Test fixtures are in `testcases/fixtures.yaml` as YAML vars consumed by promptfoo
- Evaluation results are cached in `.promptfoo/` (gitignored)
- Each iteration (v1 → v2 → v3) is a separate git commit, showing the version history required for certification

---

**Questions?** Review the daily testing guide above or check `progress.txt` for iteration notes.
