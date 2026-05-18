# Contract-Driven Development for Claude Code

A custom slash command that implements the 3-phase contract-first workflow inside Claude Code.

## Install

```bash
# Copy the command to your project
mkdir -p .claude/commands
cp adapters/claude-code/contract.md .claude/commands/contract.md

# Create contracts directory
mkdir -p .claude/contracts
```

## Usage

```
/contract
```

Claude will start Phase 1: ask 5 structured questions, write a contract, and proceed through the workflow. 

The contract is saved to `.claude/contracts/<slug>.yaml` and can be inspected at any time.

## Workflow

```
/contract                   ← you type this
  ↓
Phase 1: 5 questions        ← Claude asks, you answer
  ↓
Contract YAML               ← Claude writes, you approve
  ↓
Phase 2: Map (200-300w)     ← Claude draws, you confirm direction
  ↓
Phase 3: Slices             ← Claude works, auto-continues on green tests
```
