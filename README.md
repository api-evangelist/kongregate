# Kongregate

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

Kongregate is a browser-game platform and publisher hosting thousands of free online games. For
game developers it operates a two-part developer API: a CDN-delivered client JavaScript API
(with Unity WebGL bindings) for player identity, statistics and the Kreds purchase flow, and a
server-side REST API at `api.kongregate.com` for authentication, virtual-goods inventory,
leaderboards, guilds and characters.

Backed by: lightspeed-venture-partners, uncork-capital — https://www.kongregate.com

## APIs

| API | Base URL | Docs |
|---|---|---|
| Kongregate Server-Side API | `https://api.kongregate.com/api` | [docs](https://docs.kongregate.com/docs/server-side-http) |
| Kongregate Client JavaScript API | `https://cdn1.kongregate.com/javascripts/kongregate_api.js` | [docs](https://docs.kongregate.com/docs/javascript-api) |

## Artifacts

- `openapi/` — first-party **OpenAPI 3.1.0** (15 operations), harvested verbatim from the
  provider's ReadMe reference pages
- `overlays/` — API Evangelist enhancements (tags, missing securitySchemes, spec-defect notes)
- `llms/` — provider-published `llms.txt`, saved verbatim
- `mcp/` — candidate MCP tool surface derived from the spec (no official server exists)
- `skills/` — four packaged Agent Skills grounded in real operationIds
- `agentic-access/` — recommended `x-agentic-access` contracts for all 15 operations
- `authentication/`, `conventions/`, `errors/`, `data-model/`, `conformance/`, `lifecycle/`
- `asyncapi/` — the API Callback (webhook) catalogue and its HMAC-SHA256 signed-request scheme
- `packages/`, `sandbox/`, `well-known/`, `security/`

## Notable findings

- **Errors are returned as HTTP 200** with `success: false` in the body. Clients that branch on
  HTTP status will read failed authentication as success.
- **`components.securitySchemes` is empty** in the published spec even though every operation
  requires an `api_key`, so generated clients cannot authenticate.
- **No idempotency contract**, while `use_item` decrements inventory and `add`-type statistics
  accumulate — both unsafe to retry blindly.
- **No status page, SLA, changelog, deprecation policy, or `/.well-known/` surface.**
- Two path templates mix OpenAPI `{scope}` and Rails `:statistic_id` styles and will not
  substitute correctly in generated clients.
