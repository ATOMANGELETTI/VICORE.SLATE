# Agent Model Selection — Design Spec

**Date:** 2026-08-29  
**Status:** Revised (Auto-style tiers)

## Goal

Make Cursor agents (including Superpowers workflows and Task/multi-agent launches) choose models like Cursor **Auto**: default to Grok 4.6 for cost/capability balance, escalate through mid-tier then Opus only when a more capable model is necessary and worth the cost.

## Approach

**Tiered heuristic** (Grok-first), not a hard skill→model matrix. Documented in an always-apply Cursor rule plus a short `AGENTS.md` preference. Classify each Superpowers / Task unit of work into a tier and set `model` accordingly.

## Policy

### When to choose

Before Superpowers workflows, Task/subagent launches, or other multi-agent units of work, classify the **unit** into a tier and set Task `model` accordingly. Do not inherit blindly when a higher tier is warranted.

### Tiers (allowlist slugs only)

| Tier            | When                                                                                                                           | Preferred slug                                                        | Alternates if preferred unavailable                              |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- | ---------------------------------------------------------------- |
| **0 – Default** | Implementation, explore, shell, scaffolding, routine fixes, most parallel workers                                              | `cursor-grok-4.6-xhigh-fast` (Grok 4.6)                               | —                                                                |
| **1 – Mid**     | Complex plans, hard debugging after a Grok miss, dense cross-cutting refactors, non-trivial architecture tradeoffs             | Prefer `kimi-k3-max`; else `glm-5.2-high`; else `gpt-5.6-luna-medium` | Rotate mid-tier across parallel agents only when diversity helps |
| **2 – Top**     | Deep architecture / brainstorming for irreversible design, security review, high-stakes correctness, or mid-tier still failing | `claude-opus-5-thinking-high`                                         | —                                                                |

Use `inherit` / omit `model` only when the parent is already on an appropriate tier; otherwise set the slug explicitly for that Task.

### Decision rule

1. Start at **Tier 0 (Grok)** unless the unit is clearly mid/top.
2. Escalate **one tier** when: a weaker model already failed; the task is capability-bound; or failure cost is high (security, irreversible architecture, wrong plan = expensive rework).
3. Prefer staying on Grok when “close enough” — spend up only when the upgrade is worth it.
4. Explicit user model requests always win.
5. Never invent slugs; only session / Task allowlist. If a listed model is disabled, use the next alternate in-tier, then the adjacent tier.
6. Do not announce every pick; brief note only when escalating or when cost matters to the user.

### Superpowers / multi-agent examples (guidance, not a matrix)

- `explore` / `shell` / routine `generalPurpose` / most implementers → **Tier 0**
- `writing-plans`, hard `systematic-debugging` after Grok fail, complex plan execution leads → **Tier 1**
- `brainstorming` for deep architecture, `security-review`, critical design gates → **Tier 2**
- Parallel independent workers: default all Grok; escalate only the workers whose slice needs it

## Deliverables

| Path                                                                   | Role                                            |
| ---------------------------------------------------------------------- | ----------------------------------------------- |
| `.cursor/rules/012-agent-model-selection.mdc`                          | Always-apply Auto-style tier policy             |
| `AGENTS.md`                                                            | Preference bullet + rule-range fact `000`–`012` |
| `.superpowers/docs/plans/2026-08-29-vicore-slate-scaffold.phase.01.md` | Task 12b originally created `012` (historical)  |

## Out of scope

- Hard skill→model matrix
- Changing the Cursor UI default chat model
- moon/CI model configuration
