# vimpin-action

A GitHub Action wrapper around the [`gr1m0h/vimpin`](https://github.com/gr1m0h/vimpin) CLI.

`vimpin` rewrites `lazy.nvim` Lua plugin specs so every plugin is pinned to an explicit commit hash. This action wraps the CLI so a workflow can verify pins, fail PRs that introduce drift, or open update PRs without the four-step install-and-run boilerplate.

> **Status:** alpha. The action is published from this repo so its inputs and tags can evolve independently of the CLI's release cadence.

## Quick start

```yaml
name: vimpin
on: pull_request
permissions:
  contents: read
jobs:
  vimpin:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: gr1m0h/vimpin-action@b0f298ab...ef902e04
        with:
          mode: verify-check
```

That replaces the four-step install-and-run pattern with a single dependency on this action.

## Inputs

| Input          | Default        | Description |
|----------------|----------------|-------------|
| `mode`         | `verify-check` | One of `run`, `check`, `verify`, `verify-check`, `update`, `no-api` |
| `paths`        | (vimpin default) | Space-separated paths to scan; empty means use vimpin's discovery layout |
| `version`      | `v0.0.1`       | vimpin CLI version. Accepts `latest`, `main`, a semver tag, or a commit SHA. Pin to a tag for reproducible CI. |
| `fail-on-diff` | `false`        | For `mode: run` / `mode: update` — fail the job if any files would change |
| `go-version`   | `1.24`         | Go toolchain version used to install the CLI |

### Mode reference

| Mode           | Underlying CLI                | What it does |
|----------------|-------------------------------|--------------|
| `run`          | `vimpin run`                  | Initial pin (field-form → canonical). No-op on already-canonical specs. |
| `check`        | `vimpin run --check`          | Read-only. Exits non-zero if anything would change. |
| `verify`       | `vimpin run --verify`         | Reverse-resolves SHA, rewrites annotation if it drifted. SHA stays. |
| `verify-check` | `vimpin run --verify --check` | Read-only verify. Reports drift, exits non-zero. |
| `update`       | `vimpin run --update`         | Bump each spec to the latest semver tag (or branch HEAD). |
| `no-api`       | `vimpin run --no-api`         | Offline structural check. No network. |

## Outputs

| Output    | Description |
|-----------|-------------|
| `changed` | `"true"` if the chosen mode wrote (or would write) file changes; `"false"` otherwise |

## Recipes

### Required check on every PR

```yaml
on: pull_request
jobs:
  vimpin:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: gr1m0h/vimpin-action@b0f298ab...ef902e04
        with:
          mode: check          # any unpinned spec ⇒ red
      - uses: gr1m0h/vimpin-action@b0f298ab...ef902e04
        with:
          mode: no-api         # structural problem ⇒ red (offline, fast)
      - uses: gr1m0h/vimpin-action@b0f298ab...ef902e04
        with:
          mode: verify-check   # SHA ↔ annotation drift ⇒ red
```

Make these jobs required status checks on `main`.

### Scheduled update PR

```yaml
on:
  schedule: [{ cron: '0 9 * * 1' }]
  workflow_dispatch:
jobs:
  update:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: gr1m0h/vimpin-action@b0f298ab...ef902e04
        id: vimpin
        with:
          mode: update
      - if: steps.vimpin.outputs.changed == 'true'
        uses: peter-evans/create-pull-request@v6
        with:
          title: "chore(deps): vimpin --update"
          branch: vimpin-update
          commit-message: "chore(deps): bump pinned commits to latest tags"
```

## Versioning

This action follows a separate semantic version line from the underlying CLI:

- `@<sha>` (e.g. `@b0f298ab...ef902e04`) — for maximum supply-chain safety (recommended for production CI)
- `@v1`, `@v2`, ... — once tagged, major versions follow additive / breaking semantics respectively
- `@main` — tracks the latest action surface; useful while vimpin-action is alpha

Pin to `@<sha>` and use Renovate or Dependabot to track action updates; the action will, in turn, pin a verified CLI release.

## Relationship to vimpin

| Concern                       | Lives in `vimpin` | Lives in `vimpin-action` |
|-------------------------------|:----------------:|:------------------------:|
| Lua spec scanning / rewriting | ✓                | —                        |
| `git ls-remote` resolution    | ✓                | —                        |
| CLI entry points              | ✓                | —                        |
| Action input / output surface | —                | ✓                        |
| Composite-action wiring       | —                | ✓                        |
| Action release tags           | —                | ✓                        |

Bug reports about pinning behaviour go to `gr1m0h/vimpin`; bug reports about the Action wrapper go here.

## License

MIT
