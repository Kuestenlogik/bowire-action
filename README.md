# Bowire PR report — GitHub Action

Turns a pull request into an API report. The action runs [Bowire](https://github.com/Kuestenlogik/Bowire)
against the service built from the PR head and posts **one comment that updates
itself** on every push — API schema delta, test results, security findings and
latency deltas, each with its own configurable gate.

Everything happens inside the runner. Nothing is sent to Küstenlogik, and no
secret leaves the workflow.

```yaml
permissions:
  contents: read
  pull-requests: write     # required for the comment upsert

jobs:
  bowire:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      # Start the service the PR builds, however you normally do it.
      - run: docker compose up -d --wait

      - uses: Kuestenlogik/bowire-action@v1
        with:
          target: http://localhost:5080
```

That posts a report with the sections it can produce. Each one switches itself
off when the input it needs is missing, so the snippet above is a valid
starting point — add inputs to light up more of the report.

## What the sections need

| Section | Turned on by | Notes |
|---|---|---|
| **API delta** | `base-snapshot` | A snapshot taken on the base branch with `bowire diff snapshot`. See the caveat below. |
| **Tests** | `test` | Path to a recording or flow JSON for `bowire test`. |
| **Security** | `scan` (default `true`) | Add `scan-baseline` to show new vs. fixed findings instead of a flat list. |
| **Perf** | `test-baseline` | A base-branch JUnit XML from an earlier `bowire test --junit`. |

Gates are per section — `fail-on-schema`, `fail-on-tests`, `fail-on-scan`,
`fail-on-perf` — so a security regression can fail the check while a latency
wobble only annotates. The status column tells the two apart: 🟢 nothing to
report, 🟠 something was found and this gate lets it through (the cell names
the gate and its value), 🔴 the gate tripped and the check fails. A failed test
under `fail-on-tests: never` is 🟠, never 🟢. `perf-threshold-pct` and `perf-threshold-ms` both have to
be cleared before a latency move is reported, which keeps runner noise on very
fast tests out of the comment.

Run the action outside a pull request (say, `workflow_dispatch`) and the report
goes to the run summary instead of a comment.

## The API-delta section needs an unreleased CLI

`bowire diff` landed after the v2.4.0 tag, so the published
`Kuestenlogik.Bowire.Tool` does not have it yet. Until the next Bowire release,
the API-delta section only works with `build-from-source: true` and a checked-out
Bowire tree. Every other section works against the published CLI.

Once `bowire diff` ships, this note goes away and `build-from-source` returns to
being what it is meant for: testing the action against an unreleased CLI.

## Choosing a CLI

By default the action installs the latest published `Kuestenlogik.Bowire.Tool`.
Pin one with `tool-version` if you need reproducibility.

The action's version and Bowire's version are deliberately independent: `@v1`
keeps working across Bowire releases, and a Bowire patch never forces a new
action release.

## Versioning

Semver tags plus a moving `v1` alias. Pin `@v1` to get fixes automatically, or a
full tag like `@v1.0.0` to freeze.

## Outputs

| Output | Value |
|---|---|
| `overall-status` | `pass` or `fail` — the combined verdict across every enabled gate |
| `schema-status` | `pass` or `fail` for the API-delta section alone |
| `report-path` | Path to the assembled markdown report, for uploading as an artifact |

```yaml
      - uses: Kuestenlogik/bowire-action@v1
        id: report
        with:
          target: http://localhost:5080
      - run: echo "gate says ${{ steps.report.outputs.overall-status }}"
```

## Inputs

See [`action.yml`](action.yml) — every input carries its own description.

## License

Apache-2.0.
