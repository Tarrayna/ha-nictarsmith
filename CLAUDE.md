# Home Assistant Config — Agent Instructions

## Context
This is Tarrayna and Nicholas's Home Assistant configuration, running on a
NUC with HAOS, mounted here via Samba from Legion (the AI server).
Voice pipeline: Parakeet STT, Orpheus TTS (voice: tara), Qwen3 4B-instruct
via Ollama (CPU) as the conversation agent. Voice PE satellites throughout
the house. This is a real home, not a test environment.

## What this agent is for
Primary job: build out Home Assistant automations, scripts, and related
YAML config based on requests from Tarrayna or Nicholas, and keep this
config backed up to GitHub. This is an ongoing working relationship, not
a one-off task — expect a steady stream of new automation requests over
time.

Example projects discussed so far (not an exhaustive or required list,
just context): doorbell announcements when the baby is sleeping, a
voice prompt asking whether to dim the lights when the Xbox turns on
using assist_satellite.ask_question, general notification automations.

## Where you're allowed to touch
- automations.yaml — yes, freely
- scripts.yaml — yes, freely
- scenes.yaml — yes, with confirmation for anything affecting more than one room
- configuration.yaml — ONLY with explicit confirmation before editing; this
  file can break the whole HA instance if malformed
- secrets.yaml — NEVER read, write, or display contents. If a secret is
  needed, tell the user to add it manually and reference the key name only.
- blueprints/ — yes, freely
- Any .db, .log, deps/, tts/ files — never touch, these are runtime data

## Naming conventions
- Automation IDs: snake_case, prefixed by area/device
  (e.g., front_door_baby_sleeping, xbox_on_dim_lights)
- Always include a human-readable `alias` and a one-line `description`
- Entity references: prefer friendly names in comments even when using
  entity_id in the YAML itself

## Safety rules
- NEVER disable or modify an existing automation without explicit confirmation
  — describe what you're about to change and wait for a yes
- NEVER touch configuration.yaml without explicit confirmation first
- After any change, summarize what changed in plain language before
  considering the task done
- If a request is ambiguous about which entity/area it applies to, ask
  rather than guessing on physical devices (lights, locks, anything safety
  related)

## Git / backup workflow
- This directory is a git repo backed up to a private GitHub repo
- Do NOT commit automatically after every small edit
- When the user confirms they're happy with a change ("looks good",
  "that works", "commit that", etc.), THEN commit with a clear message
  describing what was added/changed
- Never commit secrets.yaml (already gitignored) or database/log files
- Suggested commit message format: "Add: <what>" or "Update: <what>" or
  "Fix: <what>"

## Testing / validation
- After writing YAML, check it's syntactically valid before considering
  the task done (yamllint if available, or at minimum a visual re-check)
- After editing automations.yaml, reload it yourself via the API (see
  "Reload automations after changes" below) rather than asking the user to.
  If configuration.yaml was touched, HA needs a full restart — tell the user.

## Reload automations after changes
- A long-lived access token is stored at `~/.ha_token` (chmod 600). Do not
  print, echo, or commit its contents — only reference it via `$(cat ~/.ha_token)`.
- Reload automations by calling the HA REST API:
  ```
  curl -X POST \
    -H "Authorization: Bearer $(cat ~/.ha_token)" \
    -H "Content-Type: application/json" \
    http://192.168.1.100:8123/api/services/automation/reload
  ```
- A `200` response with an empty array `[]` in the body means the reload
  succeeded (the array lists changed states, not an error).
- Make this a standard step you run automatically after ANY edit to
  automations.yaml, right before you summarize the change to the user. You
  own this step now — don't ask the user to reload manually.
