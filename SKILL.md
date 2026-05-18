---
name: contract-driven-development
description: "Develop features through a 3-phase contract-first workflow. Phase 1: 5 structured questions form a compact YAML contract. Phase 2: agent draws a map (200-300 words, no implementation). Phase 3: autonomous slices with auto-continue on green tests, human interrupt on red. Minimises human reading time -- the human answers questions, the agent writes code."
tags: [workflow, planning, autonomous, contract, tdd]
---

# Contract-Driven Development

Use this skill when the user wants to build a feature, fix a bug, or refactor code through a contract-first workflow. The philosophy: **the bottleneck is not agent speed but human review time**. This skill inverts the typical plan→approve→execute pattern: the agent asks short questions instead of writing long plans, and the human answers briefly instead of reading walls of text.

## When to use

- User explicitly requests contract-driven development, or a "contract", "карта", or "слайсы"
- User says "let's do it the way we discussed" referring to the 3-phase workflow
- User pushes back on a verbose plan and wants a compact alternative
- Task is scoped enough for 5 questions to bound it (not "rewrite the entire backend")

Not suitable for: trivial one-liners, tasks where boundaries are obvious, or multi-month epics that need decomposition first.

---

## Phase 1: Contract

**Goal**: produce a machine-readable YAML contract of ~20-30 lines. Human answers 5 structured questions. Agent does NOT write a plan.

### The 5 Questions

Ask these one at a time or as a batch — read the room. If the user is chatty, ask one at a time. If they're impatient, batch them.

1. **Boundaries** — «Что нельзя трогать? Какие файлы, модули, пакеты, API должны остаться нетронутыми?»
   - Maps to `boundaries` in the contract. The most important question.

2. **Invariants** — «Что должно остаться правдой после изменений? Какие свойства системы нельзя нарушить?»
   - Examples: backwards compatibility, performance thresholds, existing test suites must stay green. Maps to `invariants`.

3. **Acceptance** — «Как мы поймём что готово? Конкретный сценарий: кто, что делает, что видит.»
   - 1-2 scenarios, user-centric. NOT "implement feature X". Maps to `acceptance`.

4. **Style and Patterns** — «Какие библиотеки, паттерны, примеры из кодовой базы использовать? Есть ли код, на который ориентироваться?»
   - Maps to `style`. Can be empty if the user doesn't care.

5. **Context** — «Одним предложением — что случилось и почему сейчас?»
   - Maps to `context`. One sentence.

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

---

## Phase 3: Implementation with Checkpoints

**Goal**: agent works autonomously in slices. Auto-continue on green. Stop on red.

### Slice rules

1. **Slice size**: one atomic change. One file, one function, one concern. Not "write all tests" — that's multiple slices.
2. **After each slice**: run the relevant tests for that slice.
3. **Green → auto-continue**: agent moves to the next slice without asking. Human receives a short progress note: «Slice 2/7 done: ExportToCSV method. Tests green. Continuing.»
4. **Red → stop**: agent presents the failure, waits for human decision.
5. **Uncovered decision → stop**: if agent hits a decision not covered by the contract (e.g., must touch a file in `boundaries`), it stops and asks ONE question. The answer is appended to the contract.

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

**Contract needs amending mid-flight**: agent stops at the uncovered decision, asks, appends answer to contract YAML, resumes. The contract grows organically.

---

## Pitfalls

- **Don't skip the "confirm understanding" step**. Five answers can be interpreted differently than intended. The 2-3 sentence replay catches misalignment before it becomes code.
- **Don't make the map too detailed**. If it looks like a plan, it's wrong. 200-300 words. If you need more, the contract is probably too broad — suggest decomposition.
- **Don't spam progress**. One line per slice. Not "I'm now thinking about..."
- **Red slices are not failures**. They're expected. They mean the contract or map had a gap. Treat the human's fix as contract amendment, not as a bug in the agent.
- **If the same boundary question comes up twice**, the contract is too vague. Propose tightening it.
- **Never proceed past a red slice** without human input. Red means "the contract's assumptions were wrong" and only the human can resolve that.
