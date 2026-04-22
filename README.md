# Orchestration Setups for OpenClaw + ACP Workers

Goal: avoid making the main OpenClaw session a heavy execution runtime while keeping it as the control plane.

## Package entrypoint

- `SKILL.md` is the root skill entrypoint for ClawHub / OpenClaw.
- `docs/` contains the design, runtime notes, reproduction guide, and test results.
- `packs/` contains reusable workflow/agent/template assets.
- `examples/` contains minimal session launch patterns.

## Recommendation

**Start here:**
- **OpenClaw main session** = planner, router, durable state, human interface
- **ACP Claude Code sessions** = default execution workers
- **ACP Codex sessions** = optional secondary builder / reviewer / fixer workers
- **Artifacts + handoffs** = file-native state under shared artifact directories and run-state directories

This keeps the fragile part small: the main session decides and tracks, but the real long coding work happens in external ACP workers.

## Why change the setup

The current issue is not that every worker gets Active Memory.
The issue is that the **main OpenClaw orchestrator session** can accumulate runtime overhead on user-triggered prompt build hooks, while it is also the place that calls `sessions_spawn`, `sessions_list`, and other orchestration tools.

So the safer design is:
- thin control plane in OpenClaw
- heavy execution in ACP workers
- minimal plugin dependence on the orchestrator session

## Setup matrix

| Setup | What it is | Pros | Cons | Use for |
|---|---|---|---|---|
| **A. OpenClaw-only** | Main session + OpenClaw subagents everywhere | Unified tools, simple mental model | More coupling to gateway/session internals | Short multi-step tasks |
| **B. OpenClaw + ACP Claude Code** | OpenClaw orchestrates, Claude Code executes | Best default for real coding projects, persistent sessions, less pressure on main lane | Slightly more moving parts | Main recommendation |
| **C. OpenClaw + ACP Codex** | Same as B but Codex workers | Strong code search/edit/test loops, good cost profile | Slightly less strong on ambiguous planning/review prose | Batch edits, parallel fixes |
| **D. Hybrid ACP** | Claude Code planners/reviewers + Codex builders | Strongest team pattern for medium/large repos | More orchestration discipline needed | Larger engineering work |
| **E. External-first swarm** | Very thin OpenClaw, most work done outside it | Maximum isolation from gateway quirks | Less unified state unless carefully documented | Big coding swarms, later phase |

## Final recommendation

### Phase 1, adopt now
**Setup B: OpenClaw + ACP Claude Code**

Pattern:
1. Keep OpenClaw as the control plane
2. Spawn Claude Code ACP sessions for actual implementation
3. Use one persistent thread/session per workstream
4. Store state in files, not in session memory
5. Keep the main session lightweight during orchestration

### Phase 2, when needed
**Setup D: Hybrid ACP**
- Claude Code for planning, architecture, nuanced review
- Codex for focused implementation, patching, repetitive batch work

## Control plane rules

1. **OpenClaw should not be the main execution engine** for long coding swarms.
2. **Every workstream gets a file-native state path**:
   - task/spec file
   - shared artifacts directory
   - run-state directory for handoffs, checkpoints, and reviews
3. **ACP workers own code changes**.
4. **OpenClaw owns routing, checkpoints, retries, and reporting**.
5. **Git worktrees per builder stream** when multiple workers touch code.

## Minimal operating model

- **Planner**: OpenClaw main or Claude Code ACP
- **Builder**: Claude Code ACP session
- **Reviewer**: Claude Code ACP session or Codex ACP session
- **Fixer**: Codex ACP session for targeted repair loops
- **Human checkpoint**: OpenClaw main session

## Suggested first implementation

- Root skill entrypoint: `SKILL.md`
- Handoff/state kept in a file-native orchestration runtime
- Claude Code ACP as the default worker backend
- Previous architecture material bundled directly under `docs/` and `packs/` so the historical design base stays attached to the new proposal

## Bundled previous work

This folder now includes the prior material we had already developed:
- V1 control-plane spec
- OMC/Gastown audit
- runtime docs (`README`, `SCHEMAS`, `COMMUNICATION`, `OPENCLAW_LAUNCHER`, `PROJECT_BUILDER_WORKTREE`)
- agent / workflow / template packs

See:
- `SKILL.md`
- `docs/setup-matrix.md`
- `docs/design/`
- `docs/runtime/`
- `docs/reproduction.md`
- `docs/test-results.md`
- `packs/`
- `examples/session-patterns.md`
