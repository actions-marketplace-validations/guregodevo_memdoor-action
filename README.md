# Memdoor

Review pull requests, write their tests and descriptions, explain a failed run, triage new issues, fix failing tests and write release notes with an AI agent, as one step of a GitHub job, on your own model key, or with no key on a model inside the job. Nothing to add to your repository.

```yaml
on: pull_request
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: guregodevo/memdoor-action@v1
        with:
          workflow: pr-review
          openrouter-api-key: ${{ secrets.OPEN_ROUTER_API_KEY }}
```

The runner installs [Memdoor](https://memdoor.ai) (one binary), runs the workflow on your key, and the job ends with the run's outcome. The model never decides whether the job passes; a check does.

## Built-in workflows

Eight, each on your key or on a model inside the job (`local-model:`, below): with a key an agent reads the code around the change and runs it; with a local model each one is a single model call with everything it needs in the prompt.

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

```yaml
# explain a failed CI run
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
          openrouter-api-key: ${{ secrets.OPEN_ROUTER_API_KEY }}
```

`explain-failure` and `issue-triage` read GitHub with `gh` on the job's own token (`github-token`, `actions: read` / `issues: read`). Their triggers (`workflow_run`, `issues`) run the workflow file on the default branch: commit it to `main` before they fire.

### No key: a model inside the job

`local-model:` runs the review on a model inside the runner instead of a key: the action downloads [janis](https://github.com/guregodevo/janis) (one file, nothing installed), pulls and caches the model, and connects Memdoor to it. No secret, no API account, nothing leaves the runner. Linux runners.

```yaml
      - uses: guregodevo/memdoor-action@v1
        with:
          workflow: pr-review
          local-model: Qwen3-8B-4bit
```

**A free first pass, not a reviewer.** A free runner has 4 CPUs and 16 GB, so a 9B model is about the largest that fits, and it reviews in one model call (no agent, nothing run to check a finding). What we measured on `ubuntu-latest`, the same three pull requests on [llamafit](https://github.com/guregodevo/llamafit) reviewed each way:

| | `Qwen3-8B-4bit`, no key | your key (OpenRouter) |
|---|---|---|
| An obvious bug (a loop that skips the last layer) | P0, the right reason (2 of 2 runs) | P0, the right reason |
| A subtle bug (a path built from a lowercased file name) | P0, the right reason (2 of 2) | P0, proved by a test it wrote |
| The same change done correctly | **a false P0** (2 of 2): it filed the change's own fix as the fault; with the re-check below, 1 of 2 | passed |
| Review / job | 45 to 90 s / 2 to 3 minutes, most of it the model's download | about 40 s / about 1 minute |

It catches bugs, and it can fail a correct pull request: read its P0 before you believe it. Each P0 gets a second look on the same model (is it a problem of the code after the change, or before it?), and one about the code before is demoted to P2: on the correct change that turned one of two runs green; the other's "P0" was praise, which the question does not catch. `Qwen3.5-9B-4bit` is the quiet one (passed the correct change, missed the subtle bug). For a review that opens the files around the change and runs the code to prove a finding, use a key.

The other built-ins on the 8B, on GitHub's runners (llamafit [#15](https://github.com/guregodevo/llamafit/pull/15), [#16](https://github.com/guregodevo/llamafit/issues/16), [#17](https://github.com/guregodevo/llamafit/pull/17)): `security-review` failed a planted path traversal and passed the same kind of change done correctly (each of its P0s is asked once more whether the code after the change lets someone read, write or run something they could not before; a no makes it a P2). `explain-failure` named the failing test and the changed loop, `pr-description` lists each changed file, `issue-triage` labelled the issue `bug`. `write-tests` kept no test in ten tries (its tests compile but do not match the code; a test that does not pass is removed and the job stays green, after about a quarter of an hour): writing tests needs a key. The model (any mlx-community name) is cached per branch: GitHub shares a cache saved on the default branch, so run the workflow once on a push to `main` and every pull request starts from it.

### Your own workflow

`workflow: <name>` runs `.memdoor/workflows/<name>/` from your repository: one YAML file per step (a command, a model call, or an agent turn), each done when its target exists. Ask the Memdoor coder to write one, or see [Workflows](https://memdoor.ai/docs/workflows).

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
