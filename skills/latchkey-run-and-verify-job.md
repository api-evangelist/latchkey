---
name: latchkey-run-and-verify-job
description: >-
  Run one shell command on a fresh, isolated Latchkey runner over the REST Jobs API and read
  the verdict: create the job, optionally upload a context archive, submit it, poll status,
  and stream logs to completion. Use to verify a change on a clean Linux machine or run a
  build/test command without touching the local machine.
api: openapi/latchkey-jobs-api-openapi.json
operations: [createJob, submitJob, getJob, getJobLogs, cancelJob, listJobs]
generated: '2026-09-07'
method: generated
---

# Run and verify a job on a Latchkey runner

Every job runs ONE command on a fresh Ubuntu 24.04 x86_64 machine that is destroyed
afterwards. Base URL: `https://api.latchkey.dev`. Auth on every call:
`Authorization: Bearer lk_live_...` (a jobs-enabled key — `jobs:run` scope; `jobs:read`
suffices for status/logs only).

## Steps

1. **Create** — `POST /jobs` (`createJob`) with `{"command": "npm ci && npm test",
   "runner_size": "small"}`. Optional: `env` (max 64 keys), `timeout_seconds` (30–7200,
   default 1800), and `context_bytes` when you will upload a working tree. The response is
   `job_id` (and `context_upload_url` when `context_bytes` was set). Creating starts nothing
   and bills nothing.
2. **Upload context (only if requested)** — PUT the gzipped tar archive to
   `context_upload_url` with `Content-Type: application/gzip` before submitting. The URL is
   bound to the exact byte length you declared. Never pack secrets; pass them via `env`.
3. **Submit** — `POST /jobs/{id}/submit` (`submitJob`). Moves `created` → `queued`. A job not
   in `created` returns 409 and can never be resubmitted — on failure, create a new job.
4. **Poll status** — `GET /jobs/{id}` (`getJob`) until `state` is terminal (`succeeded`,
   `failed`, `cancelled`, `expired`). A submitted job that has not started within one hour is
   reported `expired`. `exit_code` and `failure_reason` appear once terminal.
5. **Stream logs** — `GET /jobs/{id}/logs?cursor=0` (`getJobLogs`); keep polling with the
   returned `next_cursor` until `complete` is true. Works while the job is still running.
6. **Cancel if needed** — `POST /jobs/{id}/cancel` (`cancelJob`). Idempotent; a running job
   stops within ~10 seconds. Cancellation is terminal.
7. **Audit later** — `GET /jobs?limit=20` (`listJobs`) lists recent jobs newest-first from
   durable storage (job status/logs themselves expire ~24h after completion).

## Rules

- Errors arrive as `{"error": "<description>"}`. 401 = bad/expired key; 403 = missing scope;
  429 = the 120 jobs/hour/workspace creation quota (no Retry-After header — back off).
- `createJob` has no idempotency key: a retried create makes a second job. Deduplicate on
  your side before retrying.
- Jobs bill per runner minute from command start to job end (small $0.0025/min … xlarge
  $0.02/min); queue and provisioning time is never billed. The exit code is the verdict.
- Linux x86_64 only; no inbound network to the runner; interactive commands hang until the
  timeout ends the job.
