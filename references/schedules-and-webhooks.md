# Schedules and webhooks

## Contents
- The user-OK rule
- Setting a schedule
- Turning it off
- Webhooks
- The webhook secret

## The user-OK rule
A scheduled run starts on its own and spends credit every time. Before you set `isActive: true`:
1. a test run has finished and the user has seen its results and actual cost;
2. you told the user the cron in plain words ("every day at 06:00 UTC"), the expected cost per run and per
   month, and the `maxCostPerRun` cap;
3. the user said yes to exactly that.
Without all three, save the schedule off (omit `isActive` or set it false) and tell the user how to turn it on.

## Setting a schedule
```json
{ "pipeline_id": "...", "schedule": { "cronExpression": "0 6 * * *", "timezone": "UTC", "isActive": true } }
```
- `cronExpression` has 5 fields: minute hour day-of-month month day-of-week (numbers, `*`, ranges `1-5`, lists
  `1,15`, steps `*/6`, names `MON-FRI` / `JAN`). It runs on the wall clock of `timezone` (an IANA name such as
  `UTC` or `Europe/Paris`, default `UTC`). An invalid cron or timezone is an `invalid_schedule` error.
- The first run is the next cron match after the save; `get_pipeline` shows it as `nextScheduledAt`.
- Scheduled runs use the pipeline's saved `input_urls` (save them with `update_pipeline`).
- `limits.maxRunsPerDay` (default 10) caps scheduled runs per day; `maxCostPerRun` caps each run.
- `get_pipeline` shows `schedule.isActive` and `nextScheduledAt` (null while the schedule is off).

## Turning it off
`update_pipeline` with `schedule: { "isActive": false }`. Any `schedule` object without `isActive: true` turns
the schedule off, so to change only the cron of an active schedule pass `isActive: true` again (with the
user's OK). `is_active: false` switches the whole pipeline off (no manual or scheduled runs).

## Webhooks
```json
{ "webhook": { "url": "https://hooks.example.com/extracthub", "events": ["completed", "failed"] } }
```
- https and a public host only (no localhost, private IPs or credentials in the URL).
- Events: `started`, `completed`, `failed`, `step_completed`, `step_failed` (default `completed`, `failed`).
- `webhook: { "url": null }` removes the URL.
- Payloads carry the job id, status, cost and the dataset id, not the records: fetch results with `get_pipeline_job`.

## The webhook secret
The signing secret is never set or shown through the tools. The user sets or rotates it in the dashboard
pipeline builder (Schedule panel); deliveries are then signed with `X-Webhook-Signature: sha256=<hex>`.
`get_pipeline` only tells whether one is set (`secret_set`).
