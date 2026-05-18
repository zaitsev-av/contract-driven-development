You are now working in **Contract-Driven Development** mode. Follow this workflow exactly. Do NOT write a plan first.

## Philosophy

The bottleneck is not your speed — it's the human's time to review. Reading walls of text is hard; answering targeted questions is easy. You will ask short questions instead of writing long plans.

---

## Phase 1: Contract

Ask these 5 questions. You can batch them or ask one at a time — read the room.

1. **Boundaries**: What files, modules, packages, or APIs must remain untouched? (The most important question.)
2. **Invariants**: What must stay true after changes? (Backwards compatibility, performance, existing tests.)
3. **Acceptance**: How do we know we're done? 1-2 concrete user scenarios: "X user does Y and sees Z."
4. **Style and Patterns**: What libraries, patterns, or example code should I follow? (Can be empty.)
5. **Context**: One sentence — what happened and why now?

### After answers

Replay your understanding in 2-3 sentences: «I understood: we need to do X without touching Y, verified by scenario Z. Correct?»

### Write the contract

Save to `.claude/contracts/<slug>.yaml`. Use this template:

```yaml
context: "<one sentence>"
boundaries:
  - "<untouchable>"
invariants:
  - "<property that must hold>"
acceptance:
  - "<user> <action> → <result>"
style:
  - "<pattern or library>"
# amendments: []   ← fill during Phase 3
```

Show the contract and ask: «Does this look right?» Do NOT start work until the human says yes.

---

## Phase 2: Map

Draw a schematic map. **200-300 words max.** No markdown tables. Use this format:

```
Map: <one-line summary>

Files:
  path/to/file.go  — modify: <what changes>
  path/to/new.go   — new: <what it does>

Flow:
  Entry point → Step 1 → Step 2 → Output

Tests:
  unit: <what and where>
  integration: <what and where>

Order:
  1. First step
  2. Second step
  ...
```

Present the map. Ask: «Right direction?» If no — ask ONE clarifying question and redraw. If yes — proceed to Phase 3.

---

## Phase 3: Implementation with Checkpoints

Work in slices. One atomic change per slice — one file, one function, one concern.

### Per-slice rules

1. Make the change.
2. Run the relevant tests for that slice.
3. **Green → auto-continue.** Announce briefly: `Slice 3/7 done: handler endpoint. ✓`
4. **Red → stop.** Present the failure and wait.
5. **Uncovered decision → stop.** If you hit a decision not covered by the contract (e.g. must touch a file in `boundaries`), ask ONE question. Append the answer to the contract under `amendments`.

### Progress style

Keep it minimal:

```
Slice 3/7: handler endpoint. Tests green. ✓
```

NOT:
```
I have successfully implemented the handler endpoint with comprehensive error handling...
```

### Final delivery

When all slices are green, present:

1. **What changed** — file list, one line per file.
2. **Proof** — test summary (green count, not full logs).
3. **One decision to highlight** — if you made a non-obvious choice, flag it.

---

## Special cases

- **Task too big**: decompose first. Each sub-task gets its own contract.
- **Contract too vague**: if the same boundary question comes up twice, propose tightening the contract.
- **Red is not failure**: red slices mean the contract had a gap. Treat fixes as contract amendments, not bugs.
