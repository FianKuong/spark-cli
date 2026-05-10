# Access, Sandbox, and Nontechnical UX Audit - 2026-05-10

## Verdict

Spark now has the core access and sandbox state machine, but not every user surface consumes it yet.

The safest truthful status is:

- Access levels 1-5 are defined and tested in Telegram, and `spark access status --level <1-5>` now reports the effective state.
- Level 4 has a real default workspace sandbox setup path in `spark-cli`.
- Level 5 is no longer just a label; it requires the full guardrail bundle before the effective state becomes 5.
- Docker doctor and no-secret smoke are real and tested on Windows and Ubuntu/WSL.
- Spawner now uses real Spark CLI commands and run policies for access lanes.
- Builder/AOC separates permission from current runner capability.
- Telegram can now run the core safe setup actions directly from chat commands. The remaining nontechnical gap is polished buttons/wizards, not backend capability.

## Access Levels

| Level | Intended meaning | Current status | Gap |
| --- | --- | --- | --- |
| 1 | Chat, memory, recall, diagnostics | Telegram gates builds/research/local work away; CLI reports `chat_memory` with no sandbox setup | Good enough |
| 2 | Requested builds and missions | Spawner side doors are gated; explicit builds allowed; CLI reports `requested_missions` | Good enough |
| 3 | Public research plus builds | Research/local boundaries are explained and tested; CLI reports `public_research` | Needs more live route evidence in AOC |
| 4 | Workspace sandbox | `spark access setup` creates/repairs Spark-owned workspace; Telegram can run `/access_setup`; runner writability is checked | Needs polished button/wizard UX |
| 5 | Whole-computer operator mode | Requires `SPARK_ALLOW_HIGH_AGENCY_WORKERS=1`, `SPARK_ALLOW_EXTERNAL_PROJECT_PATHS=1`, and `SPARK_CODEX_SANDBOX=danger-full-access`; Telegram can run `/level5_setup confirm`; restart state is explicit | Needs polished button/wizard UX and macOS live pass |

## Surface Audit

| Surface | What exists | Status |
| --- | --- | --- |
| `spark-cli` access payload | Requested vs effective access for Levels 1-5, Level 5 activation state, workspace preflight, Docker/SSH/Modal lanes, automation contract, deletion safety contract | Done |
| Telegram access policy | `/access 1-5`, natural access changes, Level 5 runtime guard, route-hijack regression tests, runner writability preflight | Mostly done |
| Telegram nontechnical setup | Runs `/access_setup`, `/docker_doctor`, `/docker_smoke confirm`, `/level5_setup confirm`, and `/level5_disable confirm` through fixed Spark CLI actions | Done for command UX; buttons/wizard still missing |
| Spawner access lanes | `/api/access/execution-lanes`, OS-aware lane recommendation, path redaction, real CLI commands, run policies, full Level 5 guardrail check | Done for status/recommendation |
| Spawner action execution | Actually running access setup, Docker smoke, Level 5 enable/disable from a safe UI | Missing |
| Builder/AOC | Access vs runner capability, stale context, source ledger, route confidence, action gates | Mostly done |
| Builder automation contract | AOC showing access vs runner capability and next route | Partial: should also render the CLI automation policy and next safe action |
| Docker sandbox | Doctor, no-secret smoke, network none, read-only root, no Docker socket, no Spark secrets | Done; verified Windows and Ubuntu/WSL, macOS pending |
| SSH sandbox | Target store, strict SSH options, host-key pinning, no private key contents, doctor/smoke/audit | Advanced path exists; not a normie default |
| Modal sandbox | SDK/auth doctor, no-secret smoke, audit | Advanced/cloud path exists; not a normie default |
| Deletion safety | Approval classifier flags delete, secrets, force push, deploy, Level 5 mutation; non-interactive sensitive commands block | Partial: quarantine/trash and executor-wide enforcement are not universal |
| Installer/onboarding | CLI and docs know the right actions | Missing: no website/Telegram wizard that performs setup end to end |

## What Should Happen For Nontechnical Users

Default path:

1. Spark checks access and runner state silently.
2. If Level 4 workspace is missing, Spark offers "Set up safe workspace" and runs `spark access setup`.
3. If Docker is useful and installed, Spark offers "Test Docker sandbox" and runs `spark sandbox docker smoke --json` after one confirmation.
4. If Docker is missing, Spark explains Docker Desktop/Engine in plain OS-specific words and keeps using the workspace sandbox.
5. If Level 5 is requested, Spark shows an explicit high-agency warning, runs `spark access setup --level 5 --enable-high-agency` only after opt-in, then asks for Spark restart.
6. After restart, Spark verifies effective Level 5 before saying whole-computer mode is active.

## What Must Stay Protected Even At Level 5

- Broad recursive deletion.
- Secrets reveal/export/delete.
- Public publish/deploy.
- Git history rewrite or force push.
- Cloud-cost actions.
- Writes outside the workspace when the user did not ask for whole-computer scope.

Level 5 changes the execution boundary. It must not disable the safety classifier.

## Next Fix Order

1. Add buttons or short replies for:
   - Set up safe workspace
   - Test Docker sandbox
   - Enable Level 5
   - Disable Level 5
2. Feed the CLI automation contract into AOC so Rec can say "allowed, blocked here, next safe action" without dumping terminal commands.
3. Make Spawner UI execute the same fixed actions safely instead of only recommending commands.
4. Keep those actions aligned to the contract run policies:
   - `auto_safe`
   - `auto_read_only`
   - `confirm_once`
   - `explicit_opt_in`
5. Add executor-wide quarantine/trash cleanup before broad delete support.
6. Run macOS live verification for Level 4, Docker, and Level 5 activation after the Windows/Ubuntu pass.
