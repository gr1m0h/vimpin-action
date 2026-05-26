# vimpin-action

A GitHub Action wrapper around the [`gr1m0h/vimpin`](https://github.com/gr1m0h/vimpin) CLI.

`vimpin` rewrites `lazy.nvim` Lua plugin specs to pin every plugin to an explicit commit hash; this action wraps the CLI so a workflow can verify pins, fail PRs that introduce drift, or open refresh PRs without the four-step install-and-run boilerplate.

> **Status:** scaffold. The action is published from this repo so its inputs and tags can evolve independently of the CLI's release cadence.

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
      - uses: gr1m0h/vimpin-action@v1
        with:
          mode: verify-strict
```

That replaces the four-step install-and-run pattern with a single dependency on this action.

## Inputs

| Input          | Default          | Description |
|----------------|------------------|-------------|
| `mode`         | `verify-strict`  | One of `verify`, `verify-strict`, `run-check`, `run`, `run-refresh` |
| `paths`        | (vimpin default) | Space-separated paths to scan; empty means use vimpin's discovery layout |
| `version`      | `latest`         | vimpin CLI version (pin to a tag for reproducible CI) |
| `fail-on-diff` | `false`          | For `mode: run` — fail the job if any files would change |
| `go-version`   | `1.24`           | Go toolchain version used to install the CLI |

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
      - uses: gr1m0h/vimpin-action@v1
        with:
          mode: run-check    # any unpinned spec ⇒ red
      - uses: gr1m0h/vimpin-action@v1
        with:
          mode: verify-strict  # drift / tag rewriting ⇒ red
```

Make both job runs required status checks on `main`.

### Scheduled refresh PR

```yaml
on:
  schedule: [{ cron: '0 9 * * 1' }]
  workflow_dispatch:
jobs:
  refresh:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: gr1m0h/vimpin-action@v1
        id: vimpin
        with:
          mode: run-refresh
      - if: steps.vimpin.outputs.changed == 'true'
        uses: peter-evans/create-pull-request@v6
        with:
          title: "chore(deps): vimpin --refresh"
          branch: vimpin-refresh
          commit-message: "chore(deps): refresh pinned commits"
```

## Versioning

This action follows a separate semantic version line from the underlying CLI:

- `@v1` — stable, additive-only changes
- `@v2`, `@v3`, ... — breaking input/output changes
- `@<sha>` — for maximum supply-chain safety (recommended for production CI)

Pin to `@<sha>` and use Dependabot to track action updates; the action will, in turn, pin a verified CLI release.

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
