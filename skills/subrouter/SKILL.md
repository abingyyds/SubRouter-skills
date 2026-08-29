---
name: subrouter
description: 'Connect an AI agent (Claude Code, Cursor, Codex, etc.) to SubRouter — an OpenAI/Anthropic-compatible AI API gateway with smart routing and transparent pricing. Use this skill to authorize the agent via device flow, obtain an API key, pick a model by price, and configure the client. Triggers: "接入 SubRouter", "配置 SubRouter", "SubRouter API key", "用 SubRouter 的模型".'
---

# SubRouter Agent Onboarding

SubRouter is an AI API gateway: one API key, OpenAI- and Anthropic-compatible endpoints, per-request smart routing across providers, transparent per-token billing. This skill teaches an agent how to connect a user to SubRouter end to end.

Base URL: use the site the user names, or the default `https://subrouter.ai`. All examples below use `$BASE`.

## Security rules (non-negotiable)

- Send the API key ONLY to the SubRouter host the user chose. Never to any other domain.
- Always call with `--max-redirs 0` so a redirect can never carry the key elsewhere.
- Never print the full key into logs or files other than the client config the user asked for. Show at most a `sk-xxxx…` prefix when confirming.
- Payment is ALWAYS done by the user in their browser. Never handle card numbers, crypto seeds, or payment credentials.

## Step 1 — Device authorization (get an API key)

Start a device authorization. `channel` identifies your agent type (`claude`, `cursor`, `codex`, …):

```bash
curl -s --max-redirs 0 -X POST "$BASE/api/device_auth" \
  -H 'Content-Type: application/json' \
  -d '{"channel":"claude"}'
```

Response `data`:

```json
{
  "user_code": "ABCD-2345",
  "device_code": "<64-char secret, keep private>",
  "verification_url": "https://subrouter.ai/device?code=ABCD-2345",
  "expires_in": 600,
  "interval": 3
}
```

Show the user the `verification_url` and the `user_code`, and tell them to open it in a browser, log in (or register), check that the page shows the same code, and confirm. The code expires in 10 minutes.

Then poll every `interval` seconds (never faster):

```bash
curl -s --max-redirs 0 -X POST "$BASE/api/device_auth/poll" \
  -H 'Content-Type: application/json' \
  -d '{"device_code":"<device_code>"}'
```

- `{"status":"authorization_pending"}` — keep waiting.
- `{"status":"slow_down"}` — you polled too fast; wait longer.
- `{"status":"approved","key":"sk-…","access_token":"…","base_url":"…"}` — done. Delivered exactly once; store both immediately.
- `success:false` with "invalid or expired" — the flow expired or was already consumed; start over from Step 1.

You receive TWO credentials with different powers:

- `key` (`sk-…`) — the **relay token**: model calls, `/v1/models`, billing lookups. Put this in client configs.
- `access_token` — the **account management credential**: full management API access under the user's account. Never put this in a client config; use it only for the management calls below, and guard it even more carefully than the sk key.

## Step 2 — Configure the client

The same key works on both protocol surfaces.

Claude Code (Anthropic protocol):

```bash
export ANTHROPIC_BASE_URL="$BASE"
export ANTHROPIC_API_KEY="sk-…"
```

OpenAI SDK / Cursor / Codex (OpenAI protocol):

```bash
export OPENAI_BASE_URL="$BASE/v1"
export OPENAI_API_KEY="sk-…"
```

Core endpoints: `POST /v1/chat/completions` (OpenAI), `POST /v1/messages` (Anthropic), `GET /v1/models`.

The sk key also serves the full multimodal surface (availability depends on the model — check `/api/pricing`):

- Images and video: see "Step 2b" below — **use the async task API**, not the synchronous endpoints
- Audio: `POST /v1/audio/transcriptions`, `/v1/audio/speech`, `/v1/audio/translations`
- Other: `POST /v1/embeddings`, `/v1/rerank`, `/v1/moderations`, `/v1/responses`, `GET /v1/realtime` (WebSocket)

## Step 2b — Images and video: use the async task API

Image and video generation take tens of seconds to several minutes. A synchronous call holds one HTTP connection for that whole time, which is what makes it fail: proxies and CDNs in front of the gateway time out at 504, and running several in parallel exhausts connections and produces 502s. **Default to async.**

| Case | Use |
|---|---|
| Video, any model | Async — always. Never call video synchronously. |
| Images, more than one at a time | Async |
| Images, batch or unattended | Async |
| A single image, user is waiting interactively | Sync is acceptable |

### Submit

Images — append `/async` to the normal path:

```bash
curl -s --max-redirs 0 -X POST "$BASE/v1/images/generations/async" \
  -H "Authorization: Bearer sk-…" -H 'Content-Type: application/json' \
  -d '{"model":"<image-model>","prompt":"…","n":1,"size":"1024x1024"}'
```

`POST /v1/images/edits/async` works the same way.

Answers `202 Accepted` with a `Location: /v1/tasks/{task_id}` header and a `media.task` body:

```json
{"id":"…","object":"media.task","type":"image","action":"generate",
 "status":"pending","progress":0,"model":"…","created_at":1756400000,
 "result":{"images":[],"videos":[]},"usage":{"quota":1234},"error":null}
```

The `/async` facade always yields a task, even when the upstream merchant only supports synchronous calls — the gateway runs it in the background for you.

Video — `POST /v1/video/generations` is already asynchronous; the request body follows the model's own schema. The response is the platform's native submit shape, but the task id in it is SubRouter's, so poll it the same way. `POST /v1/videos` is the OpenAI-compatible alias.

### Poll

One endpoint covers both image and video tasks:

```bash
curl -s --max-redirs 0 "$BASE/v1/tasks/<task_id>" -H "Authorization: Bearer sk-…"
```

`status` is one of `pending`, `processing`, `completed`, `failed`. Poll no faster than every 2–3 seconds, and back off for long video jobs. On `completed` read `result.images[].url` / `result.images[].b64_json` for images and `result.videos[].url` for video; on `failed` read `error.message`. `result` always carries both arrays — the one that does not match the task type stays empty rather than null.

Native per-platform poll paths also exist (`GET /v1/images/generations/{task_id}`, `GET /v1/video/generations/{task_id}`, `GET /v1/videos/{task_id}`) — prefer `/v1/tasks/{task_id}`, whose shape is identical across every model.

### Download the video

```bash
curl -fL "$BASE/v1/videos/<task_id>/content" -H "Authorization: Bearer sk-…" -o out.mp4
```

Upstream video URLs are often short-lived or geo-restricted; this endpoint streams the file through the gateway with the sk key. Use it instead of fetching `result.videos[].url` directly.

### Listing past tasks

`GET /api/task/self` (with the access token) lists the user's media tasks — useful for recovering a task id the client lost.

## Step 3 — Pick a model (and optionally a provider)

Prices and the provider marketplace are public and need no auth:

```bash
curl -s --max-redirs 0 "$BASE/api/pricing"                        # models + prices
curl -s --max-redirs 0 "$BASE/api/marketplace/models"             # marketplace models with provider offers
curl -s --max-redirs 0 "$BASE/api/marketplace/providers"          # provider (商家) list
curl -s --max-redirs 0 "$BASE/api/marketplace/providers/<slug>"   # one provider's detail
curl -s --max-redirs 0 "$BASE/api/marketplace/reviews"            # provider reviews
```

Choose by the user's stated preference (cheapest, specific vendor/provider, capability). Present the top candidates with prices and let the user pick; do not guess silently.

## Step 4 — Top up (user does this, not you)

The account starts with no balance. Send the user to `$BASE/console/topup` and wait. Check balance and usage with the sk key itself:

```bash
curl -s --max-redirs 0 "$BASE/v1/dashboard/billing/subscription" -H "Authorization: Bearer sk-…"
curl -s --max-redirs 0 "$BASE/v1/dashboard/billing/usage" -H "Authorization: Bearer sk-…"
```

## Account management (uses `access_token`)

Management endpoints authenticate with `Authorization: Bearer <access_token>` (the `Bearer ` prefix is optional). Examples:

```bash
AUTH='Authorization: Bearer <access_token>'

# Account info & balance
curl -s --max-redirs 0 "$BASE/api/user/self" -H "$AUTH"

# Usage reconciliation ("where did my money go")
curl -s --max-redirs 0 "$BASE/api/log/self?p=1&page_size=20" -H "$AUTH"   # per-call logs
curl -s --max-redirs 0 "$BASE/api/log/self/stat" -H "$AUTH"               # totals
curl -s --max-redirs 0 "$BASE/api/data/self" -H "$AUTH"                   # per-day quota usage
curl -s --max-redirs 0 "$BASE/api/task/self" -H "$AUTH"                   # async media tasks

# Redeem a voucher code the user already has (this is NOT payment)
curl -s --max-redirs 0 -X POST "$BASE/api/user/self/topup" -H "$AUTH" \
  -H 'Content-Type: application/json' -d '{"key":"<redemption-code>"}'

# Referral: the user's invite code and earnings
curl -s --max-redirs 0 "$BASE/api/user/self/aff" -H "$AUTH"               # invite code → share $BASE/register?aff=<code>
curl -s --max-redirs 0 "$BASE/api/user/self/aff_dashboard" -H "$AUTH"
curl -s --max-redirs 0 "$BASE/api/user/self/aff_earnings" -H "$AUTH"
# Move referral earnings into the usable balance (confirm amount with the user first)
# NOTE: quota is in internal units, 500000 = $1 — never pass a dollar number directly
curl -s --max-redirs 0 -X POST "$BASE/api/user/self/aff_transfer" -H "$AUTH" \
  -H 'Content-Type: application/json' -d '{"quota":<amount-in-quota-units>}'

# List / create API tokens
curl -s --max-redirs 0 "$BASE/api/token/" -H "$AUTH"
curl -s --max-redirs 0 -X POST "$BASE/api/token/" -H "$AUTH" \
  -H 'Content-Type: application/json' \
  -d '{"name":"my-project","expired_time":-1,"unlimited_quota":true}'
```

### Model- or provider-scoped keys

A token can be restricted to specific models, to specific providers (商家) per model, and to a price ceiling. Use this when the user says "只用某商家" or "只允许这个模型":

```bash
curl -s --max-redirs 0 -X POST "$BASE/api/token/" -H "$AUTH" \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "sonnet-via-acme",
    "expired_time": -1,
    "unlimited_quota": true,
    "model_limits_enabled": true,
    "model_limits": "claude-sonnet-4-6,claude-haiku-4-5",
    "subrouter_model_providers": "{\"claude-sonnet-4-6\":[\"acme\"]}"
  }'
```

- `model_limits` — comma-separated model whitelist (with `model_limits_enabled: true`).
- `subrouter_model_providers` — a JSON **string** mapping model name → allowed provider slugs; slugs come from `GET /api/marketplace/providers`. Models not listed keep normal routing.
- `subrouter_model_price_limits` — optional per-model input/output price caps (JSON string as well).

Echo the exact restrictions back to the user before creating the key.

### Becoming a provider (商家) and listing models

The user's account can apply to become a provider and then manage its own channels and models — all with the same `access_token`:

```bash
# Apply to become a provider (platform review may apply)
curl -s --max-redirs 0 -X POST "$BASE/api/provider/register" -H "$AUTH" \
  -H 'Content-Type: application/json' -d '{...}'

# After approval, provider management lives under /api/provider/*
curl -s --max-redirs 0 "$BASE/api/provider/self" -H "$AUTH"      # provider profile
curl -s --max-redirs 0 "$BASE/api/provider/models" -H "$AUTH"    # listed models
# List a model: POST /api/provider/models — GET first to learn the field shape
# Channels: /api/provider/channels (GET/POST/PUT/DELETE)

# Bulk model maintenance (e.g. syncing upstream price changes)
# POST /api/provider/models/batch          — create many models at once
# POST /api/provider/models/batch-status   — enable/disable in bulk
# POST /api/provider/models/batch-delete   — remove in bulk

# Channel debugging
curl -s --max-redirs 0 "$BASE/api/provider/channel/test" -H "$AUTH"
curl -s --max-redirs 0 "$BASE/api/provider/channel/fetch_models" -H "$AUTH"

# Earnings (read-only)
curl -s --max-redirs 0 "$BASE/api/provider/earnings/summary" -H "$AUTH"
curl -s --max-redirs 0 "$BASE/api/provider/earnings" -H "$AUTH"
curl -s --max-redirs 0 "$BASE/api/provider/payouts" -H "$AUTH"   # payout history
```

Withdrawing money (`POST /api/provider/payout`) is a funds operation: do NOT call it yourself — send the user to the provider console in their browser.

Before submitting a provider application or changing anything that affects listings, prices, or money, state exactly what you are about to send and get the user's explicit confirmation. For bulk operations, show the full change list (model names, prices, statuses) first.

## More capabilities (not detailed here)

The platform also offers shared subscriptions, an official-key market, invoices, 2FA/passkey management, and payment initiation. These involve credentials, purchases, or account security — do not drive them via API. When the user asks, point them to the web console (`$BASE/console`).

## Troubleshooting

- **401 Unauthorized** — key missing/wrong, or the `sk-` prefix was dropped. Re-check the config you wrote.
- **402 / quota messages** — account balance is empty; the user needs to top up (Step 4).
- **429** — rate limited; back off and retry with exponential delay.
- **Model not found** — the model name must match `GET /api/pricing` exactly; do not invent names.

## Destructive actions

Deleting tokens, changing account settings, or anything under `/console` management APIs: always confirm with the user first.
