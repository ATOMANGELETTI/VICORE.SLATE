# Agent Model Selection — Design Spec

**Date:** 2026-08-29  
**Status:** Revised (Grok-first; Sonnet/Opus escalation only)

## Goal

Default to **Grok 4.6 xhigh fast** for maximum cost-effectiveness. Use Grok as much as
possible. Escalate to **Sonnet 5** or **Opus 5** only when planning or the agent
determines a stronger model is needed to complete the task, or when Grok gets stuck.

## Approach

**Tiered heuristic** (Grok-first), not a hard skill→model matrix. Documented in an
always-apply Cursor rule plus a short `AGENTS.md` preference. Classify each Superpowers
/ Task unit of work into a tier and set `model` accordingly. Never use Composer 2.5.

## Policy

### When to choose

Before Superpowers workflows, Task/subagent launches, or other multi-agent units of work,
classify the **unit** into a tier and set Task `model` explicitly. Do not inherit blindly
from the parent chat model.

### Tiers (allowlist slugs only)

| Tier            | When                                                                                                                                 | Slug                            |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------- |
| **0 – Default** | Implementation, explore, shell, scaffolding, routine fixes, most parallel workers, most planning and execution                       | `cursor-grok-4.6-xhigh-fast`    |
| **1 – Mid**     | Planning or execution determines a stronger model is needed; hard debugging after Grok fails; dense cross-cutting refactors          | `claude-sonnet-5-thinking-high` |
| **2 – Top**     | Agent judges top-tier capability required; deep irreversible architecture; security review; Sonnet still failing or Grok stuck again | `claude-opus-5-thinking-high`   |

### Decision rule

1. Start at **Tier 0 (Grok)** unless planning or the agent clearly determines a stronger
   model is required.
2. Prefer staying on Grok when “close enough.”
3. Escalate one tier when Grok failed/stuck, planning calls for more capability, task is
   capability-bound, or failure cost is high.
4. Do not use Tier 1 or 2 preemptively.
5. Explicit user model requests always win.
6. Never invent slugs; if a tier slug is disabled, escalate to the next tier rather than
   substituting unrelated models.

### Superpowers / multi-agent examples (guidance, not a matrix)

- `explore` / `shell` / routine `generalPurpose` / most implementers → **Tier 0**
- `writing-plans` / `executing-plans` → **Tier 0** unless a slice clearly needs Sonnet
  or Opus
- `systematic-debugging` after Grok fail → **Tier 1**; Opus if still stuck → **Tier 2**
- `brainstorming` for deep irreversible architecture, `security-review` → **Tier 2**
- Parallel workers: default all Grok; escalate only workers whose slice needs Sonnet or Opus

## Deliverables

| Path                                          | Role                                            |
| --------------------------------------------- | ----------------------------------------------- |
| `.cursor/rules/012-agent-model-selection.mdc` | Always-apply Grok-first tier policy             |
| `AGENTS.md`                                   | Preference bullet + rule-range fact `000`–`012` |

## Out of scope

- Hard skill→model matrix
- Changing the Cursor UI default chat model
- moon/CI model configuration
- Kimi / GLM / GPT / Composer as automatic escalation targets
