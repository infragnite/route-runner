# route-runner

A GitHub Action that asks the Infragnite runner controller where a workflow's jobs should run:
on GitHub-hosted runners or on the controller's own pools.

The action reads the calling workflow and describes every job it can place before the run starts:
the job's name as GitHub shows it, its matrix rows, its `needs`, and the labels it may get. It sends
that to the controller's `/route/v1`, authenticated by the job's GitHub OIDC token, so no secret is
stored. The answer is a `runs-on` label list per job (per row key for a matrix job).

The answer never blocks CI. When the controller does not answer in time, refuses, or the job has
no OIDC token, the step warns and `runs-on` is `{}`, so every job keeps its own labels.

## Usage

```yaml
jobs:
  route:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    outputs:
      runs-on: ${{ steps.route.outputs.runs-on }}
    steps:
      - uses: actions/checkout@v4
      - id: route
        uses: infragnite/route-runner@v1
        with:
          workflow: .github/workflows/ci.yml
          asking: route
          # Values a matrix is built from, as JOB.OUTPUT=JSON lines.
          outputs: build.matrix=["2.4.0","2.4.10"]
```

## Asking when jobs become ready

An answer reflects the queues at the moment of the ask. Jobs that wait for a slow job (a build
the rest of the run needs) start long after the run does, so they should be placed by a later
ask: the gating job's last step asks for its dependents with `only`, and they read the gating
job's output.

```yaml
  bins:
    outputs:
      matrix: ${{ steps.matrix.outputs.matrix }}
      runs-on: ${{ steps.route.outputs.runs-on }}
    steps:
      # ... the build, then:
      - id: route
        uses: infragnite/route-runner@v1
        with:
          workflow: .github/workflows/ci.yml
          asking: bins
          only: conformance e2e
          outputs: bins.matrix=${{ steps.matrix.outputs.matrix }}
  conformance:
    needs: bins
    runs-on: ${{ fromJSON(needs.bins.outputs.runs-on || '{}').conformance[matrix.version] || 'ubuntu-latest' }}
```

Needs outside `only` were placed by an earlier ask and are not timed again.

In shadow mode (the controller's answer says `"mode": "shadow"`) the answer is each job's own
`runs-on`; the controller records its proposal beside where each job really ran.

## What is left out

A job keeps its own `runs-on`, and is named in the log, when it cannot be described before the
run starts: an `if:` other than `always()`, a matrix built from an output not given in `outputs`,
a matrix value that is not a scalar, a name or `runs-on:` that uses other expressions, or a job
that needs one of those.

## Inputs

| Input | Default | |
| --- | --- | --- |
| `workflow` | | Workflow file whose jobs are placed. |
| `asking` | | The asking job's own id; it is not placed. |
| `only` | | Whitespace-separated job ids to ask for, and no others; empty asks for every job. |
| `outputs` | | Newline-separated `JOB.OUTPUT=JSON` values a matrix is built from. |
| `url` | `https://runner-controller.infragnite.dev/route/v1` | The controller's route endpoint. |
| `audience` | `https://runner-controller.infragnite.dev` | The OIDC audience the controller accepts. |
| `timeout` | `10` | Seconds to wait for each call. |

## Outputs

| Output | |
| --- | --- |
| `mode` | `shadow` or `enforce`; empty without an answer. |
| `runs-on` | JSON object of labels by job key, and by row key (`v1\|v2`) for a matrix job; `{}` without an answer. |
