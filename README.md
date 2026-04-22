# spark-cli

Local installer and operator CLI for the Spark module ecosystem. A single
command installs a Spark module (by registry name, git URL, or local path),
collects its secrets into the OS keychain, wires its env, runs its
healthchecks, and keeps its process under supervision.

Fourteen commands cover the full install lifecycle. This repo is an
opinionated spike — single Python file, one runtime dependency (`keyring`),
and a registry file you can edit by hand.

---

## Quick start

On any machine with Python 3.11+ and git on PATH:

```bash
git clone https://github.com/vibeforge1111/spark-cli
cd spark-cli
pip install -e .

# Scaffold a new module
spark init my-module --kind python
spark install ./my-module
spark status
```

That's the full loop. Everything else is either a different source
(`install github.com/...`), a different bundle (`setup telegram-starter`), or
an operator command (`start`, `stop`, `logs`, `status`, `secrets`, `config`).

> If another `spark` binary is already on your PATH, use `spark-local`
> (pyproject aliases both to the same entrypoint).

---

## Requirements

| Dependency | Why |
|---|---|
| Python 3.11+ | The CLI itself |
| `git` on PATH | To clone git-sourced modules and pull updates |
| OS keychain (auto) | Windows Credential Manager / macOS Keychain / libsecret — for `storage = "keychain"` secrets. Falls back to a mode-0600 file when none is available. |

Per-module runtimes (Python, Node, bun, uv, ...) are the module's business.
The CLI detects them at setup time, reports whether they're present, and
enforces `[runtime].version` constraints declared in each module's
`spark.toml` before running install commands. Pass `--skip-runtime-check` to
override.

---

## Install the CLI

```bash
git clone https://github.com/vibeforge1111/spark-cli
cd spark-cli
pip install -e .
```

Confirm:

```bash
spark --help
spark-local --help   # alias if `spark` is shadowed
```

Run the test suite:

```bash
pip install pytest
python -m pytest tests/ -q
```

---

## Commands

14 top-level commands. Use `spark <cmd> --help` for flags.

| Command | What it does |
|---|---|
| `spark list` | List discoverable modules |
| `spark init <name>` | Scaffold a new module (`--kind python\|node`, `--path`, `--description`) |
| `spark install <target>` | Install by registry name, bundle, local path, or git URL |
| `spark setup <bundle>` | Interactive preflight + secret prompts for a whole bundle |
| `spark update [target]` | Re-run install commands; `git pull --ff-only` for managed clones |
| `spark uninstall [target]` | Tear down: stop process, drop env, delete clone, rotate secrets |
| `spark start [target]` | Topological launch using `needs.modules` order; polls `ready_check` |
| `spark stop [target]` | Reverse-topological kill |
| `spark status [--json]` | Run all module healthchecks with repair hints |
| `spark doctor [--json]` | Diagnostic variant of status |
| `spark logs <module>` | Tail `~/.spark/logs/<module>/process.log` (`-n N`, `-f`) |
| `spark search [query]` | Browse the registry with blessed + installed badges |
| `spark secrets list\|set\|get\|delete` | Keychain-backed secret store |
| `spark config get\|set\|unset\|list` | User config at `~/.spark/config/config.json` |

Global install-time flags on `install` and `setup`:

- `--skip-install-commands` — skip `[install.dev].commands`
- `--skip-runtime-check` — skip `[runtime].version` enforcement
- `--trust` — approve running non-blessed module's install commands and hooks without prompting
- `--resume` — skip install steps that succeeded on a prior attempt
- `--non-interactive` (`setup` only) — fail instead of prompting for missing secrets

---

## Creating your own module

```bash
spark init my-chip --kind python --description "A thing I built"
cd my-chip
# Edit spark.toml: fill [install.dev].commands, [provides.capabilities],
# [needs.secrets], [healthcheck].command, etc.
cd ..
spark install ./my-chip
spark status
```

The scaffolded `spark.toml` is schema-1 compliant and installs cleanly out of
the box with a healthcheck that always returns ok. See any of the
`spark-intelligence-builder`, `spark-telegram-bot`, or `spawner-ui` manifests
for full-featured examples.

---

## The `registry.json` caveat

`registry.json` in this repo currently points the starter bundle modules
(`spark-telegram-bot`, `spark-intelligence-builder`, `spawner-ui`) at
**local Windows paths** on the original author's machine. That's how the
spike was developed; it **will not work on any other machine** as-is.

To use the bundled `telegram-starter` on a different computer, you have two
clean options:

**1. Replace the local paths with git URLs.** Edit `registry.json`:

```json
{
  "modules": {
    "spark-telegram-bot": {
      "source": "https://github.com/<owner>/spark-telegram-bot",
      "blessed": true,
      "summary": "Telegram ingress gateway for the Spark starter stack"
    }
  }
}
```

`spark install` will `git clone --depth=1` into
`~/.spark/modules/<name>/source/` and load the manifest from the clone.

**2. Skip the bundled registry and install individual modules by path or
git URL.** The bundle is just a convenience; the CLI works fine without it:

```bash
spark install github.com/someone/spark-telegram-bot
spark install ./my-local-module
```

A future commit will flip the registry to git URLs once the three starter
modules have canonical public repos.

---

## State layout

The CLI owns everything under `~/.spark/`:

```
~/.spark/
├── state/
│   ├── installed.json             # installed modules + provenance
│   ├── setup.json                 # configured bundle + ingress owner
│   ├── pids.json                  # running process pids
│   └── install_progress.json      # checkpoint for --resume
├── config/
│   ├── config.json                # user-level config via `spark config`
│   ├── modules/<name>.env         # generated per-module env files (non-secret)
│   ├── secrets_index.json         # which backend holds each secret
│   └── secrets.local.json         # only when keychain is unavailable
├── modules/<name>/source/         # clone target for git-sourced modules
└── logs/<name>/process.log        # per-module process logs
```

`spark uninstall <module>` removes only that module's entries — never touches
other modules. No "uninstall all" flag yet; wipe `~/.spark/` manually if you
want a clean slate.

---

## How it works

### Install lifecycle

```
spark install <target>
  1. resolve target (registry name -> source, git URL, or local path)
  2. clone if git-sourced (idempotent)
  3. validate manifest schema version
  4. detect capability conflicts (e.g. two telegram.ingress owners)
  5. resolve needs.capabilities against installed + batch modules
  6. enforce [runtime].version constraints
  7. trust prompt if non-blessed (or require --trust in non-interactive mode)
  8. run [install.dev].commands (skippable with --resume if already done)
  9. record install in ~/.spark/state/installed.json
  10. clear install progress checkpoint on success
```

### Secrets flow

- Manifests declare `[needs.secrets]` and per-secret `[secrets.<id>]` blocks
  with `storage = "keychain" | "file"` and `env_var = "..."`.
- `spark setup` prompts for each required secret (deduped across bundle
  modules); `storage = "keychain"` values go to OS Credential Manager,
  `storage = "file"` values land in `~/.spark/config/modules/<name>.env`.
- At `spark start`, keychain-backed values are read back and injected into
  the subprocess env by `env_var` name. Modules only ever see env vars.
- Rotate: `spark secrets set <secret_id>` and restart the module.

### Process lifecycle

- `spark start` reads `[run.default].command`, spawns it detached on Windows
  (`DETACHED_PROCESS` + `CREATE_NEW_PROCESS_GROUP`), logs stdout/stderr to
  `~/.spark/logs/<module>/process.log`, and polls `[run.default].ready_check`
  (HTTP URL or shell command) until `[healthcheck].timeout_seconds`.
- `spark stop` walks the reverse dependency graph and kills each module's
  tracked pid (`taskkill /PID ... /T /F` on Windows, `kill` elsewhere).
- Stale pids in `pids.json` are detected (`os.kill(pid, 0)`) and dropped on
  the next `spark start`.

---

## Project layout

```
spark-cli/
├── LICENSE                        # MIT
├── README.md                      # this file
├── pyproject.toml                 # name=spark-cli, deps=[keyring>=24.0]
├── registry.json                  # blessed modules + bundles (local paths today)
├── docs/
│   ├── STATUS.md                  # living audit — update at the end of any session
│   └── design/                    # v1 design docs (friction map, flows, installer spec)
├── src/spark_cli/
│   ├── __init__.py
│   └── cli.py                     # ~2400 LOC; everything in one module
└── tests/
    └── test_cli.py                # 83 tests, unittest + mock
```

---

## Development

```bash
pip install -e .
pip install pytest

python -m pytest tests/ -q                     # 83 tests
python -m spark_cli.cli list                   # discoverable modules
python -m spark_cli.cli init demo --kind python
python -m spark_cli.cli install ./demo
python -m spark_cli.cli status
python -m spark_cli.cli uninstall demo
```

See [`docs/STATUS.md`](./docs/STATUS.md) for the current feature matrix,
what's deliberately deferred, and what's still unsure.

See [`docs/design/`](./docs/design/) for the v1 design doc, the
lessons-learned rationale, and the user-flow / friction map.

---

## License

MIT. See [`LICENSE`](./LICENSE).
