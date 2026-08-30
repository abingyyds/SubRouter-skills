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
- **Never let a credential reach stdout.** The authorization responses carry `device_code`, `key` and `access_token` in full; a bare `curl` prints them into the terminal and into the agent transcript. Always write those responses to a `0600` file and read individual fields out of it (Step 1 shows how). Report only status, `user_code`, the verification URL, and a `sk-` prefix.
- Payment is ALWAYS done by the user in their browser. Never handle card numbers, crypto seeds, or payment credentials.

## Step 0 — Agree on what you are about to change (do this first)

Getting a key and rewiring someone's coding agent are different jobs. Never infer the second from a request for the first. Before Step 1, state what you understood and get a yes:

1. **Goal** — which of these does the user want?
   - *Account only*: authorize and show the key, change nothing on disk.
   - *One-off call*: use the key in this session (env vars only, nothing persisted).
   - *Configure an SDK/app*: write into a project the user names.
   - *Switch a coding agent's model provider*: repoint Claude Code / Codex / Cursor at SubRouter.
2. **Target** — exactly which file(s) you would touch, by absolute path, and whether the change is session-only or persistent.
3. **Existing setup** — read the target first and say what is already there. If it already holds credentials or a provider, say so and ask before touching it.

Default to the least invasive option that satisfies the request. When unsure, stop at *account only* and ask.

**Codex specifically:** most Codex users are signed in with a ChatGPT subscription, not an API key. Do **not** write `~/.codex/config.toml`, do not overwrite `~/.codex/auth.json`, and do not replace an existing `model_provider` unless the user explicitly asked to move Codex onto SubRouter. Show them the config you would add and let them approve it.

Never overwrite an existing config file. Append, or write a new profile, and tell the user how to revert.

## Step 1 — Device authorization (get an API key)

Start a device authorization. `channel` identifies your agent type (`claude`, `cursor`, `codex`, …):

**Every endpoint under `/api` wraps its result in an envelope.** The payload is always under `data`, alongside `success` — `{"success":true,"message":"","data":{…}}`. Some endpoints add siblings (`/api/pricing` also returns `vendors`, `usable_group`, `group_ratio`; `/api/marketplace/models` returns `total`), so read `.data` explicitly rather than assuming the whole body is the payload. On failure `success` is `false` and `message` carries the reason. Endpoints under `/v1` are the raw OpenAI/Anthropic shapes and are **not** enveloped.

Write the response to a private file instead of letting it print:

```bash
umask 077
AUTH_TMP="$(mktemp -t subrouter-auth)"        # 0600 via umask
curl -s --max-redirs 0 -X POST "$BASE/api/device_auth" \
  -H 'Content-Type: application/json' \
  -d '{"channel":"claude"}' -o "$AUTH_TMP"
```

`.data` holds:

```json
{
  "user_code": "ABCD-2345",
  "device_code": "<64-char secret — never print this>",
  "verification_url": "https://subrouter.ai/device?code=ABCD-2345",
  "expires_in": 600,
  "interval": 3
}
```

Read `.data.user_code` and `.data.verification_url` out of the file and show only those. `device_code` is the machine-held secret: anyone who sees it can claim the key before you do, so it must never be echoed.

Tell the user to open the URL in a browser, log in (or register), check that the page shows the same `user_code`, and confirm. The code expires in 10 minutes.

Then poll every `.data.interval` seconds (never faster), again writing to the private file:

```bash
curl -s --max-redirs 0 -X POST "$BASE/api/device_auth/poll" \
  -H 'Content-Type: application/json' \
  -d "{\"device_code\":\"$DEVICE_CODE\"}" -o "$AUTH_TMP"
```

Branch on `.data.status`:

- `authorization_pending` — keep waiting.
- `slow_down` — you polled too fast; wait longer.
- `approved` — `.data.key` (`sk-…`), `.data.access_token` and `.data.base_url` are now filled. **Delivered exactly once**: if you lose them you must redo the whole flow. Persist them from the file, never via an echo.
- `success:false` with "invalid or expired" — the flow expired or was already consumed; start over from Step 1.

Shred the file when you are done: `rm -f "$AUTH_TMP"`.

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

### Billing rules (read before generating anything)

**Every successful generation is billed, including retries.** These rules are not optional:

- **One submission per user request.** Do not generate several variants, sizes or "improved" versions on your own initiative. If the user asked for one image, submit once.
- **A retry is a new charge.** Before resubmitting after a failure, a timeout, or output the user disliked, say so explicitly — "this will bill another generation" — and get a yes. The only exception is a submission that failed before billing (a 4xx rejection with no task id).
- **Never discard a billed artifact.** Every completed task's output must be written to disk before you move on, including intermediate attempts the user did not end up liking. Losing a paid image because a later attempt overwrote it is a real loss of the user's money. Use distinct filenames per task id; never reuse one path across attempts.
- **Download completed results promptly.** Upstream result URLs expire. Fetch and save as soon as `status` is `completed`; do not leave a finished task unclaimed while you do something else.
- **Validate before submitting**, not after: check `size`/aspect ratio, duration and model capability against `/api/pricing` first. A rejected parameter after billing is money gone.
- **Report at the end**: how many generations were submitted, the model, the size/duration, the total cost, and the absolute path of every saved file.

If the connection drops mid-flow, do not resubmit. The task already exists server-side — recover it via `GET /api/task/self` (Listing past tasks, below) and download the result. This is the main reason to prefer async: a dropped sync call loses an image you already paid for.

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

**Check the balance before assuming anything.** A freshly registered account has none, but a returning user re-authorizing an existing account may already be funded. Only point the user at `$BASE/console/topup` once you have seen a zero/insufficient balance or received a 402. Top-up always happens in the user's browser.

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
