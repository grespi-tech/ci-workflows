# ci-workflows

Shared CI for grespi-tech repos. Every repo's `ci.yml` is a few lines that call
these, so CI behaves the same everywhere and changes land in one place.

Jobs run on the org's self-hosted pool on the Hetzner box `universitydb`
(`udb-hetzner`): runners `udb-ci-1`, `udb-ci-2`, label `ci`, user `gha`
(uid 1001), installed under `/opt/gha-ci`. Both run in `gha-ci.slice`
(3 CPUs at low CPUWeight, 3 GB) so the box's production services keep priority. Containers a job
starts run outside that slice, so the workflows cap them with `--cpus`/`--memory`.

The box's vCPUs are shared 2.0 GHz Skylake: single-threaded code runs about 2x
slower than on GitHub-hosted runners. Tests with wall-clock budgets should
multiply them by `PERF_BUDGET_SCALE` and set it to `2` in the caller's `env`
(open-outbound-site does this).

Callers pin an exact release tag (`@v1.0.1`). Tags are never moved: a fix is a
new tag, and callers bump to it. `v1` (2026-10-26) is superseded by `v1.0.1`,
whose pnpm install is per-job so concurrent jobs on the pool can't delete each
other's pnpm binary.

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

## Adding a runner

```sh
TOKEN=$(gh api -X POST orgs/grespi-tech/actions/runners/registration-token --jq .token)
ssh udb-hetzner                                   # then, as root:
cp -al /opt/gha-ci/ci-1 /opt/gha-ci/ci-3 && rm -f /opt/gha-ci/ci-3/{.runner,.credentials*,.service}
rm -rf /opt/gha-ci/ci-3/_work /opt/gha-ci/ci-3/_diag
cd /opt/gha-ci/ci-3 && sudo -u gha ./config.sh --unattended --url https://github.com/grespi-tech \
  --token "$TOKEN" --name udb-ci-3 --labels ci --work _work
./svc.sh install gha && unit=$(cat .service) && mkdir -p /etc/systemd/system/$unit.d \
  && printf '[Service]\nSlice=gha-ci.slice\n' > /etc/systemd/system/$unit.d/slice.conf \
  && systemctl daemon-reload && systemctl restart $unit
```
