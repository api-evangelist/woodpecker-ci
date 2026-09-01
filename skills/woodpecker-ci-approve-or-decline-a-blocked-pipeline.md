---
name: woodpecker-ci-approve-or-decline-a-blocked-pipeline
description: Review a Woodpecker CI pipeline that is waiting for approval — typically one raised from a fork — and either approve it to start or decline it.
api: Woodpecker CI Server API
api_reference: https://woodpecker-ci.org/api
spec: openapi/woodpecker-ci-server-swagger.json
generated: '2026-08-27'
method: generated
source: openapi/woodpecker-ci-server-swagger.json, conventions/woodpecker-ci-conventions.yml
operations:
  - GET /repos/{repo_id}/pipelines                     # listRepositoryPipelines
  - GET /repos/{repo_id}/pipelines/{pipeline_number}   # getARepositoriesPipeline
  - GET /repos/{repo_id}/pipelines/{pipeline_number}/config    # getConfigurationFilesForAPipeline
  - POST /repos/{repo_id}/pipelines/{pipeline_number}/approve  # approveAndStartAPipeline
  - POST /repos/{repo_id}/pipelines/{pipeline_number}/decline  # declineAPipeline
operation_id_note: >-
  operationIds shown in comments come from overlays/woodpecker-ci-server-swagger-overlay.yaml;
  the provider's Swagger document declares none.
---

# Approve or decline a blocked Woodpecker CI pipeline

Woodpecker can hold a pipeline for human approval — the common case is a pull request from a
fork, where running the contributor's workflow file would execute untrusted code with the
repository's secrets in scope. `Repo.require_approval` and `Repo.approval_allowed_users`
control when this happens.

**This is a security decision, not a routine one.** An agent should gather the evidence and
present it; approving on a human's behalf is only appropriate when they have said so
explicitly for this pipeline.

## Steps

1. **Find what is waiting.**
   `GET /repos/{repo_id}/pipelines` and look for pipelines whose `status` indicates they are
   blocked. `Pipeline.from_fork` tells you whether the code came from outside the repository —
   this is the field that matters most.

2. **Read the run.**
   `GET /repos/{repo_id}/pipelines/{pipeline_number}` gives `author`, `sender`, `event`,
   `branch`, `commit`, `ref`, `changed_files`, `pr_draft`, `pr_labels` and `from_fork`.

3. **Read the workflow file that would run.**
   `GET /repos/{repo_id}/pipelines/{pipeline_number}/config` returns the resolved configuration
   files with their content and hash. This is the actual code about to execute. Look for
   changes to `.woodpecker/` in the same pull request, for steps that read secrets, and for
   image pulls from registries you do not recognise.

4. **Decide.**
   - Approve: `POST /repos/{repo_id}/pipelines/{pipeline_number}/approve` — this **starts the
     pipeline immediately**.
   - Decline: `POST /repos/{repo_id}/pipelines/{pipeline_number}/decline`.

## Reversibility

Approve and decline are the two halves of one decision, and they are mutually exclusive.
`decline` is the reversal of a pending approval — it is available *before* approval starts the
run, not after. Once approved, the only remaining lever is
`POST /repos/{repo_id}/pipelines/{pipeline_number}/cancel`, and any side effects the pipeline
has already produced are not undone by it. Woodpecker publishes no time window for either.

## Failure modes

The API returns plain text or an empty body. A `401` on these endpoints usually means the
token lacks push/admin on the repository, or the caller is not in
`Repo.approval_allowed_users` — check `GET /repos/{repo_id}/permissions` before concluding
that the token is invalid.
