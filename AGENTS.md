# Agent guide

Read [docs/RULES.md](docs/RULES.md) first: the rules of every repository of the workspace, synced from dojo. [docs/DECISIONS.md](docs/DECISIONS.md) records what was decided and why.

This repository is the status page of peleda-app at userstatus.dev: a fork of [Upptime](https://upptime.js.org), driven entirely by GitHub Actions, Issues and Pages. It holds configuration (`.upptimerc.yml`) and the upstream workflows; it has no code of its own and never runs anything outside GitHub Actions.

Status: monitoring of userstatus.dev and the status website are configured; there is no Rust, no npm package and no `mise run check` task, since nothing here builds locally.

## Layout

- `.upptimerc.yml`: the monitored sites and the status website settings
- `.github/workflows/`: Upptime's workflows, kept as upstream generates them
- `assets/`: images for the status website
- `docs/`: rules and decisions

## Rules for this repository

- `LICENSE` is Upptime's MIT license and stays as upstream ships it.
- Workflows follow upstream Upptime; change them only through Upptime's own update mechanism or with a recorded decision.
