---
name: Request and download PRS and ancestry PDF reports
description: Submit an order that generates clinical or research-use PDF reports, then list and download them from the Octopod Ancestry API.
api: openapi/galatea-bio-octopod-openapi.yml
generated: '2026-08-16'
method: generated
source: https://docs.galatea.bio/#recipe-download-pdf-reports-of-order
operations:
  - exec_orders_create
  - data_results_request_pdf_create
  - data_results_pdf_report_list
  - data_results_download_list
  - data_pdfdata_list
  - data_results_pdf_report_change_notice_create
---

# Request and download PDF reports

Galatea Bio generates PDF reports for polygenic risk score and ancestry runs. There are two routes:
ask for them at order-submission time, or request them afterwards against a completed order.

**Clinical report types produce patient-facing clinical documents.** Treat `PRS_CLINICAL_CARDIO`
and `PRS_CLINICAL_CANCER` as human-authorized actions. Do not select a clinical report type on an
agent's own initiative when a research-use type would serve.

## Report types

| Type | Use |
|---|---|
| `PRS_RUO_CARDIO` | Cardiovascular PRS, research use only |
| `PRS_RUO_CANCER` | Cancer PRS, research use only |
| `PRS_CLINICAL_CARDIO` | Cardiovascular PRS, clinical |
| `PRS_CLINICAL_CANCER` | Cancer PRS, clinical |

## Route A — request reports with the order

Call `exec_orders_create` (`POST /exec/orders`) with `source_file_id`, `model_name`, and
`pdf_report_types` set to an array of the types above. Optionally pass `pdf_metadata`. The order
then moves through `Making report` and `Collecting report results` on its way to `Completed`; a
failure in this phase surfaces as `Reports failed`.

## Route B — request reports after the fact

Call `data_results_request_pdf_create`
(`POST /data/results/{exec_order_id}/request_pdf`) against a completed order. This creates a new
PDF request, tracked with its own `request_version`.

## Download the reports

1. **List what exists.** Call `data_results_pdf_report_list`
   (`GET /data/results/{exec_order_id}/pdf_report`). It is paginated (`page`, `page_size`) and
   accepts `request_version`, `report_version` and `sample_id` filters — use `sample_id` on
   multi-sample orders. Each entry carries a `pdf_request_id`.

2. **Download all reports as a ZIP.** Call `data_results_download_list`
   (`GET /data/results/{exec_order_id}/download`) with `result_type=PDF_REPORT` and no
   `pdf_request_id`.

3. **Download a single report.** Same call, adding `pdf_request_id` for the specific report.

In both cases the body is binary and the file name comes from the `Content-Disposition` header.

## Related

- `data_pdfdata_list` (`GET /data/pdfdata`) lists PDF data records across orders.
- `data_results_pdf_report_change_notice_create`
  (`POST /data/results/pdf_report/{pdf_request_id}/change_notice`) records a change notice against
  an issued report. This is a clinical-governance action — never invoke it without explicit human
  instruction.

## Cautions

- `pdf_report_types` is only meaningful for models that generate reports. Passing it to a model that
  does not produce PDFs will not create them.
- Requesting reports again creates a new `request_version` rather than replacing the previous one,
  so an unnecessary retry leaves two versions of a clinical document in the system. There is no
  idempotency key to prevent this — check `data_results_pdf_report_list` first.
