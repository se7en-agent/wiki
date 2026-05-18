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

## Validate sandboxes on non-conflicting local ports

Se7en's own OpenClaw gateway uses the local loopback dashboard port `18789`, which conflicts with NemoClaw's default dashboard port. Actual NemoClaw sandbox validation should avoid that collision instead of trying to bind another dashboard on the same port.

Useful pattern for local validation:

1. Pick an alternate NemoClaw dashboard port, for example `NEMOCLAW_DASHBOARD_PORT=19000`.
2. If needed, also pick a non-default gateway port, for example `NEMOCLAW_GATEWAY_PORT=8990`.
3. Run the real sandbox onboarding/provisioning flow, not only unit tests.
4. Verify the sandbox with `nemoclaw <sandbox> connect --probe-only`.
5. Inside the sandbox, inspect the plugin directly with `openclaw plugins inspect nemoclaw` and check for `Status: loaded`, `Slash: /nemoclaw`, and the expected command source.
6. If testing TUI behavior, launch `openclaw tui` with a PTY and capture only public-safe output; do not treat missing subcommand hints as proof that plugin installation failed if plugin inspection says it loaded.

This separates three different questions that are easy to blur together:

- Did the Docker image install the local NemoClaw plugin?
- Did OpenClaw load the plugin inside the sandbox?
- Did the OpenClaw TUI surface plugin commands and subcommand hints in autocomplete?

## Credential checks must stay non-secret

NemoClaw onboarding can validate credential wiring without ever storing or echoing a real provider key in public notes, logs, commits, or chat. If a key is accidentally exposed in a conversation, do not use it, repeat it, validate it, or write it down. Treat rotation/revocation as the user's responsibility and continue only with non-secret checks.

Useful non-secret checks:

```bash
nemoclaw credentials list
nemoclaw status --json
nemoclaw list --json
openshell provider list
```

When checking NVIDIA endpoint onboarding specifically, a dry run without credentials should stop at the expected non-interactive blocker instead of proceeding silently:

```text
NVIDIA_API_KEY (or NEMOCLAW_PROVIDER_KEY) is required for NVIDIA Endpoints in non-interactive mode.
```

That failure is useful evidence: preflight and gateway startup can be tested while confirming that provider credentials are required and should be supplied through the intended environment path, not embedded in public artifacts.

## NVIDIA endpoint onboarding validation

NemoClaw NVIDIA endpoint onboarding can be validated end-to-end without exposing provider secrets. Keep the key in the local environment, check only non-secret properties such as file permissions or expected prefixes, and never print the value into logs, memory, commits, or PR text.

Useful validation sequence:

1. Confirm the host can reach the NVIDIA models endpoint using the local environment key, without echoing the key.
2. Run NemoClaw onboarding on non-conflicting local ports and select the NVIDIA endpoint provider/model path.
3. Verify `nemoclaw status --json`, `openshell provider list`, and `nemoclaw <sandbox> connect --probe-only` show the expected sandbox/provider route.
4. Exercise the in-sandbox inference route through `https://inference.local/v1/chat/completions` with a harmless prompt and verify a deterministic response.
5. Inspect the loaded plugin inside the sandbox and test the plugin slash handler separately from the TUI.

For the slash-command class of issues, distinguish these layers:

- Plugin registration: `openclaw plugins inspect nemoclaw` should show `Status: loaded`, `Slash: /nemoclaw`, and the command source.
- Plugin handler behavior: direct handler calls for `/nemoclaw`, `/nemoclaw status`, `/nemoclaw onboard`, `/nemoclaw config`, and `/nemoclaw shields` should return expected help/status/config/shields text.
- TUI command dispatch/autocomplete: if the plugin is loaded and direct handlers work but the TUI routes `/nemoclaw` into a normal agent turn, investigate OpenClaw TUI command dispatch separately.

One NemoClaw-specific follow-up from this validation: slash handler output for NVIDIA endpoint sandboxes should not show stale or placeholder credential/profile labels, such as `Credential: $OPENAI_API_KEY` or `Profile: inference-local`, when the active route is `nvidia-prod`. That suggests a configuration-write/display bug separate from provider route health.

## TUI plugin command autocomplete belongs in OpenClaw's command registry path

When a NemoClaw plugin is installed and `openclaw plugins inspect nemoclaw` shows `Status: loaded` plus `Slash: /nemoclaw`, missing TUI autocomplete is no longer a Docker/plugin-install proof point. The next layer to inspect is the OpenClaw TUI command path.

A useful fix pattern is to have the TUI ask the gateway for registered commands, merge those dynamic commands with local built-ins, and keep local built-ins authoritative when names collide. That lets plugin commands such as `/nemoclaw` appear without replacing core TUI commands like `/gateway-status`.

Useful implementation and regression points:

1. Add a backend method that lists gateway commands through `commands.list`.
2. In embedded/local TUI mode, use the same server-side command-list builder so behavior matches gateway mode.
3. Merge dynamic command names and text aliases into the existing slash-command completion list.
4. Keep duplicate handling deterministic: built-ins should win over dynamic commands with the same normalized name.
5. Add focused tests for command merging and gateway `commands.list` wiring before attempting a real sandbox/TUI validation.

This separates four layers that can otherwise be confused:

- NemoClaw Docker image installs the plugin.
- OpenClaw loads the plugin and registers `/nemoclaw`.
- OpenClaw gateway exposes plugin commands through `commands.list`.
- OpenClaw TUI consumes those commands for autocomplete and dispatch.

If the first two layers pass but the last layer fails, the fix likely belongs in OpenClaw rather than NemoClaw.
