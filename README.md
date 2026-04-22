# Spark CLI Spike

This repo is a local spike for the Spark installer and operator CLI.

Current scope:

- read `spark.toml` manifests from local Spark module repos
- resolve the `telegram-starter` bundle from a local registry
- install modules by registry name or local repo path
- write deduped starter config into `~/.spark/`
- route the Telegram bot token only to `spark-telegram-bot`
- run module healthchecks through `spark status`
- start and stop startable modules through `spark start` and `spark stop`

This is intentionally a local-first spike, not the final packaged installer.

## Commands

```bash
python -m spark_cli.cli list
python -m spark_cli.cli install spark-telegram-bot
python -m spark_cli.cli install C:/Users/USER/Desktop/spawner-ui
python -m spark_cli.cli setup telegram-starter --bot-token <token> --admin-telegram-ids <ids>
python -m spark_cli.cli status
python -m spark_cli.cli start
python -m spark_cli.cli stop
```

If another `spark` binary already exists on your machine, use:

```bash
spark-local status
```

## Registry

The local spike reads [registry.json](./registry.json). That file currently maps
the blessed starter-stack module names to local repo paths on this machine.
