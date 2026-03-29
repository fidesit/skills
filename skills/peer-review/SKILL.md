---
name: peer-review
description: >
  Generate a document with iterative AI peer review. Fan-out to 5 parallel
  reviewer subagents (accuracy, completeness, overlap, probe, temporal),
  revise until all pass or max iterations reached, then present for approval.
  Use when asked to "peer review", "generate and review a document",
  "create a reviewed rulebook", or "write with review loop".
context: fork
---

# Peer-reviewed document generator

You orchestrate a multi-agent review loop. Follow this procedure exactly.

## Inputs

The user provides:
- **document_prompt**: what to generate (topic, format, constraints)
- **sources**: list of authoritative URLs or references to fetch
- **completeness_checklist**: items the document must cover
- **probe_cases**: test scenarios to run against the finished document
- **reviewer_instructions**: (optional) per-reviewer guidance overrides
- **max_iterations**: (optional, default 3) review loop cap

If any required input is missing, ask for it before proceeding.

## Phase 1 — Source fetching

Spawn a subagent:

```
Task (subagent_type: research):
  Read the prompt file: prompts/source-fetcher.md
  Inputs: document_prompt, sources
  Returns: source_digest, changes_since_last_year, fetch_failures
```

If fetch_failures is non-empty, report them to the user and ask whether to
proceed or provide alternative sources.

## Phase 2 — Draft generation

Using the source_digest from Phase 1, generate the document yourself (inline).
Follow the instructions in `prompts/draft-generator.md`.

Store the output as `document` and set `version = "v1.0"`.

## Phase 3 — Parallel review (fan-out)

Spawn **5 subagents in parallel** using the Task tool. Each gets the current
`document` and the `source_digest`. Each returns `issues` (text) and
`verdict` ("PASS" or "FAIL").

```
Task 1 (subagent_type: general-purpose):
  Role: Accuracy reviewer
  Prompt: prompts/reviewer-accuracy.md
  Inputs: document, source_digest
  Optional: reviewer_instructions.accuracy

Task 2 (subagent_type: general-purpose):
  Role: Completeness reviewer
  Prompt: prompts/reviewer-completeness.md
  Inputs: document, completeness_checklist
  Optional: reviewer_instructions.completeness

Task 3 (subagent_type: general-purpose):
  Role: Overlap & precision reviewer
  Prompt: prompts/reviewer-overlap.md
  Inputs: document
  Optional: reviewer_instructions.overlap

Task 4 (subagent_type: general-purpose):
  Role: Classifier probe reviewer
  Prompt: prompts/reviewer-probe.md
  Inputs: document, probe_cases
  Optional: reviewer_instructions.probe

Task 5 (subagent_type: general-purpose):
  Role: Temporal validity reviewer
  Prompt: prompts/reviewer-temporal.md
  Inputs: document, source_digest
  Optional: reviewer_instructions.temporal
```

Wait for all 5 to return before proceeding.

## Phase 4 — Convergence check

Count verdicts:
- If all 5 are PASS → go to Phase 6 (human gate)
- If iteration_count >= max_iterations → go to Phase 6 with a warning
- Otherwise → go to Phase 5

## Phase 5 — Revision

Revise the document inline. Follow `prompts/revision-agent.md`.
Address every issue flagged by the reviewers. Change nothing else.
Increment the version (v1.0 → v1.1 → v1.2 ...).
Increment iteration_count.

Return to Phase 3.

## Phase 6 — Human gate

Present to the user:
1. The final document
2. A summary table of all 5 reviewer verdicts
3. The iteration count
4. If forced exit: list of unresolved issues with a warning

Ask: **"Approve and save, or reject with notes?"**

- **Approve** → save the document to the location the user specified
  (or `./output/<document-name>-<version>.md` by default)
- **Reject** → take the rejection notes, return to Phase 2 with the
  notes injected as additional instructions. This counts as a fresh
  draft, resetting iteration_count.

## Rules

- Never skip a reviewer. All 5 run every iteration.
- Never modify the document outside of Phase 2 or Phase 5.
- Always show the user which reviewers failed and why before revising.
- If the same issue persists across 2+ iterations, flag it explicitly
  to the user — it may require human judgment.
- Keep a running changelog in the document's CHANGELOG section.
