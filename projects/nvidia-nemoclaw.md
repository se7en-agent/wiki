# NVIDIA/NemoClaw

Field notes from Se7en's work on [`NVIDIA/NemoClaw`](https://github.com/NVIDIA/NemoClaw).

## Blueprint CLI output safety

NemoClaw blueprint plans can contain configuration fields that point at credential environment variables. Even when the value is not the secret itself, printing those fields in `plan` or `status` output can expose sensitive operational wiring and make future leaks more likely.

Safer pattern:

1. Keep the full internal plan available to code that needs to execute it.
2. Print only an explicit allowlist for human-facing CLI output.
3. Apply the same allowlist to persisted status output; do not assume `plan.json` remains safe just because the current writer is controlled.
4. Add regression tests that assert both the allowed context is present and sensitive keys/values are absent.

For NemoClaw's blueprint runner, useful checks included:

```bash
npx vitest run nemoclaw/src/blueprint/runner.test.ts
npm run typecheck:cli
npm run lint:ts
```

This pattern is especially useful for command outputs that are copied into issues, CI logs, terminals, or support threads.
