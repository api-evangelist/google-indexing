---
name: Submit a URL to Google for crawling
description: >-
  Tell Google that a job posting or livestream page has been added or updated so it can be recrawled,
  then confirm the notification landed. Covers the ownership precondition, the request shape, the daily
  quota, and the two failures that account for almost all rejections.
api: openapi/google-indexing-urlnotifications-publish-api-openapi.yml
operations:
  - publishUrlNotification
  - getUrlNotificationMetadata
generated: '2026-08-13'
method: generated
source: >-
  https://developers.google.com/search/apis/indexing-api/v3/using-api,
  https://developers.google.com/search/apis/indexing-api/v3/prereqs,
  https://developers.google.com/search/apis/indexing-api/v3/quota-pricing
---

# Submit a URL to Google for crawling

## Before you start — check eligibility, not just credentials

Google restricts this API. It may only be used for pages that carry **JobPosting** structured data, or
**BroadcastEvent** embedded in a **VideoObject**. Submitting anything else is a policy violation, and
submissions are subject to spam detection — repeated abuse can get project access revoked. If the page
is not one of those two types, stop here and use a sitemap instead.

## Preconditions

All four must be true or the call will fail:

1. The Indexing API is enabled on a Google Cloud project.
2. A service account exists in that project with a downloaded JSON private key.
3. The site is verified in Google Search Console.
4. The service account email (`…@….iam.gserviceaccount.com`) is added as a **delegated site owner** on
   that Search Console property.

Steps 3 and 4 are the ones people skip, and they produce a 403 that reads like an auth problem but is
not.

## Step 1 — Get an access token

Exchange the service account's signed JWT assertion at `https://oauth2.googleapis.com/token` for a
bearer token scoped to:

```
https://www.googleapis.com/auth/indexing
```

Every request then carries `Authorization: Bearer <access_token>`. Any first-party client library
(`packages/google-indexing-packages.yml`) does this for you from the JSON key.

## Step 2 — Publish the notification

Call `publishUrlNotification`:

```
POST https://indexing.googleapis.com/v3/urlNotifications:publish
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "url": "https://example.com/jobs/technical-writer",
  "type": "URL_UPDATED"
}
```

Rules that matter:

- `Content-Type: application/json` is **mandatory**, not conventional.
- `url` must be a fully-qualified absolute URL, and it must fall under the verified property.
- `type` is `URL_UPDATED` (added or changed) or `URL_DELETED` (gone — see the removal skill).
- One URL per request body. To do many, use a batch of up to **100** requests.

A `200` means Google *may* recrawl the URL soon. It is an acknowledgement, not a crawl. The response
body is a `PublishUrlNotificationResponse` carrying `urlNotificationMetadata`.

## Step 3 — Confirm what Google recorded

Call `getUrlNotificationMetadata`:

```
GET https://indexing.googleapis.com/v3/urlNotifications/metadata?url=https%3A%2F%2Fexample.com%2Fjobs%2Ftechnical-writer
Authorization: Bearer <access_token>
```

It returns `latestUpdate` and `latestRemove`, each a `UrlNotification` with `url`, `type` and
`notifyTime`. This endpoint only answers for URLs your project has already successfully notified about —
it is a receipt lookup, not a crawl-status API. There is no operation anywhere in this API that tells you
whether Google actually crawled or indexed the page.

## Budget the quota yourself

| Quota | Limit | Window |
|---|---|---|
| `DefaultPublishRequestsPerDayPerProject` | 200 | day, resets midnight Pacific |
| `DefaultMetadataRequestsPerMinutePerProject` | 180 | minute |
| `DefaultRequestsPerMinutePerProject` | 380 | minute |

**The API returns no rate-limit headers.** No `RateLimit-*`, no `X-RateLimit-*`, no `Retry-After`. You
cannot read remaining quota from a response; count locally or watch the Google API Console. Exhaustion
appears as `403 dailyLimitExceeded` / `403 quotaExceeded`, or `429 rateLimitExceeded`.

More quota is free but gated: fill in Google's approval form, linked from the quota and pricing page.

## Handle the two errors you will actually see

Errors use the Google `google.rpc.Status` envelope (`{"error":{"code","message","status","details"}}`),
not RFC 9457 problem+json. Full registry: `errors/google-indexing-error-codes.yml`.

- **`403 Permission denied. Failed to verify the URL ownership.`** — Preconditions 3 or 4 are
  incomplete, or the URL is outside the verified property. Do **not** retry; fix Search Console
  ownership first. Retrying burns quota and will keep failing.
- **`400 Missing attribute. 'url' attribute is required.`** and its siblings — malformed body. Fix the
  request; a retry of the same body fails identically.

## Retries

There is no idempotency key, and none is needed for correctness: re-submitting the same `{url, type}` is
a supported flow, and Google's own guidance is to submit another update notification whenever the page
changes. But every attempt costs one of 200 daily publish requests, so retry only on `429`, `500` and
`503`, with backoff, and never on `400` or `403`.

## Test carefully — there is no sandbox

There is no test mode, no test keys, and no reserved test URLs. Every publish call is a production
request against Google Search. The only side-effect-free operation is `getUrlNotificationMetadata`.
