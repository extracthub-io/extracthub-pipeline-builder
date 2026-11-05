# Example pipelines

Complete `create_pipeline` (and `validate_pipeline`) inputs. Replace `@your-username/...` with the pinned ids
of your own or public extractors (`search_extractors` / `get_extractor`), then validate before creating.

| File | Chain | Technique |
|---|---|---|
| [books-category-detail.json](books-category-detail.json) | Category page -> book detail pages | `url_output_field` on a list of objects (`books[].url`), only the last step in the dataset |
| [json-api-list-detail.json](json-api-list-detail.json) | JSON list endpoint -> JSON detail endpoint | ids turned into URLs with `url_template`, a schedule saved OFF |

## books-category-detail
Step 1 is the list extractor of the extractor skill's `books-list-static-html` example (`books[]` with
`title`, `url` (absolute, `toAbsoluteUrl`), `price`, ...). Step 2 is a detail extractor for
`/catalogue/<book>/index.html` pages (title, price, UPC, stock, description). A category page lists up to
20 books, so one start URL costs about 1 + 20 calls: with both extractors on the premium tier (proxy) that is
21 x 0.0025 = about 0.05 USD (check the real prices with `get_account`).

## json-api-list-detail
Step 1 is the extractor skill's `json-api-products` example: `products[*]` with `id`, `title`, ... The ids are
plain numbers, so `url_template` builds `https://dummyjson.com/products/{value}` for step 2 (a JSON extractor
with `sourceType: "json"` on the detail endpoint). The `schedule` has no `isActive`, so it is saved OFF:
turn it on later with `update_pipeline` only after the user agreed (references/schedules-and-webhooks.md).
