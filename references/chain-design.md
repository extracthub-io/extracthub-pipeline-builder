# Designing the chain

## Contents
- How a run flows
- url_output_field
- url_template (ids and slugs)
- Fan-out and its cap
- Pagination
- What goes into the dataset
- Pins
- Limits of the chain form

## How a run flows
```
input_urls ──> step 1 (list extractor) ──url_output_field──> step 2 (detail extractor) ──> dataset
```
Every start URL runs step 1 once. Every value of step 1's `url_output_field` becomes one URL of step 2, and so
on. Steps run one after the other (stage by stage); the URLs of one step run in parallel.

## url_output_field
- Required on every step except the last. It names a field of THAT step's output, in the format
  `get_extractor` / `validate_pipeline` show (`output_fields`):
  - `nextPageUrl`: one string field;
  - `links`: an array of strings;
  - `books[].url`: the `url` of every item of the `books` list.
- The values should be absolute URLs (`toAbsoluteUrl` transform in the extractor). Relative ones
  (`/item/1`) are resolved against the page they came from.
- `validate_pipeline` rejects a field that is not in the output (`unknown_field`) and lists
  `available_fields`, URL-looking ones first. Never guess the name.
- No suitable field? Change the list extractor (extractor skill): add a `url` sub-field with
  `extract: "attributeValue"`, `attributeName: "href"` and `toAbsoluteUrl`, release, and use the new pin.

## url_template (ids and slugs)
When the field holds ids or slugs rather than URLs, build the URL with `url_template`:
```json
{ "extractor_id": "@me/products-api@1", "url_output_field": "products[].id", "url_template": "https://dummyjson.com/products/{value}" }
```
`{value}` is replaced by each value (URL-encoded). The template needs `{value}`.

## Fan-out and its cap
- One list page with 20 links = 20 runs of the next step. Two levels of 20 = 400.
- A step runs at most 1000 URLs per run (`PIPELINE_MAX_URLS_PER_STAGE`): the first 1000 run, the rest are
  dropped and the job shows the warning `fan_out_truncated` (with `received` / `dropped`). Split big jobs into
  several runs with fewer start URLs.
- `fan_out.enabled` / `aggregateResults` default to true; leave them unless the user asks otherwise.
- `skip_if_empty: true` skips a step that received no URLs instead of reporting it.

## Pagination
The chain has no loops: a step cannot feed itself. For "all pages of a category":
- put the page URLs in `input_urls` when they are predictable (`page-1.html` ... `page-5.html`), or
- run step 1 on the first page and use a second list step on `nextPageUrl` only for one extra page.
Tell the user when full pagination needs the dashboard or several runs.

## What goes into the dataset
Only the last step's records by default. Set `include_in_output: true` on another step to save its records too
(for example the list rows with their positions). The dataset groups records per step.

## Pins
Use released majors (`@user/slug@1`): they get fixes, never breaking changes. `@draft` works only for your own
extractors and changes with every edit (warning `draft_pin`). A deprecated major gives `deprecated_version`
with the sunset date: move to the newer major before then.

## Limits of the chain form
One start, one dataset, steps in a line, at most 10 steps and 100 start URLs per run. Pipelines with
branches or merges are built in the dashboard builder; `get_pipeline` then shows `chain: null` with the raw
flow, and `update_pipeline` can change everything except their steps.
