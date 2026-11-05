# Costs, runs and results

## Contents
- What is charged
- The estimate
- Working out a realistic cost
- maxCostPerRun and holds
- Running
- Reading get_pipeline_job
- Results

## What is charged
Every step call costs its extractor's price per URL (`get_extractor` pricing, `validate_pipeline`
`unit_price_usd`): `direct` < `premium` < `premium_js` (`get_account` shows the prices). A call is charged when
it succeeded, and also when it failed after the scraping provider fetched the page (blocked page, target 404,
extraction error); failures before that are free. A step that falls back to JavaScript rendering is charged
the `premium_js` price. Saving, validating, reading and cancelling pipelines are free.

## The estimate
`validate_pipeline` / `create_pipeline` return:
- `per_input_url_usd`: the sum of the step prices (one call per step);
- `estimated_cost_usd`: start URLs x `per_input_url_usd`, capped by `maxCostPerRun`. This is what
  `run_pipeline` holds when the run starts (the same number the dashboard uses), and what the balance must
  cover (`insufficient_credit` otherwise).

## Working out a realistic cost
The estimate does not know the fan-out. Multiply yourself:
```
cost per start URL ≈ price(step 1) + links per page x price(step 2) [+ links x sub-links x price(step 3)]
```
Example: a category page (direct, 0.0005) with 20 books, each a detail page (direct, 0.0005):
0.0005 + 20 x 0.0005 = 0.0105 USD per start URL. Count the links on your saved sample page or in the list
extractor's sample output. Tell the user this number, not only `estimated_cost_usd`.

## maxCostPerRun and holds
- `limits.maxCostPerRun` (USD, default 1) caps one run: when a stage would go over it the run stops
  (status `cost_limit_exceeded`), and what ran is charged.
- The run holds credit for the estimate when it starts and extends the hold as fan-out grows; unused credit
  is released at the end.
- Set `maxCostPerRun` to the realistic cost of one run plus a margin (for example 1.5x), never far above what
  the user agreed to.

## Running
- `run_pipeline` with `pipeline_id` and `input_urls` (1-100); without `input_urls` it uses the saved ones.
- The job starts `pending` (promoted at once) or `queued` (your concurrent run slots are busy; it starts when
  one frees up).
- Test keys cannot run pipelines (`live_key_required`): there is no sandbox for pipelines.
- `cancel_pipeline_job` stops a queued run (nothing charged) or a running one before its next step (what it
  already ran stays charged).

## Reading get_pipeline_job
- `status`: `queued`, `pending`, `running`, then `completed`, `failed`, `cancelled` or `cost_limit_exceeded`.
- `stages`: one entry per step with `runs`, `by_status` and the first `errors` (URL + message).
- `actual_cost_usd` vs `estimated_cost_usd`.
- `warnings`: `fan_out_truncated` (a step got more than 1000 URLs).
- `error_message`: why a failed run stopped.
Poll every 10-30 s (`retry_after_seconds`). Steps of a running job may lag a little behind the live progress.

## Results
When the run has finished, `results` has:
- `records`: the first 20 records per step, shortened to about 64 KB;
- `download_url`: the full dataset (JSON: `{ jobId, pipelineId, completedAt, totalCost, results: { <step>: [...] } }`),
  valid for 1 hour (call `get_pipeline_job` again for a new link). Anyone with the link can download it
  while it is valid: do not paste it into public places;
- `record_count`, `size_bytes`, `expires_at` (datasets are kept `retentionDays`, default 30).
Check the records before you call the pipeline done: fields filled, URLs right, no duplicates.
