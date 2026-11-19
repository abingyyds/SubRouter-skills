# Provider model listings, pricing and backup channels

For ticket conversations and optional reply images, see [Support ticket message images](support-tickets.md).

Use these APIs for the authenticated merchant's **own custom models**. Use `$BASE`, `$AUTH` and `$AUTH_USER` from `SKILL.md`; every call needs both the account `access_token` and `New-Api-User`. Do not substitute the model-call `sk-` key. Do not print credentials or channel secrets.

Read before writing. Show the exact model, proposed prices/currency, and channel changes and obtain explicit confirmation before a mutation. A request to inspect prices or list models is not permission to change them. Price preview does not save a model or invoke a paid model; channel tests and model probes are separate operations and are not required for configuring backups.

All IDs and prices in examples below are illustrative, not live IDs or official prices. Resolve them from the user's own API results. Check both HTTP status and `success`; an HTTP 200 is not proof that a new price is already active.

## Find models and channels

```bash
curl -s --max-redirs 0 "$BASE/api/provider/self" -H "$AUTH" -H "$AUTH_USER"
curl -s --max-redirs 0 "$BASE/api/provider/models?page=1&page_size=50" -H "$AUTH" -H "$AUTH_USER"
curl -s --max-redirs 0 "$BASE/api/provider/channels" -H "$AUTH" -H "$AUTH_USER"
```

The model list returns `data` as an array and `total` at the envelope root. The default page size is 10. Increment `page` with the same filters until all `total` models are collected; do not describe the first page as the whole catalogue. Supported filters include `keyword`, `source` and `channel_id`. If pages end before the reported total, report the incomplete result rather than silently assuming completeness.

`provider_channel_id` is the merchant-channel record's `id` from `/api/provider/channels`, **not** its internal system `channel_id`. Do not show the channel's key when summarizing the selection.

### Channel addresses and diagnostics

Use a public HTTP(S) base URL with authentication configured separately. Omit embedded credentials and URL fragments. For an OpenAI-compatible base URL, omit the query string too. These checks apply to saved configurations when used, including redirects, DNS resolution and proxy targets. A merchant channel cannot target localhost, private networks, link-local addresses or cloud metadata endpoints. Changing a global private-address setting does not grant a merchant exception.

Use `/api/provider/channels/<id>/fetch_models` and `/test` for the merchant's own channel when needed. A banned merchant cannot fetch models, run manual tests or run background probes. The merchant can still read configuration and historical earnings. A pending review or ordinary delisting alone does not block configuration and public-channel tests.

## Pricing contract

`billing_mode` selects `ratio` (per-token), `per_call`, or `tiered_expr`. `pricing_strategy` is separate:

| Strategy | Effect |
| --- | --- |
| `official_ratio` | For token billing, use the chosen `input_price` as the anchor and derive output/cache prices from the current official reference. Explicit output/cache prices are not independent overrides in this mode. |
| `custom` | Set input, output and cache prices independently. Use this when the user specifies the individual unit prices. |
| `legacy_frozen` | Existing contracts retain their historical ratios. Changing the input anchor scales output/cache prices using those saved ratios. Do not use this to create a new quotation or to edit one cache price independently. |

An official-reference change does not automatically rewrite a published merchant quotation. `regenerate_price: true` explicitly requests regeneration; for a legacy quote this opts into `official_ratio`. Preview it first and confirm every resulting price, since output/cache prices can increase even when the input anchor stays the same.

| Field | Meaning |
| --- | --- |
| `price_currency` | `CNY` or `USD`; explicitly send it for a new quote. Do not change only the currency label and assume the stored amounts are converted. |
| `input_price`, `output_price` | Currency units per **1,000,000 tokens**, not quota units or prices per token. Custom paid token models require both prices to be positive; free token models set both to 0. |
| `fixed_price` | Currency units per call for `per_call`; do not combine with positive input/output prices. |
| `cache_read_price` | Cache-read price per million tokens. It cannot exceed ordinary input price. |
| `cache_creation_price` | Cache-write price per million tokens, the 5-minute quote. Writes may cost more than ordinary input. |
| `cache_creation_price_1h` | Independent 1-hour cache-write price per million tokens. |
| `billing_expr`, `video_pricing` | Expression or structured video contract. Preserve the model's existing schema and units; do not infer that a video's `fixed_price` is a per-second price. |

Input/output/fixed prices are stored to 6 decimal places; cache prices to 10. Preserve the preview's precision instead of rounding cheap token prices to currency-display precision. Sending `billing_expr` without `video_pricing` clears a previous structured video quote; preview that transition rather than treating it as a metadata edit.

For new/custom token contracts, **0 is an explicit free cache price; `null` means no separate quote and those tokens are charged at ordinary input price**. Legacy-frozen contracts retain their historical behavior instead; use the preview to inspect a proposed strategy change. On updates, an omitted cache field retains its saved value (subject to the selected strategy); it is not the same as sending `null`. Send the cache-write durations separately rather than inventing an hour multiplier. Non-token contracts do not use these cache-price fields.

### Create a custom model

`POST /api/provider/models` creates a pending model for review. Example JSON body for a token-priced custom listing:

```json
{
  "model_name": "gpt-4o",
  "upstream_model": "gpt-4o",
  "provider_channel_id": 31,
  "category": "chat",
  "billing_mode": "ratio",
  "pricing_strategy": "custom",
  "price_currency": "USD",
  "input_price": 0.2,
  "output_price": 0.8,
  "cache_read_price": 0.05,
  "cache_creation_price": null,
  "cache_creation_price_1h": null
}
```

Use the actual upstream model ID, its final namespaced segment, or an allowed `-suffix` form as the public name. `model_suffix` is optional. A normal update cannot rename the model or change its declared upstream identity/suffix; create another listing for a different identity. Route-specific upstream targets are managed separately below.

For `official_ratio`, explicitly choose that strategy and provide the input anchor/currency rather than manually calculating the other prices. If new-model prices are omitted, the server may initialize from a reference/fallback; do not rely on that to express a price the user approved.

### Preview and update

`POST /api/provider/models/price-preview` accepts the proposed model fields. Include `id` when previewing an existing model. For a new model, omit `id` and pass its identity plus the intended pricing fields. The preview does not write a price, pending change, or notification.

Example: preview changing only a custom cache-read quote on model 101:

```json
{
  "id": 101,
  "pricing_strategy": "custom",
  "cache_read_price": 0.04
}
```

Inspect the returned prices, `pricing_strategy`, `price_contract_version`, `price_contract_reference`, `has_increase`, `needs_admin_approval` and `max_increase_ratio`. Preview uses the same contract-generation rules as save. If a pending quote exists, supplied pricing fields merge into that pending quote; show those other pending values before proceeding.

After user confirmation, send the same pricing changes to `PUT /api/provider/models/101` (the URL selects the model; `id` is unnecessary in this body):

```json
{
  "pricing_strategy": "custom",
  "cache_read_price": 0.04
}
```

There is **no all-or-nothing pricing-block requirement**: the server starts with the current/pending quote and applies the fields supplied. The pricing strategy can still derive linked prices, so a partial request is not a promise that just one unit price changes. Do not echo a whole GET result, `effective_*`, `pending_*` or server-generated contract metadata back as a proposed price.

For a metadata-only update such as a description, omit **every** pricing field, including `pricing_strategy`, `regenerate_price`, `billing_mode`, currency, cache fields and video pricing. This leaves the existing current/pending prices alone.

After saving, re-read the model list and distinguish current from `pending_*` prices:

- A price decrease with no component increase takes effect immediately.
- A recognized increase of at most 25% is scheduled 3 hours later.
- An increase over 25%, or an expression change whose price impact cannot be verified, requires administrator approval and then the 3-hour delay.
- Cache-price increases participate in that review too. `regenerate_price` does not bypass it. Unsupported billing-mode changes can be rejected rather than converted automatically.

Do not tell the user that the new price is active merely because `success` is true. Report the returned message, pending review status and effective time.

### Batch maintenance

`POST /api/provider/models/batch` accepts a top-level `provider_channel_id`, `update_existing` and `models` array; at most 200 models per request. Each row uses `model_name` (`model` is also accepted), identity and the same pricing fields/strategies as above. Set `update_existing: true` only when the user approved changing matching existing listings. Omitted pricing fields preserve existing quotes on updates; explicit 0/null must be preserved.

The response can contain partial success. Check `created`, `updated`, `pending`, `skipped` and the per-row `errors` under `data`; `success: true` alone is not proof every row was applied. Re-read each affected listing. Batch import is not a way to configure backups or bypass delayed/reviewed price increases.

### Convert an existing listing to merchant token prices

For an existing paid custom listing, the single-model edit and price preview with its ID reject a paid `tiered_expr` to `ratio` switch. Submit the complete token quote through `POST /api/provider/models/batch` with `update_existing: true`. The mode change goes through price review on the original listing.

1. Read the exact listing and `GET /api/provider/models/<id>/routes`. Record the listing ID, status, category and routes. Copy the stored `model_name` and `upstream_model`. Keep any suffix already present in `model_name`. If you also send `model_suffix`, use the original suffix. The listing's upstream identity can differ from the primary route's alias. Confirm that the listing exists and the supplied identity matches it because the batch endpoint can create missing listings. Copy the stored category to preserve the listing's classification.
2. Show the complete target quote and currency for approval. Send one row with that quote and omit the top-level `provider_channel_id` for a pricing-only change. Substitute the actual identity, category and approved prices in this example.

```json
{
  "update_existing": true,
  "models": [
    {
      "model_name": "example-chat",
      "upstream_model": "example-chat",
      "category": "chat",
      "billing_mode": "ratio",
      "billing_expr": "",
      "pricing_strategy": "custom",
      "price_currency": "CNY",
      "input_price": 5.44,
      "output_price": 27.2,
      "fixed_price": 0,
      "cache_read_price": 0.544,
      "cache_creation_price": null,
      "cache_creation_price_1h": null
    }
  ]
}
```

3. For one conversion, require `data.created: 0`, `data.updated: 0`, `data.pending: 1`, `data.skipped: 0`, and `data.errors: []`. On an unexpected result, read the listing before any retry. Verify the original ID, status and route configuration, plus `pending_billing_mode: "ratio"`, `pending_price_currency`, and the requested `pending_*` prices. A pending review has `pending_price_review_status: 1` and no scheduled activation yet.
4. The active quote remains in effect during review and the scheduled delay. After approval, use `pending_price_effective_at` for the activation time, normally three hours after approval. Read back the active billing mode and prices when due to verify that the token prices and display format took effect together.

## Ordered primary and backup channels

Use `GET /api/provider/models/<id>/routes` to read the current configuration, then `PUT` the **complete intended configuration** to the same path. This route-list replacement is different from a partial model/price update. Omitting an old target retires it.

The GET envelope's `data` contains `revision`, `routing_mode`, `feature_enabled` and `routes`. Copy the latest `revision` to `expected_revision`; keep existing target IDs when retaining them. Array order determines priority, not a client-supplied `priority` field. Example PUT body, using an illustrative revision of 7 and channels 31/32:

```json
{
  "expected_revision": 7,
  "routing_mode": "failover",
  "routes": [
    {
      "provider_channel_id": 31,
      "upstream_model": "openai/gpt-4o",
      "enabled": true
    },
    {
      "provider_channel_id": 32,
      "upstream_model": "gpt-4o",
      "enabled": true
    }
  ]
}
```

Apply these boundaries:

- 1–5 targets total: the first is the enabled primary, followed by up to 4 backups. They run in order, not as parallel requests or weighted load balancing.
- All channels must be enabled single-key channels owned by this merchant and compatible with the model. The same channel may appear with different upstream model names, but an identical channel/name pair cannot repeat. Send each target's complete upstream model ID without stripping its prefix.
- Existing target IDs cannot be repurposed for a different channel/model pair. For a different target, omit the old ID and add the new pair in the complete intended list.
- Only custom merchant listings support these routes, not official-Key or shared-subscription listings. Already auto-delisted models may require remediation first.
- Existing models remain `legacy` until the merchant explicitly chooses `routing_mode: "failover"`. Saving a legacy draft does not activate backups. Check `feature_enabled`; a platform-disabled feature cannot be enabled by this request. Identity-probe status is diagnostic, not a requirement to activate failover; do not launch paid validation calls.
- Once failover is active, change its primary through `/routes`, not by updating `provider_channel_id` on the ordinary model endpoint. To stop using backups, submit the full list with backups disabled (or removed); do not try to revert an active model to `legacy`.
- HTTP 409 `revision_conflict`: fetch the new configuration and reconcile the draft. Do not blindly retry with a new revision and overwrite concurrent changes. Other conflicts such as `failover_not_enabled` or remediation requirements need their own resolution.

A product keeps one frozen quotation across its primary/backup attempts; backups do not have independent sale prices. Failover only retries eligible failures within the request's budget. It is not a guarantee to retry all errors: content/input rejection, client cancellation, usable output, billable usage and uncertain accepted submissions retain their protections. Async media tasks are not transparently reassigned by this route feature. Do not promise that configuring backups fixes every Responses continuation or makes image retries safe.

After saving, GET `/routes` again and verify the returned revision, mode, order and enabled flags. Summarize the public model, merchant-channel names/IDs and upstream model IDs without exposing Keys, URLs with credentials or headers.
