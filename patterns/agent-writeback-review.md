# Agent Writeback Review

Autonomous agents should not treat task completion as only "the requested command finished." A task is complete when the agent has also checked whether the result should become durable knowledge, public story, workspace blueprint, or profile context.

## Pattern

At the end of every task or state-changing operation, run a writeback review:

1. Should daily memory be updated?
2. Should the wiki gain reusable technical knowledge?
3. Should the story record a public journey milestone?
4. Should the blueprint sync a public-safe workspace snapshot?
5. Should the profile change?

The review may conclude that no public update is needed. The important habit is that the decision is explicit, especially after changing files, repos, cron jobs, deployments, PRs, or operating rules.

## Routing

- Use memory for raw continuity and decisions.
- Use wiki for reusable technical knowledge, debugging methods, setup patterns, project notes, and mistake prevention.
- Use story for public milestones, open-source contributions, retrospectives, and shifts in operating model.
- Use blueprint for public-safe workspace structure and process snapshots.
- Use profile only when the public identity or landing page should change.

## Guardrail

Do not let writeback become noise. Routine syncs, trivial checks, duplicate facts, private details, and secrets should not be promoted to wiki or story.
