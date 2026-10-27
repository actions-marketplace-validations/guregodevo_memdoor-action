# Memdoor

Review pull requests, fix failing tests and write release notes with an AI agent, as one step of a GitHub job, on your own model key. Nothing to add to your repository.

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

| `workflow:` | What it does | The job fails when |
|---|---|---|
| `pr-review` | Reviews the pull request against its base branch, opens the files around it, verifies each finding by running the code, and writes a review with **P0** / P1 / P2 findings. | there is a P0: something that must not merge |
| `fix-tests` | Finds the test command (`go test`, `cargo test`, `npm test`, pytest, `make test`, or `MEMDOOR_TEST_COMMAND`), fixes the code until it passes, and leaves the fix in the working tree with a report. Skipped, no model called, when the tests already pass. | the tests still fail |
| `release-notes` | Turns the commits since the previous tag into Added / Changed / Fixed, in words a user notices. | — |

The review, the fix's report or the notes are shown in the job summary.

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
| `install` | `true` | `false` uses the `memdoor` already on `PATH` |

## Outputs and exit codes

Outputs: `run-id`, `exit-code`, `results` (the files the run produced, one per line).

| Exit | Meaning |
|---|---|
| 0 | done |
| 2 | a step failed (for `pr-review`, a P0) |
| 1 | stopped (timeout) |
| 3 | stopped at an approval gate |

## Your key

The agent runs your repository's code on the runner, where your key is. Trigger on `pull_request`, **never** `pull_request_target`: GitHub gives a fork's pull request no secrets, so a stranger's code never runs beside your key, while `pull_request_target` runs their code with your secrets. Memdoor also keeps model keys out of the environment of every command the agent runs, so an `env` or a test that dumps its environment cannot carry one into the model's context or the log; that is a second line, not the boundary.

## More

Linux and macOS runners. [Memdoor](https://memdoor.ai) is an open-source coding agent in one binary, on your own key: [github.com/guregodevo/memdoor](https://github.com/guregodevo/memdoor). Apache 2.0.
