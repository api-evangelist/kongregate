# Kongregate

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
