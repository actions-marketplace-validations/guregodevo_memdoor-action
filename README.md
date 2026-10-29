# Memdoor

**AI code review, tests, failure explanations and issue triage in GitHub Actions, with no API key: the model runs inside the runner.**

One step in a job. No secret to add, no API account, no file in your repository, and nothing leaves the runner. The job passes or fails on a check, never because a model said so. When you want an agent that opens the files and runs the code to prove what it finds, give it your key instead: same step, one input changed.

## No key: a model inside the runner

```yaml
# .github/workflows/review.yml
on: pull_request
permissions:
  contents: read
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: guregodevo/memdoor-action@v1
        with:
          workflow: pr-review          # or security-review, pr-description, …
          local-model: Qwen3-8B-4bit
```

That is the whole setup. The action downloads [janis](https://github.com/guregodevo/janis) (one file, a local model runtime made for this), pulls `Qwen3-8B-4bit` once and caches it, runs it on the runner's 4 CPUs, and points [Memdoor](https://memdoor.ai) at it. A free `ubuntu-latest` runner is enough. The review lands in the job summary.

### What it wrote, on real pull requests

All from public runs on [llamafit](https://github.com/guregodevo/llamafit), a 3,000-line Go project, with `local-model: Qwen3-8B-4bit` and no key.

**`security-review`** on a pull request that added an HTTP handler reading a file named by a query parameter. The job failed:

> **P0** `models_http.go:15` — An attacker can exploit the `ServeModelFile` function to read arbitrary files on the server by leveraging the `os.ReadFile` function with a user-controlled `name` parameter, potentially leading to sensitive data exposure. The fix would be to validate and sanitize the `name` parameter to prevent directory traversal attacks.

The same kind of change done correctly passed ([run](https://github.com/guregodevo/llamafit/actions/runs/40869532725), [correct one](https://github.com/guregodevo/llamafit/actions/runs/40881880507)).

**`explain-failure`** after `go test` failed on a commit that changed a loop bound:

> **Cause** `gguf_test.go:252`
>
> **Why** The test failed because the calculated kvCachePerSlot value changed. The code was modified to iterate one less layer in the calculation, which led to a discrepancy between the actual cache size and the expected value.
>
> **Fix** Change the loop condition back to `i < m.Layers` in gguf.go line 154.

The loop is at line 157: a 9B gets the line close, a key gets it exact ([run](https://github.com/guregodevo/llamafit/actions/runs/40869605296)).

**`issue-triage`** on a new issue, "serve loads the smaller model when two GGUF files are in the folder":

> labels: bug
>
> **Kind** bug
>
> **Summary** The user reports that `llamafit serve` loads the smaller GGUF model (3B) when two models are present in the folder, despite expecting the larger one (8B) to be selected. They ask for the largest model to be prioritized.
>
> **Possible duplicates** none

([run](https://github.com/guregodevo/llamafit/actions/runs/40869547296))

**`pr-description`** on a pull request touching a workflow file and one Go file:

> **What**
> - `.github/workflows/memdoor-pr.yml` — add a new workflow for handling pull requests with memdoor
> - `picker.go` — update to match .gguf extensions case-insensitively
>
> **Why** not stated
>
> **How to test** Check that the memdoor-pr workflow runs correctly on pull requests. Verify that the picker handles both .gguf and .GGUF files.
>
> **Risk** Low - The changes are focused on improving file extension matching and adding a new CI workflow, which has minimal impact on existing functionality.

([run](https://github.com/guregodevo/llamafit/actions/runs/40881880507))

### What to expect from it

A 9B model on a free runner's CPU is a **free first pass, not a reviewer**: one model call with everything it needs in the prompt, no agent, nothing run to check a finding.

| On llamafit, `ubuntu-latest` | `local-model: Qwen3-8B-4bit` | your key (OpenRouter) |
|---|---|---|
| `pr-review`, an obvious bug (a loop that skips the last layer) | P0, the right reason, 2 of 2 | P0, the right reason |
| `pr-review`, a subtle bug (a path built from a lowercased name) | P0, the right reason, 2 of 2 | P0, proved by a test it wrote |
| `pr-review`, the same change done correctly | a false P0 in 1 of 2 runs | passed |
| `security-review`, a path traversal / done correctly | P0 / passed | P0 / passed |
| `explain-failure`, `issue-triage`, `pr-description` | right, as above | right, with more detail |
| `write-tests` | no test kept in ten tries (removed, job green) | passing tests; red when one shows the change is wrong |
| A job, model cached | 2 to 5 minutes | about 1 minute |

It catches real bugs and can fail a correct pull request, so read a local P0 before you believe it. Each local P0 gets a second question on the same model before it fails the job (is it a fault of the code after the change? does it give anyone access they lacked?): on llamafit that removed the false P0 from the correct change's security review, and from one of its two reviews. Writing tests needs the agent, on a key.

The model is cached per branch, and GitHub shares a cache saved on the default branch: run the workflow once on a push to `main` and every pull request starts warm. Any mlx-community model name works; `Qwen3.5-9B-4bit` is the quieter one (it passed the correct change and missed the subtle bug). Linux runners.

## More examples, all with no key

**Explain every failed CI run** (the logs are read with `gh` on the job's own token):

```yaml
on:
  workflow_run:
    workflows: [CI]
    types: [completed]
permissions:
  contents: read
  actions: read
jobs:
  explain:
    if: github.event.workflow_run.conclusion == 'failure'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: guregodevo/memdoor-action@v1
        with:
          workflow: explain-failure
          local-model: Qwen3-8B-4bit
```

**Triage every new issue** (suggests labels the repository has; applies nothing):

```yaml
on:
  issues:
    types: [opened]
permissions:
  contents: read
  issues: read
jobs:
  triage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: guregodevo/memdoor-action@v1
        with:
          workflow: issue-triage
          local-model: Qwen3-8B-4bit
```

`workflow_run` and `issues` jobs run from the file on the default branch: commit them to `main` first.

**Three checks on every pull request**, side by side:

```yaml
on: pull_request
permissions:
  contents: read
jobs:
  memdoor:
    strategy:
      fail-fast: false
      matrix:
        workflow: [pr-review, security-review, pr-description]
    name: ${{ matrix.workflow }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: guregodevo/memdoor-action@v1
        with:
          workflow: ${{ matrix.workflow }}
          local-model: Qwen3-8B-4bit
```

## With your key: an agent

```yaml
      - uses: guregodevo/memdoor-action@v1
        with:
          workflow: pr-review
          openrouter-api-key: ${{ secrets.OPEN_ROUTER_API_KEY }}
```

With a key the step is an agent: it reads the diff, opens the files around the change, runs the code and the tests to check each finding before it writes it down, and answers on any model you pick (`model:`). `anthropic-api-key`, `openai-api-key` and `gemini-api-key` work in place of OpenRouter. Trigger on `pull_request`, never `pull_request_target` ([why](#your-key)).

## Built-in workflows

Eight, none of them a file in your repository; each runs with no key or on your key.

| `workflow:` | Run it on | What it does | The job fails when |
|---|---|---|---|
| `pr-review` | `pull_request` | Reviews the change against its base branch and writes **P0** / P1 / P2 findings. With a key it opens the files around the change and runs the code to check a finding. | a P0: something that must not merge |
| `security-review` | `pull_request` | The same, for security only: secrets committed, injection, path traversal, missing authorization, unsafe deserialization, SSRF, weak crypto, an exploitable CI workflow. | an exploitable P0 |
| `write-tests` | `pull_request` | Writes tests for the functions the change touches (normal case, edge case, the change's own case) and leaves them in the working tree. Never edits an existing test. | an existing test already fails with the change, or a new test shows the change is wrong |
| `pr-description` | `pull_request` | Writes the pull request's description from the diff and the commits: title, What, Why, How to test, Risk. | — |
| `explain-failure` | `workflow_run` (`types: [completed]`, on failure) | Reads the failed run's logs (`gh run view --log-failed`) and the commit's diff, and writes the Cause (the failing step, file:line), Why, and the Fix. | — |
| `issue-triage` | `issues` (`types: [opened]`) | Suggests labels (only ones the repository has), the kind, a summary, possible duplicates among open issues, and what the reporter should add. Applies nothing. | — |
| `fix-tests` | `push` or by hand | Finds the test command (`go test`, `cargo test`, `npm test`, pytest, `make test`, or `MEMDOOR_TEST_COMMAND`), fixes the code until it passes, and leaves the fix in the working tree with a report. | the tests still fail |
| `release-notes` | a tag push (`partition: ${{ github.ref_name }}`) | Turns the commits since the previous tag into Added / Changed / Fixed, in words a user notices. | — |

The result is in the job summary. Nothing is committed, pushed or posted.

## Your own workflow

`workflow: <name>` runs `.memdoor/workflows/<name>/` from your repository: one YAML file per step (a command, a model call, or an agent turn), each done when its target exists. The same files run on your laptop with `memdoor workflow run <name>`; ask the Memdoor coder to write one, or see [Workflows](https://memdoor.ai/docs/workflows).

## Inputs

| Input | Default | |
|---|---|---|
| `workflow` | — | A built-in or your own workflow's name. **Required.** |
| `openrouter-api-key` | | `OPEN_ROUTER_API_KEY`: one key, every model on OpenRouter |
| `anthropic-api-key` / `openai-api-key` / `gemini-api-key` | | A vendor key instead (a key already in the job's env works too) |
| `model` | | Pin the model the agents answer with |
| `working-directory` | `.` | The project the workflow runs in |
| `timeout` | `2h` | A clock on the whole run, e.g. `45m` |
| `partition` | | Resume an earlier run, or a version for `release-notes` (`${{ github.ref_name }}`) |
| `stop-at-gate` | `true` | A workflow's approval gate ends the job with exit 3 (nobody in CI can approve); `false` waits for an approval from another door, bounded by `timeout` |
| `github-token` | the job's token | `GH_TOKEN` for the workflows that read GitHub (`explain-failure`, `issue-triage`); what it may do is what the job's `permissions` grant |
| `local-model` | | No key: a model inside the job, by its mlx-community name (`Qwen3-8B-4bit`), run with janis. A free first pass; see above |
| `install` | `true` | `false` uses the `memdoor` already on `PATH` |

## Outputs and exit codes

Outputs: `run-id`, `exit-code`, `results` (the files the run produced, one per line).

| Exit | Meaning |
|---|---|
| 0 | done |
| 2 | a step failed: a P0 in `pr-review` or `security-review`, a test showing the change is wrong in `write-tests`, tests still failing after `fix-tests` |
| 1 | stopped (timeout) |
| 3 | stopped at an approval gate |

## Your key

The agent runs your repository's code on the runner, where your key is. Trigger on `pull_request`, **never** `pull_request_target`: GitHub gives a fork's pull request no secrets, so a stranger's code never runs beside your key, while `pull_request_target` runs their code with your secrets. Memdoor also keeps model keys out of the environment of every command the agent or a workflow step runs, so an `env` or a test that dumps its environment cannot carry one into the model's context or the log; that is a second line, not the boundary.

## More

Linux and macOS runners. [Memdoor](https://memdoor.ai) is an open-source coding agent in one binary, on your own key: [github.com/guregodevo/memdoor](https://github.com/guregodevo/memdoor). Apache 2.0.
