# ci-workflows

Shared CI for grespi-tech repos. Every repo's `ci.yml` is a few lines that call
these, so CI behaves the same everywhere and changes land in one place.

Jobs run on a self-hosted runner pool (label `ci`). The pool's runners share a
CPU- and memory-capped cgroup so the host's other services keep priority.
Containers a job starts run outside that cgroup, so the workflows cap them with
`--cpus`/`--memory`.

The pool's vCPUs run single-threaded code about 2x slower than GitHub-hosted
runners. Tests with wall-clock budgets should multiply them by
`PERF_BUDGET_SCALE` and set it to `2` in the caller's `env`.

Callers pin an exact release tag (`@v1.0.1`). Tags are never moved: a fix is a
new tag, and callers bump to it. `v1` is superseded by `v1.0.1`, whose pnpm
install is per-job so concurrent jobs on the pool can't delete each other's
pnpm binary.

## node.yml: one pnpm job

```yaml
jobs:
  checks:
    uses: grespi-tech/ci-workflows/.github/workflows/node.yml@v1.0.1
    secrets: inherit                # only needed with secret-names
    with:
      name: Check & test
      postgres: postgres:17         # optional throwaway server on 127.0.0.1:$PGPORT
      postgres-databases: app app_test
      env: |
        DATABASE_URL=postgres://postgres:postgres@127.0.0.1:${PGPORT}/app
      secret-names: AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY
      run: |
        pnpm check
        pnpm test
```

Inputs: `run` (required), `name`, `cleanup` (runs even on failure), `env`
(`KEY=VALUE` lines, `${VAR}` expanded at runtime), `secret-names`,
`node-version` (22), `pnpm-version` (empty = `packageManager`), `install`,
`postgres`, `postgres-databases`, `runs-on`, `timeout-minutes`.

## playwright.yml: E2E in Microsoft's Playwright image

```yaml
jobs:
  e2e:
    uses: grespi-tech/ci-workflows/.github/workflows/playwright.yml@v1.0.1
    secrets: inherit
    with:
      image: mcr.microsoft.com/playwright:v1.57.0-noble   # must match @playwright/test
      shards: '[1]'
```

Inputs: `image` (required), `name`, `shards` (JSON list), `build`, `test`
(`--shard=i/n` appended), `env`, `secret-names`, `node-version`,
`pnpm-version`, `report-path`, `runs-on`, `timeout-minutes`.
