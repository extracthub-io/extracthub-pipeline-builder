---
name: extracthub-pipeline-builder
description: Builds, tests, runs and schedules ExtractHub pipelines (multi-step scrapes that chain released extractors: a list or category page, then every detail page it links to, into one dataset). Use when the user wants an ExtractHub pipeline, to crawl a list then the detail pages, a multi-step scrape, to follow links from one page type to another, to run extractors over many pages in the background, or to schedule a recurring scrape with ExtractHub (with the ExtractHub MCP server).
---

# ExtractHub pipeline builder

An ExtractHub pipeline is a chain of released extractors: the input URLs go to step 1, every URL that step 1
outputs (one of its output fields) goes to step 2, and so on; the records end up in one dataset. Your job:
find or build the extractors, design the chain, validate it, tell the user what a run costs, create it, run it
small once with the user's OK, check the results, and only then schedule it (again with the user's OK).

Reference files (read when needed):
- [references/chain-design.md](references/chain-design.md): list -> detail chains, `url_output_field`, `url_template`, fan-out and its cap, output.
- [references/costs-and-runs.md](references/costs-and-runs.md): what a run costs, the estimate, holds, `maxCostPerRun`, reading job status and results.
- [references/schedules-and-webhooks.md](references/schedules-and-webhooks.md): cron schedules, the user-OK rule, webhooks and their secret.
- [examples/](examples/README.md): complete `create_pipeline` inputs (Books to Scrape category -> book detail; JSON API list -> detail).

Extractors themselves (writing selectors, validating on a sample page, releasing) are the job of the
**extracthub-extractor-builder** skill. Use it for every step whose extractor does not exist yet.

## Workflow

Copy this checklist and keep it updated:

```
Pipeline progress:
- [ ] 1. Page types of each step and the fields agreed with the user
- [ ] 2. One released extractor per step found or built (pins noted)
- [ ] 3. Chain designed (url_output_field per step) and validated with a small URL list
- [ ] 4. Estimate told to the user; pipeline created
- [ ] 5. One small run (user's OK), finished, results checked
- [ ] 6. Schedule / webhook set only if asked, schedule turned on only with the user's OK
```

### 1. Clarify (ask only what is missing)
- The start page(s): a category, search or list page URL, or an API list endpoint.
- What to collect from each level: usually the list gives links, the detail page gives the fields.
- How many start URLs and how often (one-off or recurring).

### 2. Find or build one extractor per step
For each page type: `search_extractors` (domain + a few words), then `get_extractor` for the candidates. Pick
one whose `output_schema` has the fields you need, and for every step except the last a field holding the
next step's URLs (`books[].url`, `items[].link`, `nextPageUrl`, or ids you can turn into URLs, see step 3).
Note its `recommended_pin` (`@user/slug@1`).

No fitting extractor -> build it with the **extracthub-extractor-builder** skill (validate on a saved sample,
create, test `@draft`, release). For a list step, make sure it outputs absolute URLs (`toAbsoluteUrl`) or ids.
Prefer released pins (`@1`) over `@draft`: a draft changes with every edit, and only its owner can run it.

### 3. Design the chain and validate it
Write the `create_pipeline` input (start from the closest file in [examples/](examples/README.md)):

```json
{
  "name": "Books: category -> detail",
  "input_urls": ["https://books.toscrape.com/catalogue/category/books/travel_2/index.html"],
  "steps": [
    { "extractor_id": "@your-username/books-category@1", "url_output_field": "books[].url" },
    { "extractor_id": "@your-username/book-detail@1" }
  ],
  "limits": { "maxCostPerRun": 1 }
}
```

Rules that matter most:
- Every `extractor_id` is pinned (`@1`, `@1.2.3`, or `@draft` for your own). Unpinned -> `pin_required` with the majors.
- Every step except the last needs `url_output_field`: a field of THAT step's output (`get_extractor` output
  schema; array items as `list[].field`). Values that are ids or slugs need `url_template` with `{value}`.
- Only the last step's records go to the dataset unless a step sets `include_in_output: true`.
- Linear chains only (one start, one dataset). Branches are built in the dashboard pipeline builder.

Call `validate_pipeline` with the input and `url_count` = the number of start URLs you plan to run. Fix every
error (paths like `steps[0].url_output_field`; `unknown_field` lists `available_fields`) and read the
warnings (`deprecated_version`, `draft_pin`). Validation is free and works with test keys.

### 4. Tell the user the estimate, then create
`validate_pipeline` returns `estimate.estimated_cost_usd` (one call per step per start URL, capped by
`maxCostPerRun`) and `per_input_url_usd`. Fan-out multiplies it: a category page with 20 books costs 1 + 20
calls per start URL. Work out the likely cost (references/costs-and-runs.md), say it, and suggest a
`maxCostPerRun` that covers one run with margin. Then `create_pipeline` with the same input (live API key;
test keys get `live_key_required`). It returns `pipeline_id` and `builder_url` (the dashboard builder opens it).

### 5. One small run, with the user's OK
Running spends credit. Ask first, then `run_pipeline` with `pipeline_id` and one or two `input_urls`.
`insufficient_credit` means nothing started: tell the user `required` / `available` and the top-up link.
Poll `get_pipeline_job` every 10-30 s until `status` is `completed`, `failed`, `cancelled` or
`cost_limit_exceeded`. Then check:
- `stages`: runs per step, failures with their URL and message;
- `warnings`: `fan_out_truncated` means a step got more than 1000 URLs and only the first 1000 ran;
- `results.records`: the first records per step; `results.download_url` (1 hour) has the full dataset.
Empty or wrong fields -> fix the extractor (extractor skill), release, `update_pipeline` with the new pin if the
major changed, and run small again. A run you started by mistake: `cancel_pipeline_job`.

### 6. Schedule and webhook (only if the user wants them)
Scheduled runs spend credit without anyone starting them. Never turn a schedule on without the user's
explicit OK, and state the cost per run and the cron in plain words first. `update_pipeline` with
`schedule: { "cronExpression": "0 6 * * *", "timezone": "UTC", "isActive": true }`. Any `schedule` without
`isActive: true` is saved OFF. Webhooks: `webhook: { "url": "https://...", "events": ["completed", "failed"] }`;
the signing secret is set in the dashboard, never through the tools. Details:
references/schedules-and-webhooks.md.

## Never
- Never put API keys in files that get committed (pipeline inputs, scripts, notes). Use the MCP server
  configuration or an environment variable.
- Never run a large run, or turn a schedule on, without the user's OK.
- Never guess `url_output_field`: take it from `get_extractor` / `validate_pipeline` (`available_fields`).

## Final answer to the user
Report: the pipeline id and builder URL, the steps (extractor pins), the start URLs, the result of the test run
(records, actual cost, failures), the cost per run you expect, and the schedule state (off, or on with the cron
the user approved).
