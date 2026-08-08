---
name: Screen a carbon project on BeZero risk factors
description: Take a project or rating, pull its BeZero rating plus premium additionality, carbon accounting and permanence risk factor scores, and read them correctly.
api: openapi/bezero-carbon-ratings-openapi.yml
operations:
  - listRatings
  - getRiskFactors
  - listProjects
generated: '2026-08-07'
method: generated
source: openapi/bezero-carbon-ratings-openapi.yml
---

# Screen a carbon project on BeZero risk factors

Use this when a buyer, broker or diligence workflow needs the risk decomposition behind a
headline rating — not just the letter.

## Before you start

- Base URL: `https://api.bezerocarbonmarkets.com/v3`
- Auth: OAuth 2.0 client credentials against `https://login.bezerocarbonmarkets.com/oauth2/token`.
  You need `bcm/v3.ratings:list` and — for step 3 — `bcm/v3.ratings:riskFactors`.
- **Risk factors are a Premium endpoint.** If the contract does not include the Premium tier, the
  request returns `403` even though the token is valid. A 403 here usually means "not entitled",
  not "bad token".

## Steps

1. **Find the project.** Call `listProjects` (`GET /projects`) and match on `registryID` and
   `accreditor` if you are starting from a registry identifier, or on `name`/`location`
   (ISO 3166-1 alpha-3) if you are starting from a project description. Keep `id`.

2. **Find the rating.** Call `listRatings` (`GET /ratings`) and select the record whose
   `projectID` equals the project `id`. Check `vintages[]` covers the vintage you care about —
   a project can carry more than one rating across vintage ranges, and under
   `Accept-API-Version: 3.1` more than one rating is returned per project. Under the default
   `3.0` you only see the first published rating, which may not be the one that applies.
   Keep the rating's `id` and its `summaryAnalysis` — the summary now ships inline on the list
   response, so there is no need to call `getRatingDetails` (BeZero has deprecated it).

3. **Pull the risk factors.** Call `getRiskFactors`
   (`GET /ratings/{ratingID}/risk-factors`) with that rating `id`. You can also follow
   `links.riskFactors` from the rating record instead of building the path.

4. **Read the scales correctly — they are two different scales.**
   - The **headline rating** is uppercase: `AAA AA A BBB BB B C D`, plus `Withdrawn`.
     It expresses the likelihood the credit achieves a tonne of CO2e avoided or removed.
   - The **risk factor scores** are lowercase: `aaa aa a bbb bb b c d`. A score may be an
     **empty string** when that factor is not scored — treat empty as "no assessment", never as
     the bottom of the scale.
   - The three factors are `additionality`, `carbonAccounting` and `permanence`, each returned
     as an object with a `score` field.

5. **Present it honestly.** If `onRatingsWatch` is `true`, show the watch status alongside the
   rating — BeZero requires it until the watch is lifted. If `rating` is `Withdrawn`, say so
   rather than showing the last known letter. Link back to `platformURL` for the full analysis.

## Rules you must honour

- Ratings, risk factors and summary analysis are always updated together at BeZero. Fetch them in
  the same pass; a cached risk factor next to a fresh rating is a misrepresentation.
- The rating is BeZero's **opinion**, scoped to the vintage range returned. Do not extend it to
  other vintages of the same project.
- Do not cache past your agreement's terms — the API terms govern permitted display and refresh.

## Errors

| Status | Meaning | What to do |
|---|---|---|
| 403 on `/risk-factors` | Token valid but contract lacks the Premium tier | Fall back to the headline rating and `summaryAnalysis`; do not retry |
| 404 | No rating for that `ratingID` | Re-read `id` from `listRatings`; withdrawn ratings are still returned, so 404 means the id is wrong |
| 429 | Over 1000 req/min | Honour `Retry-After` (60s) |
