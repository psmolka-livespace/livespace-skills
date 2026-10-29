# Calling the Livespace API from the plugin

Load this file when the plugin needs data the `context` message does not carry — other records, lists, search, the current user's profile — or needs to write. Check first: in protocol v2 the `context` already carries most of the record being viewed (owners, stage, value, custom fields, …), and it is re-sent when the record changes. A plugin that only shows facts about that record usually needs no API call at all.

> **Authoritative source:** <https://plugin-sdk.feature.livespace.dev/v1/docs/api-reference.md>. Endpoints, parameters and response shapes: <https://api-docs.livespace.io/> (a Postman collection).

## With the SDK (default)

```js
const response = await LsPlugin.api.call("Deal/search", { limit: 10 })

const { deal } = await LsPlugin.api.deal.get(LsPlugin.context.deal.id)
```

- **`endpoint`** is `Module/action` as the API reference names it (`Deal/get`, `Contact/get`, `Default/User_getInfo`), without the `/api/public/json/` prefix. The host is whichever Livespace the plugin runs in — never hardcode one.
- **`params`** go in the query string; `undefined` and `null` are left out.
- **Every `call()` gets its own credentials**, so `Promise.all([...])` needs nothing special.
- **Credentials are issued in `view` mode only.** In `preview` there is no record — render sample data instead of calling the API.

### Errors

Every rejection is an `LsPluginApiError`; branch on `error.code`.

| `code` | Meaning | Do |
|---|---|---|
| `http_error` | The API answered with a failure; `error.status` has the status | 4xx: fix the call. 5xx: offer a retry |
| `invalid_response` | Success status but the body is not JSON (a proxy page) | Error + retry |
| `network_error` | Offline, DNS, CORS, dropped connection | "Couldn't reach Livespace" + retry |
| `timeout` | No credentials within 15 s | Retry |
| `unavailable_in_mode` | Called in `preview` or `config` | Render sample data instead |
| `unauthenticated` | No valid Livespace session | Ask the user to reload Livespace |
| `forbidden` | The user may not use the API from this plugin | Permission message |
| `not_found` | Livespace doesn't know this plugin | Error; retrying won't help |
| `conflict` | Credential issuing was busy (already retried once) | Offer a retry |
| `bad_request` | Malformed credential request — a bug | Error; retrying won't help |
| `server_error` | Livespace failed to issue credentials | Error + retry |

Map these onto the plugin's `error` state. Nothing is retried automatically except one `conflict` retry, because every attempt spends a set of credentials.

## Driving `fetch` yourself

For an action that reads only a JSON body, or a backend that calls Livespace on the plugin's behalf:

```js
const c = await LsPlugin.api.credentials()   // { apiKey, apiSha, apiSession, apiBaseUrl }
const qs = new URLSearchParams({
  _api_auth: "key", _api_key: c.apiKey, _api_sha: c.apiSha, _api_session: c.apiSession,
  id: dealId,
})
const res = await fetch(`${c.apiBaseUrl}/api/public/json/Deal/get?${qs}`)
```

Without the SDK, get the same four values with the `request-api-credentials` → `api-credentials` exchange in [postmessage-contract.md](postmessage-contract.md).

- **One set, one call.** `apiSha` is single-use: request a fresh set per request and never cache them.
- **An unused set expires 15 minutes after issue** — request it at the moment of the call, not at startup.
- **`apiBaseUrl` is the API host.** Build every URL from it; don't assume `location.origin` and don't hardcode a host.

## Endpoints worth knowing

- `Deal/get?id=<uuid>` — one deal: `{ data: { deal: {...} }, status: true }`. `api.deal.get` strips the `data` envelope for you.
- `Contact/get?type=<company|person>&id=<uuid>` — one CRM contact: `{ data: { company | person: {...} } }`.
- `Default/User_getInfo` — the current user's profile (`login`, `email`, `firstname`, `lastname`, `locale`, …). It has **no `id`**; the viewer's id is `context.user.id`.

Read fields defensively — candidate lists rather than one hardcoded property — and log the raw response on a parse failure, so a shape change is diagnosed from the console rather than a redeploy.
