---
name: Upload a genomic file and submit an analysis order
description: Upload a VCF source file to the Octopod Ancestry API, wait for validation, and submit an execution order against a named analysis model.
api: openapi/galatea-bio-octopod-openapi.yml
generated: '2026-08-16'
method: generated
source: https://docs.galatea.bio/#recipes-contents
operations:
  - users_me_list
  - organizations_models_list
  - data_files_upload_create
  - data_files_list
  - data_files_update
  - exec_orders_create
  - exec_orders_list
  - exec_tags_create
  - exec_orders_partial_update
  - exec_cancel_create
---

# Upload a genomic file and submit an analysis order

This is the core Octopod workflow: a source file goes in, an execution order runs a named analysis
model against it, and results come out. Everything is asynchronous.

**This operation is billable and handles identifiable human genomic data.** Order submission draws
down the organization's credit balance and cannot be undone by deleting the order. Confirm with a
human before submitting.

## Prerequisites

Hold a bearer token — see `skills/galatea-bio-authenticate.md`. Base URL
`https://api.galatea.bio/api/v1`, or `https://api.sandbox.galatea.bio/api/v1` for sandbox.

## Steps

1. **Discover which models you may run.** Call `users_me_list` (`GET /users/me`) and read
   `org.available_models`. For the full model list with deprecation status, call
   `organizations_models_list` (`GET /organizations/{organization_id}/models`) with
   `hide_deprecated=true`. Never submit a `model_name` that is not in this list.

2. **Upload the source file.** For files under 50 MB call `data_files_upload_create`
   (`POST /data/files/upload`) as `multipart/form-data` with the file in the `file` part. The
   response is the new file object; keep its `id`.

   For anything larger, upload over SFTP instead — the provider documents SFTP as the preferred
   path for any file size. SFTP uses a separate SSH-key credential managed through
   `organizations_sftp_users_list` and `organizations_sftp_users_ssh_keys_create`. After an SFTP
   upload the file appears through the normal list endpoint.

   File names may contain only letters, digits, spaces and the `-+_.` symbols.

3. **Wait for validation.** Validation is asynchronous. Either subscribe to the
   `source_file_validation_completed` webhook (see
   `skills/galatea-bio-configure-webhooks.md`), or poll `data_files_list` (`GET /data/files`)
   filtered with the `file` parameter set to the file id. The paginated envelope is
   `{count, next, previous, results}` — read `results[0]`. Do not submit an order until the file
   reports a valid status.

4. **Optionally set a sample alias.** `data_files_update`
   (`PUT /data/files/{source_file_id}`) sets `sample_alias`, which makes results easier to
   reconcile later.

5. **Optionally create a tag.** `exec_tags_create` (`POST /exec/tags`) with `{"name": ...}` returns
   a tag object. Collect the tag ids you want on the order.

6. **Submit the order.** Call `exec_orders_create` (`POST /exec/orders`) with:

   - `source_file_id` — the uploaded file's UUID
   - `model_name` — one of the organization's available models
   - `tags_ids` — array of tag UUIDs, or `[]`
   - `pdf_report_types` — only for PDF-producing models; see
     `skills/galatea-bio-request-pdf-reports.md`
   - `pdf_metadata` — optional object

   The response is an array; the order object is its first element. Keep the order `id`.

7. **Track the order.** Poll `exec_orders_list` (`GET /exec/orders`) with `filter` set to the order
   id, the source file id, or the source file name — that one parameter accepts all three. Narrow
   with `status`, `status_group`, `model_name`, `tags_ids`, `min_date` or `max_date`.

   Statuses run `Registered` → `Preparing` → `Submitted` → `Running` → `Model completed` →
   `Completed`. Terminal failure states are `Failed`, `Canceled`, `Reports failed` and `QC failed`.
   The coarse `status_group` values are `initializing`, `running`, `completed`, `failed`.

8. **Adjust or cancel if needed.** `exec_orders_partial_update`
   (`PATCH /exec/orders/{order_id}`) replaces the whole `tags_ids` set — send the complete list, not
   a delta. `exec_cancel_create` (`POST /exec/cancel`) with `{"order_id": ...}` cancels a run.

Then retrieve output — see `skills/galatea-bio-retrieve-results.md`.

## Cautions

- **Not idempotent.** Submitting the same file and model twice creates two orders and two charges.
  If a `POST /exec/orders` call times out, poll `exec_orders_list` with the file id before retrying.
- **Poll politely.** No rate limits are published and no `Retry-After` header is returned. Prefer
  webhooks over polling; when polling, use tens of seconds, not milliseconds.
- **Deleting a file does not cancel an order.** `data_files_delete` removes the source file only.
