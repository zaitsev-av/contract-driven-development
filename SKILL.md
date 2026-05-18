---
name: contract-driven-development
description: "3-phase contract-first workflow for human and AI agent pair programming. Pass a task description as argument: /contract-driven-development <task>. Phase 0: agent reads the task, explores relevant code, and asks only the questions it cannot answer from context. Phase 1: targeted questions → YAML contract. Phase 2: schematic map (200-300 words). Phase 3: autonomous implementation in checkpointed slices with auto-continue on green tests."
license: MIT
compatibility: "Designed for Claude Code, Hermes Agent, Cursor, OpenCode, Aider, and any agent that reads SKILL.md files. Requires file system access to write contract YAML."
metadata:
  author: zaitsev-av
  version: "1.1.0"
---

# Contract-Driven Development

Use this skill when the user wants to build a feature, fix a bug, or refactor code through a contract-first workflow. The philosophy: **the bottleneck is not agent speed but human review time**. This skill inverts the typical plan→approve→execute pattern: the agent asks short questions instead of writing long plans, and the human answers briefly instead of reading walls of text.

## Invocation

The skill accepts a task description as argument:

```
/contract-driven-development <task description>
```

If no argument is given, ask the user to describe the task in one sentence before proceeding.

## When to use

- User explicitly requests contract-driven development, or a "contract", "карта", or "слайсы"
- User says "let's do it the way we discussed" referring to the 3-phase workflow
- User pushes back on a verbose plan and wants a compact alternative
- Task is scoped enough to be bounded by a contract (not "rewrite the entire backend")

Not suitable for: trivial one-liners, tasks where boundaries are obvious, or multi-month epics that need decomposition first.

---

## Phase 0: Task & Context Analysis (silent)

**Before asking any questions**, the agent reads the task and explores the codebase. This phase is silent — no output to the user until Phase 1 begins.

### Steps

1. **Parse the task** — extract the core intent: what needs to change, what domain it touches (transport, service, storage, config, etc.).

2. **Explore relevant code** — read the files most likely affected. Use grep/find to locate interfaces, handlers, types, and tests related to the task. Do NOT read the entire repo — read what the task points at.

3. **Pre-fill contract fields from context** — for each of the 5 contract dimensions, determine what can be derived from code vs. what is genuinely ambiguous:
   - **Boundaries**: Can be partially inferred from the affected layer (e.g., generated files are always off-limits).
   - **Invariants**: Often derivable from existing tests and linter config.
   - **Acceptance**: Almost always needs human input — the agent can propose, not decide.
   - **Style**: Usually derivable from existing patterns in the same package.
   - **Context**: Provided by the task description itself.

4. **Identify gaps** — list only the dimensions where the agent cannot make a confident assumption. These become the questions in Phase 1.

### Rule: ask only what you don't know

If the agent can derive a confident answer from code, **do not ask** — state the assumed answer in the contract and ask for confirmation only if it's load-bearing (i.e., wrong assumption would cause significant rework).

There is no hard cap on the number of questions. Ask exactly as many as Phase 0 left open — no more, no less. Before adding a question, apply the test: «Can I derive this from the code?» If yes — don't ask. If all dimensions are covered by code, skip to contract draft and say «Я вывел контракт из кода — проверь.»

---

## Phase 1: Contract

**Goal**: produce a machine-readable YAML contract of ~20-30 lines. Agent asks only the questions Phase 0 couldn't answer. Agent does NOT write a plan.

### The 5 Contract Dimensions

These are the dimensions to fill — ask about only the ones Phase 0 left open:

1. **Boundaries** — What must not be touched? Which files, modules, packages, or APIs must stay unchanged?
   - Maps to `boundaries` in the contract. The most important dimension.
   - Often partially known from codebase: generated files, vendor/, transport layer, mocks.

2. **Invariants** — What must remain true after the change?
   - Examples: backwards compatibility, performance thresholds, existing test suites must stay green.
   - Often derivable from linter config, existing test coverage targets, interface contracts.

3. **Acceptance** — How do we know it's done? A concrete scenario: who does what and sees what.
   - 1-2 user-centric scenarios. NOT "implement feature X".
   - Usually needs human input unless the task description is already acceptance-shaped.

4. **Style and Patterns** — Which libraries, patterns, or code examples to follow?
   - Usually derivable from the same package: if existing handlers use pattern X, new ones should too.
   - Ask only if the task crosses into a new area with no existing examples.

5. **Context** — One sentence: what happened and why now?
   - Usually provided by the task description. Skip the question if the task already answers it.

### How to ask

- **Batch by default**: present all open questions at once, numbered. Don't drip one at a time unless the user is clearly chatty.
- **For each question, show what you already know**: «Я вижу, что X — это верно? И ещё один вопрос: Y»
- This lets the human correct wrong assumptions AND answer open questions in one reply.

### After answers: confirm understanding

Before writing the contract, replay your understanding in 2-3 sentences: «Я понял так: нужно сделать X, не трогая Y, результат проверяется по сценарию Z. Верно?»

### Write the contract

Save the contract to a `contracts/` directory within your harness's artifact path: `.hermes/contracts/<slug>.yaml`, `.claude/contracts/<slug>.yaml`, or equivalent. Template in `templates/contract.yaml`. Show it to the user and ask for approval. Human says «ок» or corrects.

The contract is the **only** document the human approves before work starts. 20-30 lines, not 1000.

---

## Phase 2: Map

**Goal**: agent draws a schematic map. 200-300 words. Twitter format. Human checks direction, not details.

### What the map contains

- **Files touched** — which files will be created/modified. One line per file.
- **Data flow** — arrows: where data enters, how it moves, where it lands.
- **Test strategy** — what tests will be written and where.
- **Work order** — numbered sequence of steps (5-10 steps).

### Format

ASCII or compact text. No markdown tables. Use indentation and arrows:

```
Map: <one-line summary>

Files:
  src/export/csv.go        — modify: add ExportToCSV method
  src/handlers/reports.go  — modify: add GET /reports/export
  tests/export_test.go     — new: 12 test cases

Flow:
  HTTP GET → handler → CSV builder → file download

Tests:
  unit: ExportToCSV edge cases (empty, unicode, large)
  integration: handler returns 200 + valid CSV

Order:
  1. ExportToCSV method
  2. Handler endpoint
  3. Tests
```

### Human check

Present the map. Human says «да, направление верное» or corrects. If the map is wrong, redo it — no need to redo the contract. The contract is boundaries; the map is trajectory.

If the human says «no», ask ONE clarifying question, redraw, re-present.

**The map is a hypothesis, not a promise.** During implementation, discovering an additional file is normal — the agent adds it to the map silently and continues, as long as the file is not in `boundaries`. The map is considered wrong only if the overall direction changes (different layer, different approach), not when a new file appears.

---

## Phase 3: Implementation with Checkpoints

**Goal**: agent works autonomously in slices. Auto-continue on green. Stop on red.

### Slice rules

1. **Slice size**: one atomic change. A slice is atomic if: (a) it can be described in one verb phrase («добавить метод X», «написать тест для Y»); (b) it compiles on its own; (c) it has exactly one test target. Not "write all tests" — that's multiple slices. Not "implement the feature" — that's the whole map.
2. **After each slice**: run the relevant tests for that slice.
3. **Green → auto-continue**: agent moves to the next slice without asking. Human receives a short progress note: «Slice 2/7 done: ExportToCSV method. Tests green. Continuing.»
4. **Red → distinguish before stopping**:
   - **TDD-red** (test written, implementation not yet): expected. Auto-continue to the implementation slice — do not stop.
   - **Regression-red** (test was green before this slice, now fails): diagnose first, then present a stop card. Do not ask the human to diagnose for you.
   - If unclear which kind: treat as regression-red and stop.
5. **Uncovered decision → stop**: if agent hits a decision not covered by the contract (e.g., must touch a file in `boundaries`), it stops and asks ONE question. The answer is appended to the contract.

### Stop card format

When a regression-red is detected, the agent diagnoses the failure first, then presents a stop card in this exact format:

```
STOP — Regression: <test name>

Problem:  <one sentence — what failed and what was expected>
Cause:    <what in this slice triggered it — be specific>
Fix:      <concrete proposed fix>

Next:
  A) Apply proposed fix → continue
  B) Roll back slice N, try differently: <alternative approach>
  C) Amend contract: <which assumption turned out wrong>
```

Rules for the stop card:
- **Diagnose before showing the card.** Read the failure, trace it to the slice. Don't ask the human to diagnose.
- **Option C appears only if** the root cause is a wrong contract assumption, not just a bug in the implementation.
- **The human picks a letter or gives a free-form instruction.** Either way, the agent executes and continues.
- After the human responds, append the resolution to the contract YAML if option C was chosen.

### Progress notifications

Keep them minimal. The human shouldn't feel spammed:

```
Slice 3/7: handler endpoint. Tests green. → next slice.
```

Not:
```
I have successfully implemented the handler endpoint with full error handling,
logging, and input validation. The tests are passing...
```

### Final delivery

When all slices are green, present:

1. **What changed** — file list, one line per file with reason.
2. **Proof** — test output (green summary, not full logs).
3. **One decision to highlight** — if applicable: «I did X because Y. Keep it in mind.»

---

## Special cases

**Task too big for one contract**: decompose first. Each sub-task gets its own micro-contract. The parent contract defines the decomposition rules.

**Human is unavailable during execution**: slices auto-continue on green. Red slices queue up. When the human returns, they see stacked reds and address them in order.

**Contract needs amending mid-flight**: two triggers — (1) regression-red where option C is chosen from the stop card, or (2) agent hits a decision not covered by the contract. In both cases: agent stops, presents the gap in one sentence, waits for the human's answer, appends it to the contract YAML, resumes. The contract grows organically.

---

## Pitfalls

- **Don't skip the "confirm understanding" step**. Answers can be interpreted differently than intended. The 2-3 sentence replay catches misalignment before it becomes code.
- **Don't make the map too detailed**. If it looks like a plan, it's wrong. 200-300 words. If you need more, the contract is probably too broad — suggest decomposition.
- **Don't spam progress**. One line per slice. Not "I'm now thinking about..."
- **Red slices are not failures**. They're expected. They mean the contract or map had a gap. Treat the human's fix as contract amendment, not as a bug in the agent.
- **If the same boundary question comes up twice**, the contract is too vague. Propose tightening it.
- **Never proceed past a red slice** without human input. Red means "the contract's assumptions were wrong" and only the human can resolve that.
