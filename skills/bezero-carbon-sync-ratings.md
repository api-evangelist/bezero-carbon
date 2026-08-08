---
name: Sync BeZero carbon credit ratings
description: Keep a local copy of BeZero Carbon ratings and rated projects in sync using incremental changedSince polling, without re-walking the whole catalog.
api: openapi/bezero-carbon-ratings-openapi.yml
operations:
  - listProjects
  - listRatings
generated: '2026-08-07'
method: generated
source: openapi/bezero-carbon-ratings-openapi.yml
---

# Sync BeZero carbon credit ratings

Use this when you need a current local view of BeZero's ratings — for a marketplace listing,
a portfolio risk screen, or a reporting pipeline.

## Before you start

- Base URL: `https://api.bezerocarbonmarkets.com/v3`
- Auth: OAuth 2.0 **client credentials**. POST to `https://login.bezerocarbonmarkets.com/oauth2/token`
  with the Client ID and Client Secret BeZero issued, requesting scopes
  `bcm/v3.projects:list` and `bcm/v3.ratings:list`. Send the result as `Authorization: Bearer <token>`.
  There is no self-service signup and no sandbox — credentials come from a commercial agreement.
- Budget: **1000 requests per minute, API-wide**. There are no remaining-quota headers on success,
  so pace yourself rather than discovering the ceiling with a 429.

## Steps

1. **Pick a compatibility version.** Send `Accept-API-Version: 3.1` if you want every published
   rating for a project. Omit the header, or send `3.0`, and you get only the first published
   rating per project. Any other value is rejected with `400`. The resolved value comes back on
   the response — check it rather than assuming.

2. **First run — walk the catalog.** Call `listProjects` (`GET /projects`) and then `listRatings`
   (`GET /ratings`). Both page at a fixed 100 records. Follow `links.nextPage` until it is null.
   Do not construct page URLs yourself.

3. **Store the watermark.** Both list responses return `links.queryLatestChanges` — a relative URL
   already carrying the `changedSince` timestamp for your next poll. Persist it. Persist
   `dataLastUpdatedAt` per record too; it is the field that moves whenever anything about a rating
   or its details changes.

4. **Subsequent runs — poll incrementally.** Call both list endpoints with the saved
   `changedSince` value (ISO 8601 datetime). Only changed records come back. Page through them
   the same way and store a fresh `queryLatestChanges`.

5. **Poll projects and ratings at the same cadence.** BeZero adds new ratings at any time, and
   warns that ratings, risk factors and summary analysis are always updated together. Fetching
   them on different schedules leaves the three out of sync.

6. **Join.** `ratings[].projectID` joins to `projects[].id`. `registryID` on both sides is the
   project's identifier in its own registry and is *not* interchangeable with the BeZero id —
   use it to reconcile against a registry export, not as a primary key.

## Rules you must honour

- **Watch status is not optional.** When `onRatingsWatch` is `true`, the rating is under review
  and may be upgraded, downgraded or reaffirmed. BeZero requires the watch status to be displayed
  next to the rating for as long as it is set.
- **Withdrawn is a value, not a deletion.** A withdrawn rating comes back with
  `rating: "Withdrawn"`. Do not treat its presence as a live rating, and do not treat its absence
  from your store as a reason to fall back to a stale one.
- **Respect the vintage range.** The rating applies only to credits inside
  `vintages[].startDate`–`endDate`. A rating applied to a credit outside that window is wrong.
- **Treat `id` as opaque.** Published examples are `ABC123`, `DEF123`, `GH1000000100` — no fixed
  prefix or length. Never construct one.

## Errors

| Status | Meaning | What to do |
|---|---|---|
| 400 | Unsupported `Accept-API-Version` | Send `3.0` or `3.1`, or omit the header |
| 403 | Unauthorised, or token lacks the operation's scope | Re-acquire a token with the right scope |
| 429 | Over 1000 req/min | Honour `Retry-After` (60s). Switch to `changedSince` polling instead of full walks |

Error bodies have no documented schema — branch on the status code, not on the body.
