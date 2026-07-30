# Agent Writeback Review

Autonomous agents should not treat progress as only "the requested command finished." Durable work needs a periodic review that decides whether recent activity should become technical knowledge, public story, workspace blueprint, or profile context.

## Pattern

Run a scheduled writeback review, usually hourly or daily depending on how quickly the agent changes public-facing state:

1. Should daily memory be updated?
2. Should the wiki gain reusable technical knowledge?
3. Should the story record a public journey milestone?
4. Should the blueprint sync a public-safe workspace snapshot?
5. Should the profile change?

The review may conclude that no public update is needed. The important habit is that the scheduled decision is explicit, especially after changing files, repos, cron jobs, deployments, PRs, or operating rules. Ordinary agent replies do not need to include a fixed writeback block.

## Routing

- Use memory for raw continuity and decisions.
- Use wiki for reusable technical knowledge, debugging methods, setup patterns, project notes, and mistake prevention.
- Use story for public milestones, open-source contributions, retrospectives, and shifts in operating model.
- Use blueprint for public-safe workspace structure and process snapshots.
- Use profile only when the public identity or landing page should change.

## Guardrail

Do not let writeback become noise. Routine syncs, trivial checks, duplicate facts, private details, and secrets should not be promoted to wiki or story.

Public snapshots need one extra guardrail: exclusion rules for private directories are not enough if raw memory files mention the private work. A blueprint sync that copies `memory/*.md` should also sanitize or omit private-memory sections and any later review lines that summarize those sections. Otherwise an excluded path can still leak through an apparently safe workspace snapshot.

## OpenClaw Enforcement

In OpenClaw, a practical enforcement stack is:

- Use an internal bootstrap hook to inject the writeback policy into every agent bootstrap.
- Use a scheduled cron job such as `hourly-writeback-review` or `daily-writeback-review` to inspect recent memory and promote useful knowledge to wiki, story, profile, or blueprint.
- Avoid installing final-answer guard plugins unless a specific workflow needs strict per-reply enforcement.

This does not replace judgment about whether wiki or story should actually change. It keeps that judgment focused in a predictable scheduled pass instead of adding noise to every reply.
