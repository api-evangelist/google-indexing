---
name: Tell Google a page has been removed
description: >-
  Signal that a job posting or livestream page is gone so Google can drop it from the index, and verify
  the removal notification was recorded. Covers what URL_DELETED does and does not do, and the site-side
  work that has to happen first.
api: openapi/google-indexing-urlnotifications-publish-api-openapi.yml
operations:
  - publishUrlNotification
  - getUrlNotificationMetadata
generated: '2026-08-13'
method: generated
source: >-
  https://developers.google.com/search/apis/indexing-api/v3/using-api,
  https://developers.google.com/search/apis/indexing-api/v3/core-errors
---

# Tell Google a page has been removed

Use this when a job posting closes or a livestream event page is taken down. It is the same operation as
a crawl request, with `type: URL_DELETED`.

## Do the site-side work first

`URL_DELETED` is a notification, not a delete. Google will recrawl the URL to confirm the removal, so the
page itself must already be gone or excluded. Before calling, one of these must be true:

- the URL returns `404` or `410`, **or**
- the page carries `<meta name="robots" content="noindex" />`.

If the page still returns a normal `200` with content, the notification will not remove it.

## Preconditions

Identical to the submit flow, and all four are required:

1. Indexing API enabled on a Google Cloud project.
2. Service account with a JSON private key.
3. Site verified in Search Console.
4. Service account added as a delegated site owner of that property.

## Step 1 — Get a token

Exchange the service account JWT assertion at `https://oauth2.googleapis.com/token` for a bearer token
with scope `https://www.googleapis.com/auth/indexing`.

## Step 2 — Publish the removal

Call `publishUrlNotification`:

```
POST https://indexing.googleapis.com/v3/urlNotifications:publish
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "url": "https://example.com/jobs/closed-role",
  "type": "URL_DELETED"
}
```

`URL_DELETED` requests draw on the **same** 200-per-day publish quota as `URL_UPDATED` — there is no
separate allowance for removals. If you are expiring a large batch of postings, count them against the
same budget and use batching (up to 100 requests per batch).

## Step 3 — Confirm the notification was recorded

```
GET https://indexing.googleapis.com/v3/urlNotifications/metadata?url=<url-encoded-url>
```

Read `latestRemove`. A populated `latestRemove.notifyTime` proves Google received your removal
notification. It does **not** prove the page has been dropped from the index — this API exposes no
indexing status at all. Use Search Console to check whether the URL actually left the index.

## Errors

Same `google.rpc.Status` envelope and the same two dominant failures as the submit flow
(`errors/google-indexing-error-codes.yml`):

- `400 Unknown type. 'type' attribute is required and must be 'URL_REMOVED' or 'URL_UPDATED'.` — note
  that Google's error text says `URL_REMOVED` while the accepted enum value in the contract is
  `URL_DELETED`. Send `URL_DELETED`.
- `403 Permission denied. Failed to verify the URL ownership.` — Search Console ownership, not the token.
  Fix it; do not retry.

Retry only on `429`, `500`, `503`, with backoff. Every retry costs a daily publish request.
