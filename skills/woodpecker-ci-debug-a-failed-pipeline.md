---
name: woodpecker-ci-debug-a-failed-pipeline
description: Diagnose a failed Woodpecker CI pipeline by walking its workflows and steps, pulling the failing step's logs, and reading the workflow configuration that produced it.
api: Woodpecker CI Server API
api_reference: https://woodpecker-ci.org/api
spec: openapi/woodpecker-ci-server-swagger.json
generated: '2026-08-27'
method: generated
source: openapi/woodpecker-ci-server-swagger.json, data-model/woodpecker-ci-data-model.yml
operations:
  - GET /repos/{repo_id}/pipelines                                   # listRepositoryPipelines
  - GET /repos/{repo_id}/pipelines/{pipeline_number}                 # getARepositoriesPipeline
  - GET /repos/{repo_id}/pipelines/{pipeline_number}/metadata        # getMetadataForAPipeline...
  - GET /repos/{repo_id}/pipelines/{pipeline_number}/config          # getConfigurationFilesForAPipeline
  - GET /repos/{repo_id}/logs/{pipeline_number}/{step_id}            # getLogsForAPipelineStep
  - GET /repos/{repo_id}/logs/{pipeline_number}/{step_id}/download   # downloadLogsForAPipelineStep
  - GET /stream/logs/{repo_id}/{pipeline}/{step_id}                  # streamLogsOfAPipelineStep
  - POST /repos/{repo_id}/pipelines/{pipeline_number}                # restartAPipeline
operation_id_note: >-
  operationIds shown in comments come from overlays/woodpecker-ci-server-swagger-overlay.yaml;
  the provider's Swagger document declares none.
---

# Debug a failed Woodpecker CI pipeline

## The object graph you are walking

`Repo` → `Pipeline` (numbered per repo) → `Workflow` → `Step`. Logs hang off a **step**, and
the step id is what the log endpoints want — not the workflow id, and not the step's `pid`.

## Steps

1. **Find the failure.**
   `GET /repos/{repo_id}/pipelines?page=1&perPage=50` and take the most recent pipeline whose
   `status` is a failure. `Pipeline.errors[]` (`errors.PipelineError`) often already names the
   problem — a configuration error surfaces here rather than in any step's logs, and if it is
   populated you can frequently stop at this step.

2. **Expand the run.**
   `GET /repos/{repo_id}/pipelines/{pipeline_number}` returns `workflows[]`, each with its own
   `state`, `error`, `platform`, `agent_id` and `children` (the steps). Find the first step
   whose `state` is a failure and note its `id` and `exit_code`. Work forward from the first
   failure, not backward from the last.

3. **Read that step's logs.**
   `GET /repos/{repo_id}/logs/{pipeline_number}/{step_id}` returns `LogEntry` records. Since
   3.18.0 the server enforces one entry per line with a unique index and returns them strictly
   in line order, so you can rely on the ordering. Use the `/download` variant when you want
   the whole thing as a file rather than a parsed array, and
   `GET /stream/logs/{repo_id}/{pipeline}/{step_id}` when the pipeline is still running.

4. **Read the configuration that ran.**
   `GET /repos/{repo_id}/pipelines/{pipeline_number}/config` returns the resolved workflow
   files. Compare against the current `.woodpecker/` in the branch — the pipeline ran what was
   in the tree at `Pipeline.commit`, not what is there now.

5. **Get the environment.**
   `GET /repos/{repo_id}/pipelines/{pipeline_number}/metadata` returns the metadata document,
   including previous-pipeline information, which is how you tell a newly-broken build from a
   long-broken one. The same document can be downloaded from the UI and replayed locally with
   `woodpecker-cli exec --metadata-file`, though the docs warn its format is only guaranteed
   for the same server and CLI version it came from.

6. **Reproduce before you re-run.**
   `woodpecker-cli exec --backend-engine docker .woodpecker/<file>` runs the workflow from a
   local checkout with no server, and `woodpecker-cli lint` validates it against the published
   JSON Schema (`json-schema/woodpecker-ci-pipeline-schema.json`). Secrets are not downloaded
   for local runs; pass them explicitly with `--secrets`.

7. **Re-run only when you have a reason.**
   `POST /repos/{repo_id}/pipelines/{pipeline_number}` restarts the pipeline. It is not
   idempotent and it consumes agent capacity. Its reversal is `.../cancel`.

## Gotchas

- `Pipeline.number` and `Pipeline.id` are different. Every URL takes the **number**.
- Log deletion is permanent. `DELETE /repos/{repo_id}/logs/{pipeline_number}` and the per-step
  variant have no undo and no retention window.
- Nothing in the API tells you *why* a step's container could not be pulled beyond the log
  text; check `Registry` configuration at repo, org and global scope if image pulls fail.
