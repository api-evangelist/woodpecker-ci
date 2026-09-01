---
name: woodpecker-ci-trigger-and-follow-a-pipeline
description: Trigger a manual pipeline on a Woodpecker CI repository and follow it to completion, including cancelling it if it should not have run.
api: Woodpecker CI Server API
api_reference: https://woodpecker-ci.org/api
spec: openapi/woodpecker-ci-server-swagger.json
generated: '2026-08-27'
method: generated
source: openapi/woodpecker-ci-server-swagger.json, conventions/woodpecker-ci-conventions.yml
operations:
  - GET /repos/lookup/{repo_full_name}    # lookupARepositoryByFullName
  - POST /repos/{repo_id}/pipelines       # triggerAManualPipeline
  - GET /repos/{repo_id}/pipelines/{pipeline_number}   # getARepositoriesPipeline
  - GET /repos/{repo_id}/pipelines        # listRepositoryPipelines
  - POST /repos/{repo_id}/pipelines/{pipeline_number}/cancel   # cancelAPipeline
  - GET /stream/events                    # streamEventsLikePipelineUpdates
operation_id_note: >-
  The published Swagger 2.0 document declares no operationIds. The names in the comments
  above are the ones added by overlays/woodpecker-ci-server-swagger-overlay.yaml; the
  method+path pairs are what actually exist in the provider's contract.
---

# Trigger and follow a Woodpecker CI pipeline

## Before you start

- **The base URL is the operator's, not a vendor's.** Woodpecker is self-hosted. Ask for the
  server URL; do not assume one. `https://ci.woodpecker-ci.org/api` is the Woodpecker
  project's own instance and almost certainly not the one you want.
- **Auth**: every call below needs `Authorization: Bearer <personal access token>`. The token
  comes from the user's profile page on that server.
- **There is no idempotency key.** Calling `POST /repos/{repo_id}/pipelines` twice starts two
  pipelines. Never retry it blindly on a timeout — list pipelines first and check.
- **Errors are opaque.** Failures come back as a plain-text sentence or an empty body with
  only a status code. A `401` may mean "bad token" *or* "that repository does not exist",
  because authorization is checked before the resource is resolved.

## Steps

1. **Resolve the repository id.**
   `GET /repos/lookup/{repo_full_name}` with the `owner/repo` name. Every other call is keyed
   on the numeric `id` this returns — the API does not accept the full name anywhere else.

2. **Confirm you may write to it.**
   `GET /repos/{repo_id}/permissions` returns `pull`, `push` and `admin`. Triggering needs
   push. Checking first turns an unexplained 401 into a clear answer.

3. **Trigger the run.**
   `POST /repos/{repo_id}/pipelines`. Accepts a `PipelineOptions` body — use it to set the
   branch and any pipeline `variables`. Since 3.18.0 a custom message can be attached to a
   manual pipeline. Keep the returned `number`; it is the per-repository pipeline number that
   every follow-up call uses, and it is **not** the same as `id`.

4. **Follow it.**
   Poll `GET /repos/{repo_id}/pipelines/{pipeline_number}` and read `status`, `started`,
   `finished` and `workflows[]`. Woodpecker publishes no rate limits and returns no
   rate-limit headers, so choose your own interval — a few seconds is polite, and back off
   once the pipeline has been running for a while.
   For a push-based alternative, subscribe to `GET /stream/events`, a Server-Sent Event
   stream of pipeline updates. Note that the frame payload is **not** described in the
   contract, so you must be prepared for an undocumented shape.

5. **Reverse it if you were wrong.**
   `POST /repos/{repo_id}/pipelines/{pipeline_number}/cancel` while the pipeline is pending or
   running. There is no published time window — the constraint is the pipeline's state, not
   elapsed time. Once it has finished, cancel is no longer meaningful and the run cannot be
   undone; `DELETE /repos/{repo_id}/pipelines/{pipeline_number}` removes the record, not the
   effects.

## Pagination

Collection endpoints take `page` (default 1) and `perPage` (default 50) and return a **bare
JSON array** — no total, no next link. You know you have reached the end when a page comes
back empty or shorter than `perPage`.

## What this skill will not do

Nothing here touches secrets or registries. `Secret` carries a `value` field and `Registry`
carries a `password` field; treat both as out of scope for an agent unless a human has
explicitly asked.
