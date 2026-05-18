# Contract-Driven Development

**Three phases, three artifacts. A workflow for human + AI agent pair programming where the bottleneck is not the agent's speed — it's the human's review time.**

## The Problem

Agents generate anything in seconds. The standard pattern — plan, approve, execute — breaks at "approve." The agent writes a 1000-line plan. You read it, think "this is all wrong," and lose an hour of your life.

**Root cause**: reading someone else's text is hard. Answering a specific question is easy.

## The Solution

The agent doesn't write *to* you. The agent asks *you*. You answer briefly. The agent works autonomously within the boundaries you set.

```
Phase 0: Analysis      → agent reads task + explores code (silent)
Phase 1: Contract      → targeted questions → 20-line YAML
Phase 2: Map           → 200-300 words, direction check
Phase 3: Implementation → slices, auto-continue on green tests
```

You touch the process three times:
1. Answer questions derived from code analysis (1-2 minutes)
2. Look at a map, say "ok" (15 seconds)
3. Check a red slice (rare)

## Works With

One skill file, every harness. Copy `SKILL.md` to the right place:

| Harness | Where to put it | How to invoke |
|---------|----------------|---------------|
| **Claude Code** | `.claude/commands/contract.md` | `/contract` |
| **Hermes Agent** | `~/.hermes/skills/software-development/contract-driven-development/SKILL.md` | `skill_view(name='contract-driven-development')` |
| **Cursor** | `.cursor/rules/contract-driven.md` | Loaded automatically per task |
| **OpenCode** | `.opencode/skills/contract-driven.md` | Loaded automatically |
| **Aider** | `.aider/commands/contract.md` | `/contract` |

Contracts land in `.hermes/contracts/`, `.claude/contracts/`, or wherever the harness stores task artifacts — the path is configured in the skill itself.

The contract YAML is the portable artifact. Even if you switch harnesses mid-project, the contract stays valid.

## Phase 0: Analysis (silent)

Before asking anything, the agent reads the task and explores relevant code. For each contract dimension it checks: can this be derived from the codebase? If yes — it fills the field itself and doesn't ask. If no — it adds a question to Phase 1.

## Phase 1: Contract

Agent does NOT write a plan. Agent asks only what Phase 0 couldn't answer — no fixed number, no ritual questions. There are 5 contract dimensions:

| Dimension | Question | Contract field |
|-----------|----------|---------------|
| Boundaries | What must NOT be touched? | `boundaries` |
| Invariants | What must remain true? | `invariants` |
| Acceptance | How do we know it's done? | `acceptance` |
| Style | What libraries/patterns to use? | `style` |
| Context | What happened and why now? | `context` |

Questions the agent can answer from code are skipped. If all dimensions are clear from context, the agent goes straight to the contract draft.

Output: a 20-30 line YAML file. The human approves the contract, not a plan.

## Phase 2: Map

Agent draws a schematic map from the contract. 200-300 words. Twitter format:

```
Map: Add CSV export for reports

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

Human checks one thing: is the direction right?

**The map is a hypothesis.** New files discovered during implementation are added silently — no stop, no re-approval — as long as they don't touch `boundaries`. The map is wrong only if the approach itself changes.

## Phase 3: Implementation with Checkpoints

Agent cuts work into atomic slices (one verb phrase, compiles alone, one test target). After each slice:

```
Slice → tests → green?           → auto-continue
                    ↓ no
            TDD-red (expected)?  → auto-continue to impl slice
                    ↓ no
            Regression-red
                    ↓
            Agent diagnoses root cause
                    ↓
            Stop card → human picks A / B / C
                    ↓
            Agent executes → continue
```

Stop card format:
```
STOP — Regression: <test name>

Problem:  <what failed>
Cause:    <what in this slice triggered it>
Fix:      <proposed fix>

Next:
  A) Apply proposed fix → continue
  B) Roll back slice N, try differently: <alternative>
  C) Amend contract: <wrong assumption>
```

Human intervenes only on regression failures or decisions not covered by the contract.

## Why This Works

| Standard approach | Contract-Driven |
|---|---|
| Agent writes a plan (easy for agent) | Agent asks questions (easy for agent) |
| Human reads a plan (hard for human) | Human answers questions (easy for human) |
| Agent executes (easy for agent) | Agent executes within boundaries (easy for agent) |

The asymmetry is flipped in the human's favor.

## Closest Analogues

- **Shape Up (Basecamp)**: pitch → scope → build. Compact documents, explicit boundaries.
- **TDD**: tests as contract, red-green-refactor as checkpoints.
- **Dependabot/Renovate**: auto-merge on green CI, human intervenes only on red.

Contract-Driven Development brings these ideas into an agentic workflow.

## Repository Structure

```
contract-driven-development/
├── README.md                          ← English (this file)
├── README.ru.md                       ← Russian
├── LICENSE                            ← MIT
├── SKILL.md                           ← Universal skill — copy to any harness
├── templates/
│   └── contract.yaml                  ← Blank contract template
└── examples/
    └── example-contract.yaml          ← Worked example: CSV export feature
```

## License

MIT
