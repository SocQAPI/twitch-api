<!-- Generated from the SocQ API catalog and docs. Do not edit by hand. -->
# Twitch API

[![SocQ Twitch API](https://img.shields.io/badge/API-SocQ%20Twitch%20API-F64C31)](https://socq.ai/apis/twitch?utm_source=github&utm_medium=readme&utm_campaign=socq-twitch-api)
[![API documentation](https://img.shields.io/badge/Docs-docs.socq.ai-111827)](https://docs.socq.ai/api-manual/twitch/overview)
[![License: MIT](https://img.shields.io/badge/License-MIT-2563EB)](LICENSE)
[![Check examples](https://github.com/SocQAPI/twitch-api/actions/workflows/check.yml/badge.svg)](https://github.com/SocQAPI/twitch-api/actions/workflows/check.yml)

3 endpoints for public Twitch data through one SocQ API key, with runnable cURL, Node.js and Python examples and a valid request body for every endpoint.

[Browse Twitch APIs](https://socq.ai/apis/twitch?utm_source=github&utm_medium=readme&utm_campaign=socq-twitch-api) · [Get an API key](https://socq.ai/dashboard/api-key?utm_source=github&utm_medium=readme&utm_campaign=socq-twitch-api) · [Documentation](https://docs.socq.ai/api-manual/twitch/overview) · [All SocQ examples](https://github.com/SocQAPI/socq-examples)

## Endpoints

Prices are in SocQ credits; 1 credit = $0.005.

| Endpoint | publicId | What it returns | Price | Launched |
| --- | --- | --- | --- | --- |
| [Twitch Profile API](#twitch-profile-api) | `twitch/profile` | Resolve one public Twitch username into a normalized profile record with identity, biography, imagery, account flags, and visible follower, following, and post counts when available. | 0.19 credits ($0.00095) per call | 2026-08-15 |
| [Twitch User Videos API](#twitch-user-videos-api) | `twitch/user-videos` | Collect one page of public videos for a Twitch username with optional type and order filters, returning normalized video identity, creator context, media, duration, and visible engagement. | 0.19 credits ($0.00095) per call | 2026-08-14 |
| [Twitch Clip Downloader API](#twitch-clip-downloader-api) | `twitch/clip` | Get a Twitch clip's title, views, length, creation time, broadcaster and downloadable video file from its clip URL. | 0.5 credits ($0.0025) per call | 2026-11-17 |

## Quick start

1. Create an API key at [socq.ai/dashboard/api-key](https://socq.ai/dashboard/api-key?utm_source=github&utm_medium=readme&utm_campaign=socq-twitch-api).
2. Export it. The examples read only `SOCQ_API_KEY`.

```bash
git clone https://github.com/SocQAPI/twitch-api.git
cd twitch-api
export SOCQ_API_KEY="your-api-key"
```

3. Run any endpoint by its publicId. Without a second argument the script sends `payloads/<slug>.json`; pass a JSON file or an inline JSON object to change the input.

```bash
# Node.js 20+
node examples/node/socq.mjs twitch/profile

# Python 3.8+ (standard library only)
python3 examples/python/socq.py twitch/profile payloads/profile.json

# cURL + jq
bash examples/curl/socq.sh twitch/profile '{"username":"ninja"}'
```

Each script submits the task, polls it until it succeeds or fails, follows every result page, and writes the records to `output/<slug>-<task_id>.json`. Run `npm run check` to validate the payloads and scripts offline.

Never expose `SOCQ_API_KEY` in browser code, mobile apps, public repositories, screenshots, fixtures or logs.

## How the API works

Base URL `https://api.socq.ai`. Every request sends the API key as a bearer token:

```http
Authorization: Bearer <SOCQ_API_KEY>
Content-Type: application/json
```

All endpoints are asynchronous:

```text
POST /v1/<publicId>  ->  data.task_id  ->  GET /v1/tasks/{task_id}  ->  queued | running | succeeded | failed
```

1. **Submit** the JSON body to `POST https://api.socq.ai/v1/<publicId>`. The response returns `data.task_id` immediately; it means the task was created, not that data is ready.
2. **Poll** `GET https://api.socq.ai/v1/tasks/{task_id}` until `data.status` is `succeeded` or `failed` (a failed task carries `data.error_message`).
3. **Read results.** A succeeded task returns normalized records in `data.results.items`. Use `limit` (default 50, max 500) and pass `data.results.next_cursor` as `cursor` while `data.results.has_more` is true. `fields` selects up to 50 comma-separated fields or dot paths.
4. **Raw files (optional).** `GET https://api.socq.ai/v1/tasks/{task_id}/files` lists raw JSONL exports.

```bash
curl "https://api.socq.ai/v1/tasks/$TASK_ID?limit=100" \
  -H "Authorization: Bearer $SOCQ_API_KEY"

curl "https://api.socq.ai/v1/tasks/$TASK_ID?limit=100&cursor=$NEXT_CURSOR" \
  -H "Authorization: Bearer $SOCQ_API_KEY"
```

`callback_url` (where an endpoint accepts it) makes SocQ POST to your URL when the task reaches a terminal state; keep polling as a fallback.

**Billing.** A submit may reserve credits from `results_limit` or the number of submitted inputs; when the task finishes SocQ settles the charge from the actual result count and returns unused reserved credits. `402` means insufficient credits.

**Errors.** `400` invalid body or unsupported field, `401` missing or invalid key, `402` insufficient credits, `403` IP allowlist or task owned by another account, `404` endpoint or task not found, `422` input validation failed, `429` rate or credit-window limit (respect `Retry-After`), `500` submission failed. A client-side timeout does not cancel a task: resume polling the same `task_id` instead of submitting again.

Reference: [Authentication](https://docs.socq.ai/api-manual/agent-api/authentication) · [Task status and results](https://docs.socq.ai/api-manual/agent-api/status) · [Task files](https://docs.socq.ai/api-manual/agent-api/files) · [Callbacks](https://docs.socq.ai/api-manual/agent-api/callbacks) · [Errors](https://docs.socq.ai/api-manual/agent-api/errors)

## Endpoint reference

### Twitch Profile API

Resolve one public Twitch username into a normalized profile record with identity, biography, imagery, account flags, and visible follower, following, and post counts when available.

- **Request:** `POST https://api.socq.ai/v1/twitch/profile`
- **publicId:** `twitch/profile`
- **Price:** 0.19 credits ($0.00095) per call
- **Launched:** 2026-08-15
- **Example body:** [`payloads/profile.json`](payloads/profile.json)

Fetch normalized profiles data from Twitch.

Successful tasks are billed per task.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `username` | string | Yes | A public account username. |

```bash
curl -X POST "https://api.socq.ai/v1/twitch/profile" \
  -H "Authorization: Bearer $SOCQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "username": "ninja"
}'
```

Full docs: [https://docs.socq.ai/api-manual/twitch/profile](https://docs.socq.ai/api-manual/twitch/profile) · Landing page: [socq.ai/apis/twitch/profile](https://socq.ai/apis/twitch/profile?utm_source=github&utm_medium=readme&utm_campaign=socq-twitch-api)

### Twitch User Videos API

Collect one page of public videos for a Twitch username with optional type and order filters, returning normalized video identity, creator context, media, duration, and visible engagement.

- **Request:** `POST https://api.socq.ai/v1/twitch/user-videos`
- **publicId:** `twitch/user-videos`
- **Price:** 0.19 credits ($0.00095) per call
- **Launched:** 2026-08-14
- **Example body:** [`payloads/user-videos.json`](payloads/user-videos.json)

Fetch normalized user videos data from Twitch.

Successful tasks are billed per task.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `filter_by` | string | No | (one of `all`, `archive`, `archives`, `highlight`, `highlights`, `upload`, `uploads`) |
| `sort_by` | string | No | (one of `time`, `views`) |
| `username` | string | Yes | A public account username. |

```bash
curl -X POST "https://api.socq.ai/v1/twitch/user-videos" \
  -H "Authorization: Bearer $SOCQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "username": "ninja",
  "filter_by": "all",
  "sort_by": "time"
}'
```

Full docs: [https://docs.socq.ai/api-manual/twitch/user-videos](https://docs.socq.ai/api-manual/twitch/user-videos) · Landing page: [socq.ai/apis/twitch/user-videos](https://socq.ai/apis/twitch/user-videos?utm_source=github&utm_medium=readme&utm_campaign=socq-twitch-api)

### Twitch Clip Downloader API

Get a Twitch clip's title, views, length, creation time, broadcaster and downloadable video file from its clip URL.

- **Request:** `POST https://api.socq.ai/v1/twitch/clip`
- **publicId:** `twitch/clip`
- **Price:** 0.5 credits ($0.0025) per call
- **Launched:** 2026-11-17
- **Example body:** [`payloads/clip.json`](payloads/clip.json)

Clips are the short highlights viewers cut from a stream, and they are what gets shared on X, Reddit and Discord. Pass a clip link to get its title, view count, length, creation time, the broadcaster who streamed it with their avatar, and the clip's video file and thumbnail. Teams use it to archive standout moments, measure how far a clip traveled, or credit the right channel when reposting.

Provide one Twitch clip URL in url, either twitch.tv/channel/clip/slug or clips.twitch.tv/slug. Each request costs 0.5 credits and returns one row. The video file link is signed and expires after about a day, so download it promptly if you need a copy.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `url` | string | Yes | Twitch clip URL, such as https://www.twitch.tv/<channel>/clip/<slug> or https://clips.twitch.tv/<slug>. |

```bash
curl -X POST "https://api.socq.ai/v1/twitch/clip" \
  -H "Authorization: Bearer $SOCQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "url": "https://www.twitch.tv/staryuuki/clip/CloudySavageMarjoramRuleFive--ErzsYbE7UWvgCMQ"
}'
```

Full docs: [https://docs.socq.ai/api-manual/twitch/clip](https://docs.socq.ai/api-manual/twitch/clip) · Landing page: [socq.ai/apis/twitch/clip](https://socq.ai/apis/twitch/clip?utm_source=github&utm_medium=readme&utm_campaign=socq-twitch-api)

## Repository contents

| Path | Purpose |
| --- | --- |
| [`examples/curl/socq.sh`](examples/curl/socq.sh) | Submit, poll and paginate with cURL and jq |
| [`examples/node/socq.mjs`](examples/node/socq.mjs) | Same workflow in Node.js 20+ (no dependencies) |
| [`examples/python/socq.py`](examples/python/socq.py) | Same workflow in Python 3.8+ (standard library) |
| [`payloads/`](payloads) | One documented request body per endpoint |
| [`endpoints.json`](endpoints.json) | Endpoint list, prices and request schemas used by the scripts and checks |
| [`fixtures/`](fixtures) | Synthetic submit and task response shapes |
| [`docs/production-notes.md`](docs/production-notes.md) | Retries, timeouts, pagination and data handling |
| [`scripts/check.mjs`](scripts/check.mjs) | Offline check: payloads, syntax and secrets |

## Responsible use and platform scope

- Use only publicly accessible Twitch data and the fields supported by each endpoint.
- Do not use the examples to access restricted content, login-only surfaces, authentication controls or non-public personal information.
- SocQ is not an official API of the represented platform and is not affiliated with or endorsed by it. Platform names and trademarks belong to their respective owners.
- Before production use, assess the laws, platform terms, privacy obligations and retention requirements that apply to your organization and use case.
- Collect only the fields needed for a defined purpose, restrict access, set retention periods, and support correction or deletion workflows where required.

This section describes the public-data boundary; it is not legal advice.

## License

[MIT](LICENSE)
