# Multi-Agent Setup for Claude Code

A set of eight specialized Claude Code subagents that split software work into planning, implementation, debugging, research, and adversarial review, plus a `CLAUDE.md` template with shared development standards.

## Agents

| Agent | Model | Tools | Use it for |
|---|---|---|---|
| `orchestrator` | Opus | Read, Grep, Glob, Agent | Breaking a large task into subtasks and delegating to the other agents. It has no write tools. |
| `architect` | Opus | Read, Grep, Glob, Edit, Write | Planning before implementation. It writes a todo-style plan file and does not implement it. |
| `code` | Sonnet | Read, Grep, Glob, Edit, Write, Bash, PowerShell | Implementing features, fixes, and refactors. |
| `debug` | Sonnet | Read, Grep, Glob, Edit, Write, Bash, PowerShell | Diagnosing failures. It lists candidate causes, instruments to test them, and confirms the diagnosis with you before fixing. |
| `ask` | Sonnet | Read, Grep, Glob, WebFetch, WebSearch | Read-only explanations and answers. It makes no changes. |
| `literature-researcher` | Sonnet | Read, Grep, Glob, Edit, Write, WebFetch, WebSearch, Bash, PowerShell | Source-verified surveys of papers, algorithms, and reference code, run before a design is finalized. Unverifiable claims are labeled "⚠️ Unverified". |
| `red-team-code` | Opus | Read, Grep, Glob, Write, Bash, PowerShell | Auditing code against an independently stated purpose, or pressure-testing a plan for dead ends. Findings only. |
| `red-team-research` | Sonnet | Read, Grep, Glob, WebFetch, WebSearch, Bash, PowerShell | Auditing a paper, RFC, design doc, or benchmark writeup, including the output of `literature-researcher`. Claims are classified Verified, Unverified, Contradicted, or Unfalsifiable. Critique only. |

## Workflow

```mermaid
flowchart LR
    O[orchestrator]

    subgraph RL[Research loop]
        direction LR
        L[literature-researcher]
        RR[red-team-research]
        L -->|research report| RR
        RR -->|claim classifications| L
    end

    subgraph BL[Build loop]
        direction LR
        A[architect]
        RP[red-team-code: plan review]
        C[code]
        RC[red-team-code: code review]
        A -->|plan| RP
        RP -->|approved plan| C
        C -->|diff| RC
    end

    O <-->|starts research, receives validated findings| RL
    O <-->|starts each step, relays findings| BL
    O -.->|debugging subtask or runtime error| D[debug]
```

Only the orchestrator starts agents, and it drives both loops. In the diagram, the orchestrator connects to each loop as a whole, and the arrows inside a loop show the order of hand-offs. Each of those hand-offs is relayed by the orchestrator:

- **Research loop (`literature-researcher` ⇄ `red-team-research`).** The orchestrator starts `literature-researcher`, either on its own initiative or because `architect` listed an open research item in its plan. As soon as the report arrives, `red-team-research` audits its claims, and its Verified/Unverified/Contradicted/Unfalsifiable classifications go back to `literature-researcher` to correct or re-verify. The loop repeats until the stop condition below is met, and only then are the findings used in planning. If the research came from an open research item, the orchestrator sends the plan back to `architect` with the validated findings.
- **Build loop (`architect` → `red-team-code` → `code` ⇄ `red-team-code`).** `architect` turns the task into a plan and hands it back. The orchestrator has `red-team-code` pressure-test the plan, returns findings to `architect` for revision, and once the plan is approved has `code` implement it. `red-team-code` then reviews the result, and `code` revises against its findings. `architect` asks you its clarifying questions and gets your approval on the plan directly.

### Research loop stop condition

Literature review is token-intensive, so the loop spends fully on the first round and narrows afterward:

- **Round 1:** full-breadth survey and full audit.
- **Rounds 2+:** `literature-researcher` re-verifies only the flagged claims, and `red-team-research` audits only the revised and newly introduced claims.

The orchestrator stops the loop when all of these hold:
- No claim is Contradicted.
- Every load-bearing claim is Verified, or is Unverified with a documented reason verification is infeasible (e.g. paywalled paper, dead link). Those are passed on as stated uncertainty and not retried.
- Only peripheral findings remain. They are recorded and passed on without triggering another round.

Two guardrails stop the loop regardless:
- **Round cap of 3** (one full round plus two targeted rounds). Unresolved load-bearing claims are returned labeled ⚠️ and reported to you.
- **No-progress exit.** A round that resolves none of the previously flagged claims ends the loop.

`literature-researcher` must list any claim it removes as "withdrawn".

Both loops are adversarial by design: the producing agent (`literature-researcher`, `code`) and the critic (`red-team-research`, `red-team-code`) refine each other, and the critics never write the fix. `debug` is called by the orchestrator outside the loops, for debugging subtasks and runtime errors. `ask` is for you to use directly; the orchestrator does not call it.

Design rules baked into the prompts:
- **Review agents give findings, not fixes.** `red-team-research` never proposes remediation, and `red-team-code` proposes none on plans or behavioral gaps. The only exception is a one-sentence comparative note on bloat findings.
- **Purpose comes from independent sources.** `red-team-code` takes the spec from a ticket, PR description, or your statement, never from the code's own comments.
- **Planners don't implement or spawn.** `architect` hands its plan back to the orchestrator, which decides what runs next. It lists unverified techniques as open research items and does not research them itself.
- **Debugging confirms before fixing.** `debug` asks you to confirm the diagnosis before it changes anything.
- **Null results are valid.** Reviewers and the orchestrator report "no findings" or "no change needed" plainly. A finding requires evidence of actual failure or harm, not a plausible hypothetical.
- **Producers stay in scope.** `code` implements only the approved plan, `debug` fixes only the reported failure, and `ask` answers only the question. Each reports adjacent observations instead of acting on them. `architect` asks you before adding discovered requirements.
- **Loops are bounded.** Both loops have round caps and a no-progress exit.

## Install

1. Copy the agent files:
   - **Global (all projects):** `agents/*.md` → `~/.claude/agents/`
   - **Single project:** `agents/*.md` → `<project>/.claude/agents/`
2. Merge `CLAUDE.template.md` into your `~/.claude/CLAUDE.md` (global) or `<project>/CLAUDE.md`.

The git-reserved section of the template is commented out because it is a personal preference. Uncomment it if you want agents to leave all git operations to you.

## Caveats

- **Write restrictions are stated in the prompts.** `red-team-code` is told to write only under `tests_scratch/`, and `red-team-research` is told never to edit files. Both have Bash in their `tools:` list.
- **The orchestrator hardcodes agent names.** If you rename or remove an agent, update the delegation sections in `orchestrator.md`, plus the cross-references in `literature-researcher.md`, `red-team-code.md`, and `red-team-research.md`.
- **Path conventions are assumed.** `architect` writes plans to `/plans` if it exists, `literature-researcher` writes reports to `docs/research/`, and `red-team-code` writes probes to `tests_scratch/`.
