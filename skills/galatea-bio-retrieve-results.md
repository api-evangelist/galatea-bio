---
name: Retrieve ancestry and PRS results for a completed order
description: Determine which result types a completed Octopod order produced, then download them as files or as JSON.
api: openapi/galatea-bio-octopod-openapi.yml
generated: '2026-08-16'
method: generated
source: https://docs.galatea.bio/#recipe-get-results-of-order
operations:
  - exec_orders_list
  - data_results_download_list
  - data_results_json_list
  - data_prsdata_list
---

# Retrieve results for a completed order

Results are not a standalone resource. They are always addressed through their order id and a
`result_type` selector.

## Steps

1. **Confirm the order finished and learn what it produced.** Call `exec_orders_list`
   (`GET /exec/orders`) with `filter` set to the order id. Read `results[0]`:

   - `status` must be `Completed`. If it is `Failed`, `Reports failed` or `QC failed`, download the
     `EXEC_ERRORS` result instead of the data results.
   - `result_types` is the authoritative list of what is actually available for this order. Do not
     guess a result type — read this array and download only what it names.

2. **Download a result file.** Call `data_results_download_list`
   (`GET /data/results/{exec_order_id}/download`) with the `result_type` query parameter. The
   response body is binary; the file name is in the `Content-Disposition` header's `filename="..."`
   parameter. Parse it from there rather than inventing one.

3. **Or read a result as JSON.** Call `data_results_json_list`
   (`GET /data/results/{exec_order_id}/json`) with `result_type`. JSON is only available for the
   summary and detailed types — `SUMMARY_SUPERSET`, `SUMMARY_CHROMS`, `DETAILED_SUPERSET`,
   `DETAILED_CHROMS`. Everything else must go through the download endpoint.

## Result types

| Result type | What it is |
|---|---|
| `SUMMARY_SUPERSET` | Ancestry summary at superpopulation level |
| `SUMMARY_CHROMS` | Ancestry summary per chromosome |
| `DETAILED_SUPERSET` | Detailed ancestry at superpopulation level |
| `DETAILED_CHROMS` | Detailed ancestry per chromosome |
| `WHOLE_RESULT` | Complete result bundle |
| `CHROMS_SVG` | Chromosome painting as SVG |
| `PDF_REPORT` | Generated PDF report |
| `PRS_DATA` | Polygenic risk score data |
| `PRS_TECH_DATA` | PRS technical data |
| `PRS_QC` | PRS quality control output |
| `EXEC_ERRORS` | Execution errors for a failed order |
| `UNKNOWN` | Unclassified output |

For PRS runs, `data_prsdata_list` (`GET /data/prsdata`) lists PRS data records across orders.

## Error handling

- `404` — the order id does not exist or does not belong to your organization.
- `406` — usually a malformed identifier (`{"detail": "Incorrect ID"}`), not content negotiation.
  Verify the value is a UUID v4 before retrying.
- `403` — your role cannot read this order.

## Cautions

- **Only request result types listed in the order's `result_types`.** Asking for one the model did
  not produce is an error, not an empty result.
- **These payloads are identifiable human genomic data** produced under a CLIA/CAP-regulated
  workflow. Do not write them to shared or unencrypted locations, and do not include their contents
  in logs or model context without explicit authorization.

For PDF reports specifically, see `skills/galatea-bio-request-pdf-reports.md`.
