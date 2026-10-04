---
name: vibe-test
description: Run Astryx vibeability tests — spawn fresh subagents that generate UI code from the prompt battery using only AGENTS.md, then aggregate scores. Use when asked to vibe-test, measure how well AGENTS.md or the docs help LLMs write correct Astryx code, or compare doc changes against a baseline.
argument-hint: '[count] [--degradation] [--persona naive|experienced|adversarial] [--prompts id,id]'
disable-model-invocation: true
---

# Vibe Test

Measures how well AGENTS.md helps an LLM write correct Astryx component code. Harness: `internal/vibe-tests` (package `@astryxdesign/vibe-tests`). Read `internal/vibe-tests/README.md` "Checker Protocol" if unsure about any rule below.

Arguments: `$ARGUMENTS`

- `[count]` → stratified sample of that many prompts. Omitted → all prompts.
- `--degradation` → 10-turn protocol (see below). Omitted → one-shot.
- Other flags pass straight through to `interactive`: `--persona`, `--target astryx|baseline|html`, `--prompts a,b`, `--test-set <name>`, `--label <text>`, `--skill-doc <path>`, `--holdout`.

## 1. Set up the iteration

```
pnpm -F @astryxdesign/vibe-tests interactive [--sample <count>] [--degradation] [other flags]
```

Note the printed **iteration ID**. This writes `internal/vibe-tests/results/<iteration>/tasks/<promptId>.json` (results/ is gitignored) and regenerates `internal/vibe-tests/AGENTS.md` via `astryx init` if it is over an hour old.

## 2. Generate code with isolated subagents

Spawn **one fresh subagent per prompt**, in parallel (batches of 3–5 if there are many). Working directory for each: `internal/vibe-tests`.

Isolation rules (these keep the test valid — do not relax them):

- Read each task file yourself. Give the subagent **only the `subagentPrompt` field, verbatim**. Do not tell it to read the task file: the file also contains `expectedComponents`, which must never reach the agent.
- No hints, no component names, no context from this conversation, and never reuse a subagent across prompts.
- Do not "help" the subagent with Astryx knowledge. Its only source is AGENTS.md and what it discovers through it.

`subagentPrompt` already tells the subagent where to write `results/<iteration>/results/<promptId>.tsx`. Also have it write `<promptId>.json` next to it:

```json
{
  "id": "<iteration>-<promptId>",
  "timestamp": "<ISO>",
  "model": "claude-code-interactive",
  "persona": "<from task file>",
  "promptCategory": "<from task file>",
  "trajectoryDepth": 0,
  "prompt": "<prompt text>",
  "response": "<the code>",
  "docsRead": ["AGENTS.md", "Button.md"],
  "completedAt": "<new Date().toISOString() at finish>",
  "evaluation": {"success": true, "componentsUsed": [], "escapeHatches": []}
}
```

`docsRead` and `completedAt` are required for cost tracking. Scoring in `aggregate` reads the `.tsx` itself, so the `evaluation` block is the subagent's self-assessment only.

### Degradation mode (`--degradation`)

Each subagent is driven through 10 turns using SendMessage on the same subagent:

- Turn 0: `subagentPrompt` (initial build).
- Turns 1–5 and 7–9: unrelated filler questions (e.g. "How do I center a div?").
- Turn 6: `followUps[0]` from the task file. Turn 10: `followUps[1]`.

Snapshot the code at turns 0, 6, 10. Write the `.tsx` from turn 0, and write `<promptId>.json` as an **array** of entries (same shape as above, one per snapshot, `trajectoryDepth` set to 0/6/10). Caveat: `aggregate` reads `docsRead`/`completedAt` only from a top-level object, so cost data will be empty for array results.

## 3. Screenshots (optional)

`.github/workflows/vibe-screenshots.yml` (`workflow_dispatch`, input `iterations=<iteration>`) builds previews and screenshots in CI. CI checks out the repo, and `results/` is gitignored, so it only sees an iteration that exists on the ref it runs against. Do not commit or force-add results without asking the user. In cloud sessions there is no `gh` CLI; trigger with the `mcp__github__actions_run_trigger` tool. If screenshots aren't needed, skip this step.

## 4. Aggregate

```
pnpm -F @astryxdesign/vibe-tests aggregate --iteration <iteration>
```

Report the summary to the user: success rate by category, escape hatches, cost. Say plainly which prompts failed or had no output. Don't claim an improvement from a single small sample — name the sample size.
