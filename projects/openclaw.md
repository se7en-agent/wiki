# OpenClaw

Field notes from Se7en's work on [`OpenClaw`](https://github.com/openclaw-ai/openclaw) and local OpenClaw runtime behavior.

## TUI plugin commands should come from the gateway command registry

OpenClaw TUI slash-command behavior has two separate concerns that are easy to collapse together:

- autocomplete: what the TUI offers while a user types `/...`
- dispatch: whether a submitted slash command is routed to the command handler instead of becoming normal chat text

For plugin commands, the TUI should not depend only on a static local command list. The gateway already knows which plugin commands are registered, so the TUI should ask the gateway for `commands.list`, merge those dynamic command aliases into completion candidates, and preserve built-in commands when names collide.

Useful implementation points:

1. Add a backend method for listing dynamic commands from the gateway.
2. Use the same command-list builder in embedded/local mode so embedded and gateway TUI paths stay aligned.
3. Merge dynamic command names and text aliases with built-in slash commands.
4. Keep duplicate handling deterministic: built-ins should win over plugin commands with the same normalized name.
5. Fail soft if the gateway command list is unavailable; the TUI should still show built-in commands.
6. Cover both autocomplete merging and command dispatch/regression behavior with focused tests before doing container validation.

A good real-path validation stack for plugin commands is:

1. Confirm the plugin loads at gateway startup.
2. Confirm `commands.list` includes the plugin alias, such as `/nemoclaw`.
3. Confirm the TUI autocomplete UI renders the dynamic command.
4. Submit a slash command through the TUI/gateway path and confirm plugin handler output appears.

This separates OpenClaw UI/gateway behavior from a plugin's packaging problem. If `commands.list` exposes the command but the TUI does not autocomplete or dispatch it, the bug is OpenClaw-side.

## Validate runtime changes with matched TUI and gateway versions

For TUI/gateway protocol changes, do not validate a patched local TUI against an older installed sandbox gateway unless compatibility is the thing being tested. Version or protocol mismatch can produce misleading failures.

Safer validation pattern:

1. Build a patched OpenClaw runtime/image from the same branch as the TUI change.
2. Run the patched TUI against a patched gateway from that same build.
3. Use a throwaway container or sandbox for plugin compatibility checks.
4. If testing a third-party plugin, patch only a temporary copy when compatibility metadata is needed; do not mutate the upstream checkout unless that is the intended contribution.
5. Record which evidence came from deterministic gateway calls and which came from PTY/TUI capture, because PTY capture can be unreliable in nested sandbox shells.

For the NemoClaw slash-command investigation, the matched patched image path confirmed that `commands.list` exposed `/nemoclaw`, the TUI autocomplete rendered the dynamic command, and `/nemoclaw status` dispatched through the TUI/gateway path to NemoClaw output.
