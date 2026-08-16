---
name: Configure and verify Octopod webhook deliveries
description: Register webhook endpoints for file validation and order completion, rotate the signing secret, verify the HMAC signature, and send a test delivery.
api: openapi/galatea-bio-octopod-openapi.yml
generated: '2026-08-16'
method: generated
source: https://docs.galatea.bio/#recipe-handling-webhooks-deliveries
operations:
  - notification_webhook-actions_list
  - organizations_webhooks-info_list
  - organizations_webhooks-info_create
  - organizations_generate-webhooks-secret_create
  - organizations_webhooks_test_create
  - notification_list
---

# Configure and verify Octopod webhooks

Webhooks are how you avoid polling. The Octopod Ancestry API emits two events, both signed with an
HMAC-SHA256 signature you must verify.

These endpoints are **organization-admin scoped** — a non-admin token gets `403`.

## Events

| Action value | Fires when |
|---|---|
| `source_file_validation_completed` | Validation of an uploaded source file finishes |
| `order_moved_to_completed_state` | An order reaches `Completed` or `Failed` |

Payloads:

```json
{
  "source_file_id": "7e53320c-c631-45ec-bad6-c88bb9d30960",
  "new_status": "VALID",
  "source_file_name": "d101979c-348c-44cd-8279-e9207b5a67a7.vcf.gz"
}
```

```json
{
  "order_id": "c1ea8279-25c2-4a36-9ea6-41e54239f17b",
  "new_status": "COMPLETED",
  "source_file_id": "7e53320c-c631-45ec-bad6-c88bb9d30960",
  "source_file_name": "d101979c-348c-44cd-8279-e9207b5a67a7.vcf.gz"
}
```

Both shapes carry `new_status`, `source_file_id` and `source_file_name`; only the order event adds
`order_id`. Discriminate on the presence of `order_id`, not on payload length.

## Steps

1. **Enumerate supported actions.** Call `notification_webhook-actions_list`
   (`GET /notification/webhook-actions`). It returns `name`/`value` pairs — use the `value` strings
   rather than hardcoding them.

2. **Read the current subscriptions.** Call `organizations_webhooks-info_list`
   (`GET /organizations/{organization_id}/webhooks-info`). Each entry has `id`, `webhook_action`,
   `remote_endpoint`, `max_retries_count`, `save_error_in_notifications` and `created_at`.

3. **Register or update endpoints.** Call `organizations_webhooks-info_create`
   (`POST /organizations/{organization_id}/webhooks-info`) with an **array** of subscription
   objects. Set `save_error_in_notifications: true` so failed deliveries are recoverable.
   Read the existing list first and send it back complete — this endpoint takes the full set.

4. **Mint the signing secret.** Call `organizations_generate-webhooks-secret_create`
   (`POST /organizations/{organization_id}/generate-webhooks-secret`). The secret is also visible in
   the console under Integrations → Webhooks. Store it as a secret; never log it or write it to a
   repository.

5. **Verify every delivery.** Compute:

   ```
   expected = base64(HMAC_SHA256(key = webhooks_secret, message = sender_host + raw_request_body))
   ```

   and compare it to the `X-Octopod-Signature` request header. `sender_host` is `api.galatea.bio` in
   production and `api.sandbox.galatea.bio` in sandbox.

   Sign over the **raw body bytes**, before any JSON parsing or re-serialization — re-encoding
   changes whitespace and breaks the comparison. Use a constant-time comparison. Reject with `422`
   when the header is absent or the signature does not match; the provider's reference handler does
   exactly this.

6. **Respond correctly.** Return HTTP `200` with an **empty body**. Anything else counts as a failed
   delivery and consumes the subscription's `max_retries_count` budget.

7. **Send a test delivery.** Call `organizations_webhooks_test_create`
   (`POST /organizations/{organization_id}/webhooks/{webhook_id}/test`). It sends published fake
   payloads, so you can validate the endpoint without running a real order:

   - file event: `source_file_id` `11111111-1111-1111-1111-111111111111`,
     file `test_message_source_file_validated.vcf.gz`
   - order event: `order_id` `22222222-2222-2222-2222-222222222222`,
     `source_file_id` `33333333-3333-3333-3333-333333333333`,
     file `test_message_order_completed.vcf.gz`

   Treat these UUIDs as test data and never write them into production records.

8. **Audit failures.** Call `notification_list` (`GET /notification`) to review saved delivery
   errors when `save_error_in_notifications` is enabled.

## Cautions

- Your endpoint must be publicly reachable over HTTPS.
- Deliveries are at-least-once with a retry budget — make the handler idempotent on `order_id` or
  `source_file_id`, because the API itself offers no idempotency key.
- Rotating the secret invalidates in-flight signatures; accept both old and new for a short window
  during a rotation.

See `asyncapi/galatea-bio-octopod-webhooks.yml`.
