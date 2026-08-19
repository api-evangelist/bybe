# BYBE

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

BYBE, Inc. is a promotion platform for the beer, wine and spirits industry. Brands fund cash-back
rebates in the BYBE dashboard; retailers embed those offers in their own apps, sites and loyalty
programs; BYBE handles US state-by-state alcohol promotion compliance, redemption validation,
clearing and consumer payout. Backed by Techstars and Rev1 Ventures, and acquired by
[Swiftly](https://www.swiftly.com/news/swiftly-acquires-bybe-powering-the-future-of-alcohol-promotions-in-retail)
in March 2024, BYBE continues to operate under its own brand and domains.

## The API

BYBE publishes a real OpenAPI 3.0.1 contract for its Retail API — **16 operations across 7 tags**
(Clips, Consumers, Manufacturers, Offers, Products, Redemptions, Stores).

| | |
|---|---|
| Specification | <https://api.bybe.io/v1/swagger.yaml> |
| Reference | <https://docs.bybe.io/> (Swagger UI at `api.bybe.io/docs/index.html`) |
| Base URL | `https://api.bybe.io` — paths carry the `/v1` prefix |
| Staging | `https://api.bybestaging.io` (docs at `docs.bybestaging.io`) |
| Auth | HTTP Basic — API key as username, API secret as password, issued at <https://developer.bybe.io/> |
| Overview | <https://bybe.com/developers> — also documents an SFTP CSV batch alternative |
| Pricing | <https://bybe.com/pricing> — 25% + $0.25 per successful transaction, capped at $1.75; free for retailers |
| Status | <https://status.bybe.com/> |

The contract expresses BYBE's compliance boundary directly: its only component schema is a US state
enum, used as the `state` filter on offers and stores.

### Notes for anyone integrating

- **No `Idempotency-Key` header.** Safe retry comes from the caller-supplied `retailer_identifier`
  natural key — `POST /v1/clips` returns `303` with a `location` pointing at the existing clip on a
  duplicate offer/consumer pair, and `409` when the key collides with a different clip.
- **Errors are not RFC 9457.** The envelope is `{"<resource>": {"errors": {"<field>": ["<message>"]}}}`
  with no machine-readable code.
- **No published rate limits** — no `X-RateLimit-*`, no `Retry-After`, no `429` in the contract.
- The specification declares **no `servers[]` block and no `operationId`s**, so every artifact in
  this repository cites operations by method + path.

### What BYBE does not publish

No SDK in any package registry, no CLI, no MCP server, no A2A agent card, no webhooks or AsyncAPI,
no `/.well-known/` documents, no security.txt, no trust center or named certifications, no dated
changelog, and no deprecation policy. Each of those absences is recorded with its probe evidence in
the relevant artifact rather than left blank.

Backed by: techstars — https://bybe.com/
