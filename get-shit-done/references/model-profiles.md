# Model Profiles

Model profiles control which Claude model each GSD agent uses. This allows balancing quality vs token spend.

## Profile Definitions

| Agent | `quality` | `balanced` | `budget` |
|-------|-----------|------------|----------|
| gsd-planner | opus | opus | sonnet |
| gsd-roadmapper | opus | sonnet | sonnet |
| gsd-executor | opus | sonnet | sonnet |
| gsd-phase-researcher | opus | sonnet | haiku |
| gsd-project-researcher | opus | sonnet | haiku |
| gsd-research-synthesizer | sonnet | sonnet | haiku |
| gsd-debugger | opus | sonnet | sonnet |
| gsd-codebase-mapper | sonnet | haiku | haiku |
| gsd-verifier | sonnet | sonnet | haiku |
| gsd-plan-checker | sonnet | sonnet | haiku |
| gsd-integration-checker | sonnet | sonnet | haiku |

## Profile Philosophy

**quality** - Maximum reasoning power
- Opus for all decision-making agents
- Sonnet for read-only verification
- Use when: quota available, critical architecture work

**balanced** (default) - Smart allocation
- Opus only for planning (where architecture decisions happen)
- Sonnet for execution and research (follows explicit instructions)
- Sonnet for verification (needs reasoning, not just pattern matching)
- Use when: normal development, good balance of quality and cost

**budget** - Minimal Opus usage
- Sonnet for anything that writes code
- Haiku for research and verification
- Use when: conserving quota, high-volume work, less critical phases

## Resolution Logic

Orchestrators resolve model before spawning:

```
1. Read .planning/config.json
2. Check model_overrides for agent-specific override
3. If no override, look up agent in profile table
4. Pass model parameter to Task call
```

## Per-Agent Overrides

Override specific agents without changing the entire profile:

```json
{
  "model_profile": "balanced",
  "model_overrides": {
    "gsd-executor": "opus",
    "gsd-planner": "haiku"
  }
}
```

Overrides take precedence over the profile. Valid values: `opus`, `sonnet`, `haiku`.

## Switching Profiles

Runtime: `/gsd:set-profile <profile>`

Per-project default: Set in `.planning/config.json`:
```json
{
  "model_profile": "balanced"
}
```

## Design Rationale

**Why Opus for gsd-planner?**
Planning involves architecture decisions, goal decomposition, and task design. This is where model quality has the highest impact.

**Why Sonnet for gsd-executor?**
Executors follow explicit PLAN.md instructions. The plan already contains the reasoning; execution is implementation.

**Why Sonnet (not Haiku) for verifiers in balanced?**
Verification requires goal-backward reasoning - checking if code *delivers* what the phase promised, not just pattern matching. Sonnet handles this well; Haiku may miss subtle gaps.

**Why Haiku for gsd-codebase-mapper?**
Read-only exploration and pattern extraction. No reasoning required, just structured output from file contents.

**Why `inherit` instead of passing `opus` directly?**
Claude Code's `"opus"` alias maps to a specific model version. Organizations may block older opus versions while allowing newer ones. GSD returns `"inherit"` for opus-tier agents, causing them to use whatever opus version the user has configured in their session. This avoids version conflicts and silent fallbacks to Sonnet.

## Custom Model Profiles (Ollama, GLM, Minimax, etc.)

You can define custom model profiles for use with Ollama, OpenAI-compatible APIs, or any other LLM provider.

### Option 1: Use Parent Model

Set `use_parent_model: true` to make all agents use the parent session's model (useful when launching Claude Code with Ollama):

```json
{
  "use_parent_model": true
}
```

### Option 2: Define Custom Models

Define your own model assignments in config:

```json
{
  "model_profile": "balanced",
  "model_profiles": {
    "gsd-planner": {
      "quality": "glm-5:cloud",
      "balanced": "minimax-m2.5:cloud",
      "budget": "qwen3-next:cloud"
    },
    "gsd-executor": {
      "quality": "minimax-m2.5:cloud",
      "balanced": "glm-5:cloud",
      "budget": "qwen3-coder-next:cloud"
    },
    "gsd-phase-researcher": {
      "quality": "glm-5:cloud",
      "balanced": "minimax-m2.5:cloud",
      "budget": "qwen3-next:cloud"
    },
    "gsd-project-researcher": {
      "quality": "glm-5:cloud",
      "balanced": "minimax-m2.5:cloud",
      "budget": "qwen3-next:cloud"
    },
    "gsd-research-synthesizer": {
      "quality": "minimax-m2.5:cloud",
      "balanced": "qwen3-next:cloud",
      "budget": "rnj-1:cloud"
    },
    "gsd-debugger": {
      "quality": "minimax-m2.5:cloud",
      "balanced": "glm-5:cloud",
      "budget": "qwen3-coder-next:cloud"
    },
    "gsd-codebase-mapper": {
      "quality": "qwen3-coder-next:cloud",
      "balanced": "devstral-small-2:cloud",
      "budget": "rnj-1:cloud"
    },
    "gsd-verifier": {
      "quality": "minimax-m2.5:cloud",
      "balanced": "qwen3-next:cloud",
      "budget": "nemotron-3-nano:cloud"
    },
    "gsd-plan-checker": {
      "quality": "glm-5:cloud",
      "balanced": "minimax-m2.5:cloud",
      "budget": "qwen3-next:cloud"
    },
    "gsd-integration-checker": {
      "quality": "minimax-m2.5:cloud",
      "balanced": "qwen3-coder-next:cloud",
      "budget": "nemotron-3-nano:cloud"
    }
  }
}
```

### Global Defaults

Instead of configuring each project, you can set defaults globally in `~/.gsd/defaults.json`. These apply to all projects unless overridden.

## Debugging Model Resolution

Use the `resolve-model` command to see which models are being assigned:

```bash
get-shit-done-cc resolve-model all
```

This shows the current profile, whether custom profiles are loaded, and each agent's assigned model.
