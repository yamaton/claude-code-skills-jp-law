---
name: egov-law-api
description: "e-Gov Law API v2 (https://laws.e-gov.go.jp/api/2) for programmatic
  access to Japanese law data. Covers article text retrieval, metadata search,
  full-text keyword search, revision history, attachment files (images/PDFs),
  point-in-time snapshots, and file format downloads (XML/JSON/HTML/RTF/DOCX).
  No authentication required."
---

# egov-law-api — e-Gov Law API v2

## Quick start

```bash
# No authentication required
curl 'https://laws.e-gov.go.jp/api/2/laws?limit=1&response_format=json'
```

Base URL: `https://laws.e-gov.go.jp/api/2`

Default response is JSON. For `/law_data`, also set `law_full_text_format=json` — mismatched formats cause Base64 encoding of the law text body.

## Endpoint selection

| Goal | Endpoint | Method |
|------|----------|--------|
| List/search laws by metadata | `GET /laws` | Query params for filtering |
| Revision history of a law | `GET /law_revisions/{law_id_or_num}` | Path = law_id or law_num |
| Law text (JSON or XML) | `GET /law_data/{id}` | Path = law_id, law_num, or revision_id |
| Attachment files (images, PDFs) | `GET /attachment/{revision_id}` | Optional `src` for specific file |
| Full-text keyword search | `GET /keyword` | `keyword` param required |
| Download as file (XML/JSON/HTML/RTF/DOCX) | `GET /law_file/{type}/{id}` | `type` = xml\|json\|html\|rtf\|docx |

## Common parameters

| Parameter | Endpoints | Purpose |
|-----------|-----------|---------|
| `response_format` | All | `json` (default) or `xml` |
| `asof` | `/laws`, `/law_data`, `/keyword`, `/law_file` | Point-in-time snapshot (YYYY-MM-DD, enforcement-date basis) |
| `elm` | `/law_data` only | Partial element extraction (see below) |
| `law_full_text_format` | `/law_data` only | `json` or `xml` for law text body |
| `limit` / `offset` | `/laws`, `/keyword` | Pagination (default limit=100). Response has `total_count` and `next_offset` |
| `order` | `/laws`, `/keyword` | Sort order. `+field` (asc) / `-field` (desc), comma-separated for compound sort. Default: `+law_info.law_id`. E.g. `order=-law_info.promulgation_date` |

## elm syntax — partial element extraction

Extract specific parts of a law via the `elm` parameter on `/law_data`:

| Target | elm value |
|--------|-----------|
| Main provision (全文) | `MainProvision` |
| Article 1 | `MainProvision-Article_1` |
| Article 1, Paragraph 2 | `MainProvision-Article_1-Paragraph[2]` |
| Part 2, Chapter 1 | `MainProvision-Part_2-Chapter_1` |
| Supplementary provision (1st set) | `SupplProvision[1]` |
| Appendix table (1st) | `AppdxTable[1]` |

Pattern: element names joined by `-`, numbered elements use `_N`, array indices use `[N]`.

## Format negotiation

`/law_data` has two format params: `response_format` (envelope) and `law_full_text_format` (law text body). **Set both to the same value** — when they differ, the law text is Base64-encoded.

## Gotchas

- **Consumed amending laws return 404.** Laws like `公職選挙法の一部を改正する法律` are unavailable after their provisions merge into the amended law. NDL 日本法令索引 API is one alternative for name/metadata lookup.
- **`asof` is enforcement-date based**, not promulgation-date. The API returns the revision that was in force on that date.
- **`asof` must be 2017-04-01 or later.** Earlier dates return HTTP 400 with code `400044`. The revision archive does not cover pre-2017 states.
- **`/keyword` default `sentence_text_size` is 100 chars.** Increase it (e.g. `sentence_text_size=500`) to see more context around matches.
- **Large laws may timeout.** Laws like 地方税法 (1,376 articles) can cause server-side timeouts. Use `elm` for partial extraction.
- **`Misc` law_type (告示/訓令/通達) is not served by v2 API.** `law_type=Misc` queries return 0 results.
- **`law_revision_id` overrides `asof`** on `/law_data`. If you specify both, `asof` is silently ignored.
- **No explicit rate limit, but throttle anyway.** The API has no documented rate limit, but bulk/rapid requests should be spaced out. For large batch workloads, consider the e-Gov XML bulk download instead of repeated API calls.

## Error handling

| HTTP | Code | Meaning | Action |
|------|------|---------|--------|
| 400 | `400001` | Parameter format error | Check parameter types/values |
| 400 | `400004` | Invalid date format | Use YYYY-MM-DD |
| 400 | `400044` | `asof` before 2017-04-01 | Use `asof` >= 2017-04-01 |
| 404 | — | Law not found or consumed | Try `law_id` instead of `law_num`; law may be a consumed amending law |
| 404 | `404003` | No attachment file | Verify `src` path from law XML `<Fig>` element |
| 500 | `500001` | Server error | Retry after delay; use `elm` for large laws |

## Full reference

Based on the official OpenAPI specification at <https://laws.e-gov.go.jp/api/2/swagger-ui>. Consult it for response schemas, full parameter enums, and category codes.
