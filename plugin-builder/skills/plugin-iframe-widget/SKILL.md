---
name: plugin-iframe-widget
version: "2.0"
description: Use when building an iframe-widget Livespace plugin — after the `plugin-builder` router selected the iframe-widget type. A web page embedded as an `<iframe>` in a deal, company, person or space profile (or opened from the apps menu), built on the Livespace plugin SDK (`LsPlugin`, protocol v2) — rich record context, admin settings forms, host toasts, fullscreen and dialog layouts, design tokens. For plugins that must keep third-party secrets server-side, see the `plugin-iframe-widget-backend` sub-skill.
---

# Livespace iframe-widget plugin — builder recipe

A Livespace **iframe widget** is a web page loaded as an `<iframe>` inside a Livespace profile — **Deal, Company, Person or Space** — or opened from the **apps menu**, where there is no record. It is served over **HTTPS** from a domain on the account's plugin whitelist, never from the Livespace origin itself. It talks to the host only through `window.postMessage`, normally via the **Livespace plugin SDK** — one script tag, no dependencies, the `LsPlugin` global.

## Step 0 — read the published SDK docs

Livespace publishes the SDK documentation with every SDK release, and **it is authoritative** — the reference files in this skill are summaries that defer to it. Before designing, fetch the overview, then the page for each step as you reach it:

| Page | Covers |
|---|---|
| <https://plugin-sdk.feature.livespace.dev/v1/docs/index.md> | Modes, the rules that bite |
| <https://plugin-sdk.feature.livespace.dev/v1/docs/getting-started.md> | Loading the SDK, `ready`, reacting to changes |
| <https://plugin-sdk.feature.livespace.dev/v1/docs/api-reference.md> | Every `LsPlugin` method, event and error code |
| <https://plugin-sdk.feature.livespace.dev/v1/docs/objects.md> | What a deal / company / person / space carries |
| <https://plugin-sdk.feature.livespace.dev/v1/docs/config-forms.md> | Admin settings forms |
| <https://plugin-sdk.feature.livespace.dev/v1/docs/styling.md> | Design tokens |
| <https://plugin-sdk.feature.livespace.dev/v1/docs/protocol.md> | Raw messages, sandbox |

The same content is in one file at <https://plugin-sdk.feature.livespace.dev/v1/llms.txt> (~65 KB — too large to read reliably in one fetch; use the pages). If something below disagrees with them, follow the SDK docs and say so.

Anything `llms.txt` marks **Preview** (today: the backend session token and secret config fields) depends on Livespace services that are not live. Don't build on it; mention it as a future option where relevant.

## Prerequisites

Reach this skill **after the `plugin-builder` router captured the plugin's purpose and selected the iframe-widget type.** If you arrived directly, confirm the purpose is captured (`plugin-builder` Step 1) — it drives every decision below. Follow the **Interaction Rules** from the `plugin-builder` router: one question at a time, multiple-choice preferred, short headers.

## Build flow

### Question 1 — where does it live?

Header `Where`:

- **On a record** (deal / company / person / space) — the plugin is about the record being viewed. Ask which record types; they become `supportedContextTypes`.
- **Apps menu** — a standalone tool with no record (a dashboard, a calculator, a search). It must declare **no** context requirements (`supportedContextTypes: null`), and its `context` carries only the viewer.

### Question 2 — what data does it need?

Header `Data`:

- **The record only** — in protocol v2 the `context` already carries the record's full description (owners, groups, custom fields, and per type e.g. a deal's stage, status, value, probability, dates, contact and company) and is re-sent whenever the record changes. **No API call, no credentials.** Prefer this whenever it is enough.
- **More Livespace data** — other records, lists, search, writes → `LsPlugin.api.call(...)`. See [livespace-api.md](livespace-api.md).
- **Nothing from Livespace** — a link out, a form posting elsewhere, static content.

### Question 3 — admin settings?

Header `Settings`: does an administrator need to set anything per account (an account id, a region, a default view, a toggle)?

- **Yes** → declare a **configuration form** with `LsPlugin.createConfigForm()`; Livespace renders, validates and stores it, and the plugin receives the values as `config`. No settings UI to build. See [config-forms.md](config-forms.md).
- **No** → nothing to declare; the SDK still answers Livespace's settings request with an empty form.

Config values are **not secret** — anyone who can open the plugin's settings can read them. A token or API key is never a config field; it means Question 4 is "backend".

### Question 4 — secrets server-side?

Header `Backend`: must the plugin call a **third-party service** (JIRA, Linear, an internal HR system, …) whose credentials must never reach the browser?

- **Client-only** — everything runs in the iframe; any static HTTPS host on the whitelist. Continue here.
- **Backend-enabled** → **invoke `plugin-iframe-widget-backend`** before scaffolding. Everything in this skill still applies to the frontend half.

A backend is a real operational cost (deploy target, env vars, secret rotation, monitoring). Push for client-only first.

## Project layout

Kebab-case, English file names.

```
<project>/
├── index.html        # entry point: SDK <script> + tokens <link> in <head>
├── assets/           # optional: JS modules, CSS, JSON data, images
├── docs/architecture/c4-context.mmd   # host ↔ iframe ↔ (API) — required: a plugin is a separate deployable
├── README.md
└── CLAUDE.md
```

A plugin is a separate deployable, so capture its boundary as a C4 **context** diagram at `docs/architecture/c4-context.mmd` before building: the **Livespace host page**, **this plugin**, and the **Livespace API** (only if the plugin calls it). Show `postMessage` between host and iframe and HTTPS between iframe and API. The backend variant adds a **container** diagram — see `plugin-iframe-widget-backend`.

A single `index.html` (inline JS + CSS, vanilla or a CDN framework) suits small plugins. Split into `assets/*` once logic, state or data grows.

## Skeleton

Load the SDK and the tokens in `<head>` — the SDK sends `ready` as soon as it runs, and Livespace sends nothing until it hears it:

```html
<!doctype html>
<html lang="pl">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <script src="https://plugin-sdk.feature.livespace.dev/v1/ls-plugin.js"></script>
    <link rel="stylesheet" href="https://plugin-sdk.feature.livespace.dev/v1/ls-tokens.css" />
  </head>
  <body>
    <main id="app"></main>
    <script>
      LsPlugin.declareMetadata({ supportedContextTypes: ["deal"], minWidthPx: 260 })
      LsPlugin.createConfigForm()                      // only if Question 3 was "yes"
        .addTextInput("accountId", "ID konta", { required: true })

      LsPlugin.on("mode", (mode) => { if (mode === "config") return; boot(mode) })
      LsPlugin.on("context", ({ deal }) => renderDeal(deal))
      LsPlugin.on("config", (config) => applyConfig(config))
      LsPlugin.on("preferences", () => rerender())
    </script>
  </body>
</html>
```

`ls-plugin.mjs` is the ES-module build for bundlers. `v1` follows fixes and backwards-compatible additions; pin an exact version (`…/1.2.0/ls-plugin.js`) only when you must freeze.

**Without the SDK** (rare — e.g. a page you don't control), implement the raw protocol from [postmessage-contract.md](postmessage-contract.md): validate the host origin, send `ready` with `protocolVersion: 2`, then follow the mode.

## Modes and states

Livespace loads the page in one of three **modes**, announced before anything else:

| Mode | What to do |
|---|---|
| `view` | The real thing: render from `context` and `config`, and re-render on every `context` / `config` event — a plugin that reads them once goes stale. |
| `preview` | An admin is configuring the plugin. `context` is `null`; `config` is re-sent on every keystroke. Render the real layout with **sample data** and the live config. Don't call the API. |
| `config` | Livespace wants the settings form. Nothing drawn is shown — answer (the SDK does, from `createConfigForm()`) and skip rendering. |

Within `view`, distinguish at least these states; never show a blank iframe (it looks broken):

1. `loading` — data in flight. A skeleton, not a spinner.
2. `ready` (or a domain equivalent like `eligible`) — main render.
3. `not-configured` — `config === null`: nobody has set the plugin up. Say an administrator needs to configure it. (`{}` is configured-but-blank — not this state.)
4. `not-applicable` — context arrived but there is nothing to show (e.g. the user is not eligible). A polite placeholder.
5. `error` — fetch / parse / API failure. A message **and** a retry CTA that re-runs the failed step, not a full iframe reload.

Styling for each state, motion and toasts: [ui-design.md](ui-design.md).

## What the host gives you

| Capability | SDK | Reference |
|---|---|---|
| The viewed record, full description, live updates | `on("context")`, `LsPlugin.context` | [postmessage-contract.md § Objects](postmessage-contract.md#objects) |
| Admin settings, rendered and stored by Livespace | `createConfigForm()`, `on("config")` | [config-forms.md](config-forms.md) |
| The Livespace API as the viewing user | `api.call()`, `api.deal.get()`, `api.credentials()` | [livespace-api.md](livespace-api.md) |
| A toast in the Livespace chrome | `notify({ title, body?, variant? })` | [ui-design.md](ui-design.md) |
| Fullscreen (admin-enabled, Livespace-driven) | `on("fullscreen")`, `isFullscreen` | [ui-design.md § Fullscreen](ui-design.md#fullscreen-and-the-dialog-layout) |
| Dialog layout bounds | `setModalSize({...})` | [ui-design.md § Fullscreen](ui-design.md#fullscreen-and-the-dialog-layout) |
| Where it can be placed, minimum size | `declareMetadata({...})` | [postmessage-contract.md § Placement](postmessage-contract.md#placement-metadata) |
| Viewer's language, time zone, formats (when Livespace sends them) | `on("preferences")`, `LsPlugin.preferences` | [ui-design.md § i18n](ui-design.md#i18n-dates-and-numbers) |
| Livespace look and feel | `ls-tokens.css` / `ls-theme.css` | [ui-design.md § Design tokens](ui-design.md#design-tokens) |

Never invent a message type or an SDK method — the host ignores what it does not know.

## Security caveats

### Platform constraints (Livespace-enforced)

- **HTTPS only**, and the host must be on the account's **plugin whitelist**: `*.livespace.dev` is on it for every account; any other domain is added per account by Livespace support. Confirm the domain before promising a deploy target.
- **Never the Livespace origin.** A plugin served from the host's own origin would defeat the sandbox, so Livespace refuses to render it.
- **Iframe `sandbox`:** `allow-scripts allow-same-origin allow-forms allow-popups allow-downloads`, the same for every plugin. Scripts, own-origin cookies and storage, forms, popups (open an external tool in a new tab) and downloads work; top-level navigation of Livespace does not. Microphone may be requested (the browser prompts). Denied outright: camera, geolocation, payment, USB, MIDI, display capture, the native Fullscreen API, autoplay, motion sensors, clipboard read. `referrerPolicy` is `no-referrer`, so your server sees no `Referer`.
- **Origin validation.** Livespace checks your origin; the plugin checks the host's — the SDK does this for you. Hand-rolled code must drop inbound messages from any other origin and never post to `'*'`.

Full and current: <https://plugin-sdk.feature.livespace.dev/v1/docs/protocol.md> (Sandbox section).

### Data exposure (always disclose in README)

- API credentials arrive in the iframe and are reachable from JS / DevTools.
- Config values are readable by anyone who can open the plugin's settings, and travel in a `postMessage`.
- Anything the plugin fetches (a static config JSON, an API response) is visible in the network panel.
- Data that must stay confidential needs a backend that returns only rendered results — `plugin-iframe-widget-backend`.

## Naming & language

- Project and file names, code identifiers: English, kebab-case.
- User-facing copy: Polish first, English fallback — or the viewer's Livespace language when `preferences` provides it.

## Verification

Exercise the plugin from the bundled host harness — [examples/host-harness.html](examples/host-harness.html). It plays the Livespace side of protocol v2: waits for `ready`, sends `mode`, then `context` / `config` / `preferences`, answers credential requests (or fails them on demand), asks for the config form and renders it, toggles fullscreen, and logs `notify`, `metadata` and `modal-size`. Serve the harness and the plugin from **two different ports** — different origins, as in production — and point the harness at the plugin URL. Confirm:

1. **`view`:** `ready` arrives within a second of load; the plugin leaves `loading` for the right end state. Editing the sample record and re-sending `context` updates the plugin without a reload.
2. **`preview`:** the plugin renders sample data, never calls the API, and follows config edits live.
3. **`config`:** the plugin answers with its form (or `fields: []`), and `metadata` arrives just before it.
4. **Config states:** `null` shows the not-configured state; `{}` does not.
5. **(API-backed)** Two parallel calls both succeed; a credential error from the harness lands in the `error` state with a working retry.
6. **Fullscreen** toggles the plugin's layout; **notify** shows up in the harness log with the right variant.
7. **PL/EN:** the plugin renders with its defaults when preferences are never sent (the harness default), and switches language when the harness sends Polish or English preferences.
8. **Layout** is correct at each harness width/height preset, and at the minimums the plugin declares.
9. Opened **directly** in a browser tab (nothing embedding it), the plugin renders its empty state without throwing.

## Reference files

Loaded on demand — read the one relevant to the step you're on:

| File | Read it when |
|---|---|
| <https://plugin-sdk.feature.livespace.dev/v1/docs/index.md> and the pages in Step 0 | Always, first — the authoritative SDK docs. |
| [postmessage-contract.md](postmessage-contract.md) | Implementing without the SDK, or debugging the messages; record fields; placement metadata. |
| [livespace-api.md](livespace-api.md) | The plugin calls the Livespace API — `api.call`, error codes, one-shot credentials. |
| [config-forms.md](config-forms.md) | The plugin has admin settings. |
| [ui-design.md](ui-design.md) | Layout, tokens, sizes, fullscreen/dialog, states, toasts, i18n. |
| [examples/host-harness.html](examples/host-harness.html) | Verifying locally — a runnable host for protocol v2. |
