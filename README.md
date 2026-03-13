# Google Indexing API

The Google Indexing API allows site owners to directly notify Google when pages are added or removed. It enables requesting crawling for updated content and notifying of page removals, leading to fresher content in search results.

## APIs

- **Google Indexing API** - Notify Google about URL updates and removals for faster indexing.

## Base URL

```
https://indexing.googleapis.com/v3
```

## Key Endpoints

- **Publish URL Notification** - `POST /urlNotifications:publish` - Notify Google a URL was updated or removed
- **Get URL Notification Metadata** - `GET /urlNotifications/metadata` - Get metadata about a URL's notification status

## Artifacts

- [APIs.yml](apis.yml) - APIs.json index
- [OpenAPI](openapi/openapi.yml) - OpenAPI 3.1.0 specification
- [JSON Schema](json-schema/UrlNotification.json) - JSON Schema (draft 2020-12) for UrlNotification
- [JSON-LD Context](json-ld/context.jsonld) - JSON-LD context mapping

## Resources

- [Getting Started](https://developers.google.com/search/apis/indexing-api/v3/quickstart)
- [Prerequisites](https://developers.google.com/search/apis/indexing-api/v3/prereqs)
- [Using the API](https://developers.google.com/search/apis/indexing-api/v3/using-api)

## Maintainers

- **Kin Lane** - kin@apievangelist.com
