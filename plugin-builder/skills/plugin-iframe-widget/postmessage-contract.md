# postMessage contract — protocol version 2

The raw host↔iframe contract. **Use the SDK instead whenever you can** — `LsPlugin` does the handshake, origin validation, request correlation and replay for you, and all four are easy to get subtly wrong. Load this file when the plugin cannot load the SDK, or when debugging what the SDK sends and receives.

> **Authoritative source:** <https://plugin-sdk.feature.livespace.dev/v1/docs/protocol.md>. This file is a summary; if the two diverge, the published SDK docs win.

Every message is `{ type, payload }`, and every `type` is prefixed `livespace-plugin:`. The plugin is the iframe; the host is `window.parent`.

## Origin — validate it, both ways

Livespace checks that messages come from your plugin's origin. **The plugin must do the same in reverse**: work out the host origin, drop every inbound message from anywhere else, and post only to that origin.

```js
const hostOrigin =
  location.ancestorOrigins?.[0] ??
  (document.referrer ? new URL(document.referrer).origin : null)

window.addEventListener("message", (e) => {
  if (e.origin !== hostOrigin || e.source !== window.parent) return
  const { type, payload } = e.data ?? {}
  // dispatch on type …
})

const post = (type, payload) => window.parent.postMessage({ type, payload }, hostOrigin)
```

A plugin that skips this accepts messages from any page that embeds it. Never post to `'*'`.

## The handshake

1. **Attach the `message` listener, then send `ready`** — as early as possible, and before the page finishes loading. Livespace sends nothing until `ready` arrives, and answers straight away; a message posted before the listener exists is dropped and never resent.
2. The host answers with **`mode`**, always first.
3. Then, depending on the mode:

| Mode | What it is | Host sends |
|---|---|---|
| `view` | Running on a real profile — the normal case | `context` and `config`, each re-sent when it changes |
| `preview` | An admin is configuring the plugin; there is no record | `config` only, re-sent on every keystroke |
| `config` | Livespace is asking which settings the plugin needs; nothing drawn is shown | `request-config-form` — the plugin must answer |

A page that loads without sending `ready` is given about one second, then treated as a **version 1** plugin: `context` is `{ <type>: { id, name }, user }`, sent once. Everything below is version 2.

## Plugin → host

| Type | Payload | Notes |
|---|---|---|
| `ready` | `{ protocolVersion: 2, sdkVersion? }` | Leave `protocolVersion` out and you get version 1 data. |
| `request-api-credentials` | `{ requestId }` | `requestId` is yours (`crypto.randomUUID()`); the reply echoes it. `view` mode only. |
| `config-form` | `{ fields: [...] }` | The answer to `request-config-form`. **Always send it**, `fields: []` if there are no settings — silence leaves the admin on an empty screen after a timeout. Field shapes: [config-forms.md](config-forms.md). |
| `metadata` | `{ supportedContextTypes?, minWidthPx?, minHeightPx? }` | What the plugin needs in order to be placed. Send it immediately before `config-form`. See [Placement metadata](#placement-metadata). |
| `notify` | `{ title, body?, variant? }` | A toast in the Livespace chrome. `variant`: `default` \| `success` \| `warning` \| `error`. Title clamped to 120 chars, body to 300, about one per second — faster ones are dropped. Extra fields are discarded. |
| `modal-size` | `{ minWidthPx?, maxWidthPx?, minHeightPx?, maxHeightPx? }` | Bounds for the dialog layout, if an admin enabled it. Each 1–10 000; a minimum above its maximum drops that axis; `{}` resets. |

## Host → plugin

| Type | Payload | Notes |
|---|---|---|
| `mode` | `{ mode: "view" \| "config" \| "preview" }` | Always first. |
| `context` | `{ deal? \| company? \| person? \| space?, user: { id } \| null }` | `view` mode only. Exactly one record key; **the key name is the discriminator**. **Re-sent whenever the record changes** — handle every one, not just the first. The record is described in [Objects](#objects). |
| `config` | `{ config: {...} \| null }` | Sent once, then on every change. `null` means nobody has ever configured the plugin — not the same as `{}`. |
| `api-credentials` | `{ requestId, apiKey, apiSha, apiSession, apiBaseUrl }` or `{ requestId, error }` | `error`: `bad_request` \| `unauthenticated` \| `forbidden` \| `not_found` \| `conflict` \| `server_error`. Drop a reply whose `requestId` you did not send. |
| `request-config-form` | — | `config` mode only. Answer with `metadata` (optional) then `config-form`. |
| `fullscreen-changed` | `{ active: boolean }` | The embed entered or left fullscreen. Livespace drives both — a plugin cannot request it — and only where an administrator has enabled fullscreen for the plugin. |
| `preferences` | `{ preferences: {...} \| null }` | The viewer's language, time zone and date/number/money formats, in every mode. **Not every Livespace sends it yet** — render with your own defaults and re-render if it arrives. |

`request-fullscreen` / `request-exit-fullscreen` (plugin → host) are **deprecated**: still answered so old plugins keep working, but never use them in new code.

## Objects

The record always has `id` (UUID) and `name`. Livespace normally adds its full description, marked by **`version`** — check it before reading anything else, because an older Livespace sends `id` and `name` alone.

- **Every type:** `version`, `type`, `createdAt`, `modifiedAt`, `lastActivityAt`, `customId`, `owners[{ type, id, name }]`, `groups[]`, `customFields{ <uuid>: { name, type, value } }`.
- **Deal:** `number`, `status`, `stage`, `value{ amount, currency }`, `probability`, `startDate`, `endDate`, `source`, `contact`, `company`.
- **Company:** `taxId`, `regon`, `website`, `clientStatus`, `source`.
- **Person:** `firstName`, `middleName`, `lastName`, `position`, `source`, `company`.
- **Space:** `number`, `description`.

Related records (`contact`, `company`) arrive as `{ name, restricted }` — when `restricted` is `true`, `name` is `null` because the viewer may not see it; say so rather than showing a blank. Most fields can be `null`. Look custom fields up by **UUID**, never by their (renamable) `name`, and treat an unknown `type` as opaque. Full field tables and real example payloads: <https://plugin-sdk.feature.livespace.dev/v1/docs/objects.md>.

## Placement metadata

`supportedContextTypes` is `null` (no requirements — anywhere, including the apps menu, where there is no record) or a non-empty list of `deal`, `company`, `person`, `space`. An empty list is refused, never read as "anywhere". The admin screen uses `minWidthPx` / `minHeightPx` to disable columns that are too narrow and heights that are too short. It is advisory, and `supportedContextTypes` applies only to a plugin added by URL — a Catalog plugin goes where its Catalog entry says. `allowedObjectTypes` is the deprecated old name.

## Rules

- Keep the `livespace-plugin:` prefix and dash-case. Never invent a message type — the host ignores anything it does not know.
- One credential set authorises exactly one API call (`apiSha` is single-use) and expires 15 minutes after issue. See [livespace-api.md](livespace-api.md).
