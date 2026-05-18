# Contract-Driven Development

**Three phases, three artifacts. A workflow for human + AI agent pair programming where the bottleneck is not the agent's speed — it's the human's review time.**

## The Problem

Agents generate anything in seconds. The standard pattern — plan, approve, execute — breaks at "approve." The agent writes a 1000-line plan. You read it, think "this is all wrong," and lose an hour of your life.

**Root cause**: reading someone else's text is hard. Answering a specific question is easy.

## The Solution

The agent doesn't write *to* you. The agent asks *you*. You answer briefly. The agent works autonomously within the boundaries you set.

```
Phase 1: Contract      → 5 questions, 20-line YAML
Phase 2: Map           → 200-300 words, direction check
Phase 3: Implementation → slices, auto-continue on green tests
```

You touch the process three times:
1. Answer 5 questions (2 minutes)
2. Look at a map, say "ok" (15 seconds)
3. Check a red slice (rare)

## Works With

### Claude Code — first-class support

Drop a single file and type `/contract`:

```bash
cp adapters/claude-code/contract.md .claude/commands/contract.md
```

From that point, Claude asks the 5 questions, writes the contract, draws the map, and works in checkpointed slices. Contracts land in `.claude/contracts/`.

### Other agent harnesses

The methodology itself is harness-agnostic. The core artifact is a YAML contract — any agent that can read a file and follow instructions can use it.

| Harness | How to adopt |
|---------|-------------|
| **Hermes Agent** | Native skill: `skill_view(name='contract-driven-development')`. Contracts to `.hermes/contracts/`. |
| **Cursor** | Write the contract manually, add to `.cursorrules`: "Read `.cursor/contracts/current.yaml` before starting. If a decision falls outside boundaries, ask — do not proceed." |
| **Aider** | Add contract path to `read:` in `.aider.conf.yml`. Use `/read-only` for boundary files. |
| **OpenHands / Devin** | Pass the contract as `instructions` in the task config. The agent reads boundaries before generating a plan. |
| **GitHub Copilot** | Include the contract YAML in the issue body. Copilot Workspace reads it as the spec. |
| **Any agent with file access** | Write a contract YAML in the repo. Add a rule: "Read `.contracts/<task>.yaml` before starting." |

### A note on Claude Code vs others

Claude Code has the best fit because its custom slash commands (`/contract`) support interactive multi-turn workflows. The adapter in this repo gives you the full dialogue experience — the agent asks, you answer, the contract forms organically.

For other harnesses that lack custom commands, you pre-write the contract yourself (using the template) and feed it as a constraint. The methodology still holds; the contract still shrinks the approval surface from "1000-line plan" to "20-line YAML."

## Phase 1: Contract

Agent does NOT write a plan. Agent asks 5 structured questions:

| # | Question | Contract field |
|---|----------|---------------|
| 1 | What must NOT be touched? | `boundaries` |
| 2 | What must remain true? | `invariants` |
| 3 | How do we know it's done? | `acceptance` |
| 4 | What libraries/patterns to use? | `style` |
| 5 | What happened and why now? | `context` |

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

## Phase 3: Implementation with Checkpoints

Agent cuts work into slices. After each slice:

```
Slice → tests → green? → auto-continue
                    ↓ no
               Stop. Human looks.
```

Human intervenes only when tests are red or the agent hits a decision not covered by the contract.

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
├── SKILL.md                           ← Native skill for Hermes Agent
├── templates/
│   └── contract.yaml                  ← Blank contract template
├── examples/
│   └── example-contract.yaml          ← Worked example: CSV export feature
└── adapters/
    └── claude-code/
        ├── README.md                  ← Install instructions
        └── contract.md                ← /contract slash command
```

## License

MIT
