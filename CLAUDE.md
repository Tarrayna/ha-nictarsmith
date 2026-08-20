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
- automations/ — yes, freely (see "Automations file layout" below)
- scripts.yaml — yes, freely
- scenes.yaml — yes, with confirmation for anything affecting more than one room
- dashboards/ (Lovelace, YAML mode) — yes, freely for single-dashboard edits;
  confirmation for anything affecting the default Overview dashboard or a
  dashboard shared across the household (same bar as scenes.yaml)
- configuration.yaml — ONLY with explicit confirmation before editing; this
  file can break the whole HA instance if malformed
- secrets.yaml — NEVER read, write, or display contents. If a secret is
  needed, tell the user to add it manually and reference the key name only.
- blueprints/ — yes, freely
- Any .db, .log, deps/, tts/ files — never edit/delete, these are runtime
  data. Reading logs for diagnosis is fine (standing permission) - but note
  home-assistant.log isn't actually present on this mounted share and
  /api/error_log returns 404, so log-based diagnosis is limited to what
  /api/logbook exposes (state-transition events, not raw error/debug
  output). Deeper log access needs the HA UI (Settings → System → Logs).

## Automations file layout
Automations no longer live in a single automations.yaml. configuration.yaml
loads them via `automation: !include_dir_merge_list automations/`, which
merges every *.yaml file in that directory into one automation list.

- `automations/poc.yaml` — scratch/proof-of-concept area. Unless a new
  automation is trivial, build it here first to confirm it actually works
  (trigger fires, actions do the right thing) before moving it to its
  permanent home.
- `automations/<group>.yaml` — grouped by area or purpose once an
  automation is confirmed working (e.g. office.yaml, living_room.yaml,
  xbox.yaml). Create a new group file when a natural category emerges;
  don't force everything into one bucket.
- When "migrating" an automation out of poc.yaml, cut its entry from
  poc.yaml and paste it into the appropriate group file in the same edit
  — don't leave duplicates in both places.

## Dashboards
Dashboards run in Lovelace YAML mode: one file per dashboard under
`dashboards/`, referenced from a `lovelace:` block in configuration.yaml
(the block registers each dashboard's url_path/title/icon and points it at
its file — that registration part is a configuration.yaml edit, so it needs
confirmation like any other configuration.yaml change). Storing dashboards
as files instead of in `.storage/lovelace*` is what makes them editable and
git-backed like everything else here.

- When building or editing a dashboard, favor reusable pieces over one-off
  duplication: template/helper entities (input_boolean, input_number,
  template sensors, etc.) behind card conditions rather than hardcoding
  per-card logic, and consistent, repeated card/section structures instead
  of copy-pasted one-off blocks, so a change in one place propagates.
- If a dashboard needs a new helper entity that doesn't exist yet, flag it
  rather than assuming — creating helpers may itself need a
  configuration.yaml edit or a user action in Settings → Helpers.

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
- Push to the private GitHub remote automatically after every commit — the
  user has standing permission for this and would otherwise forget. Do not
  wait to be asked.
- Never commit secrets.yaml (already gitignored) or database/log files, and
  never commit .storage/ runtime state (auth, entity_registry, restore_state)
- Suggested commit message format: "Add: <what>" or "Update: <what>" or
  "Fix: <what>"

## Testing / validation
- After writing YAML, check it's syntactically valid before considering
  the task done (yamllint if available, or at minimum a visual re-check)
- After editing any file under automations/, reload automations yourself via
  the API (see "Reload automations after changes" below) rather than asking
  the user to. If configuration.yaml was touched, HA needs a full restart —
  tell the user and get confirmation before restarting (same as the
  confirmation needed to edit configuration.yaml in the first place).

## assist_satellite.ask_question gotchas
Built while debugging the xbox_downstairs_lights_prompt automation
(2026-07-12). Read this before building anything else with ask_question.

- **Never call `assist_satellite.ask_question` directly inside an
  automation's `actions`.** There's a known HA core bug where it can
  silently hang forever - the automation trace shows "running"
  indefinitely, nothing appears in the log. Community-confirmed fix: put
  the ask_question call inside a **script**, and have the automation just
  call that script. See `script.xbox_ask_downstairs_lights_prompt` in
  scripts.yaml for the working pattern. Related upstream issues:
  github.com/home-assistant/core/issues/151589, #142363, #160806, #149584.
- **The matched answer's `id` is top-level on the response dict, not
  nested under a `response` key.** With `response_variable: my_response`,
  the correct check is `{{ my_response.get('id') == 'yes' }}` — NOT
  `{{ my_response.response.id == 'yes' }}`. The nested form throws
  `UndefinedError: 'dict object' has no attribute 'response'` on a
  successful match (confirmed via our own trace error) even though some
  official-looking examples suggest the nested form. Always use `.get('id')`
  rather than direct attribute access, since the key is absent entirely on
  timeout/no-match (not just null) - direct access crashes the template in
  that case.
- **`preannounce` plays before the spoken `question`, not before
  listening.** There's no built-in "now listening" chime. If you want a
  distinct cue for when to start answering, you'd need to either split into
  `assist_satellite.announce` (preannounce: false, speaks the question)
  followed by `assist_satellite.ask_question` with a short non-empty
  filler question and preannounce:true (empty `question: ""` risked
  hanging the TTS call - use a real short word instead), or accept there's
  no such cue for now. We reverted this idea for the Xbox automation to
  keep the working version simple - revisit if it comes up again.
- **Known intermittent audio-capture issue on Voice PE**: even with the
  above fixed, the "listening" state that opens after the question
  (no wake word involved) sometimes captures zero speech regardless of
  listening-window length (confirmed up to 12s, standing right next to the
  device, no background noise). Wake-word-triggered listening ("Hey
  Jarvis...") works reliably on the same hardware, so this is isolated to
  the no-wake-word continue-listening path specifically - looks like an
  upstream/firmware reliability issue, not something fixable from config.
  `select.<device>_finished_speaking_detection` (options: default/
  relaxed/aggressive) controls how long it waits, but doesn't fix
  zero-capture cases. If this keeps blocking a use case, the fallback is
  to have the satellite announce and rely on a normal wake-word follow-up
  command instead of ask_question's continue-listening.
- **A generic reusable `script.ask_question` wrapper was tried and
  reverted** (2026-07-12) in favor of keeping ask_question calls inline in
  each purpose-built script. Not because the wrapper was broken - it
  worked structurally - but to keep the one proven-working version
  isolated while the intermittent capture issue above is still
  unresolved. Revisit building a shared helper once ask_question is
  reliably working end-to-end.

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
