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

## Plugin installation should fail closed

NemoClaw's Docker image and smoke tests should not hide OpenClaw plugin installation failures. A fail-open install such as:

```bash
openclaw plugins install /opt/nemoclaw > /dev/null 2>&1 || true
```

can leave the container appearing healthy while `/nemoclaw` is unavailable in the TUI. That is especially dangerous when OpenClaw's plugin scanner blocks installation for an explicit reason, such as dangerous-code detection around `child_process`.

Safer pattern:

1. Install the local plugin as a required build/smoke step, not as a best-effort side effect.
2. Use `--dangerously-force-unsafe-install` only when the plugin's implementation has been reviewed and the override is intentional.
3. Preserve scanner error details in PR descriptions or troubleshooting notes so maintainers can see why the override is needed.
4. Make e2e smoke tests fail when plugin installation fails, even if built files such as `dist/index.js` exist.
5. Add regression tests around the provisioning behavior so future refactors do not reintroduce silent plugin failures.

Useful local checks for this class of fix included:

```bash
openclaw plugins install ./nemoclaw --dangerously-force-unsafe-install
npm test -- --run test/sandbox-provisioning.test.ts
npm run build:cli
npm run lint -- --files Dockerfile test/e2e-test.sh test/sandbox-provisioning.test.ts
npm run typecheck:cli
git diff --check
```

One remaining diagnostic to keep separate: successful plugin installation does not prove plugin slash-command autocomplete is wired through the OpenClaw TUI. If `/nemoclaw` still does not appear after a strict install succeeds, the next likely investigation is whether OpenClaw merges plugin command registry entries into TUI autocomplete.
