# Changelog

All notable changes to the `plugin-builder` plugin are documented here.

## 2.0.0

Rebuilt on the Livespace plugin SDK and protocol version 2. Plugins built from 1.x keep working (Livespace serves them protocol 1), but miss everything below.

- **SDK-first.** The builder scaffolds on `ls-plugin.js` (`LsPlugin`) and reads the published SDK docs first, page by page, treating them as authoritative.
- **Protocol 2 handshake:** `ready` → `mode` (`view` / `preview` / `config`). The host origin is validated both ways, replacing the old "post to `'*'`" advice.
- **Rich, live record context:** the full description of the deal / company / person / space (owners, custom fields, stage, value, …), re-sent on every change. Intake now prefers it over API calls.
- **Admin settings** through configuration forms that Livespace renders and stores (new `config-forms.md`).
- **Host toasts** (`notify`), **fullscreen** (`fullscreen-changed`), **dialog layout** bounds (`setModalSize`), **placement metadata** (`declareMetadata`), and the viewer's **preferences** (language, time zone, formats) when Livespace sends them.
- **Apps menu** plugins with no record.
- **Design tokens** (`ls-tokens.css` / `ls-theme.css`, px on purpose) replace the hand-picked palette. Heights are presets or a custom height, not a fixed S/M/L.
- `LsPlugin.api.call` with typed error codes; credentials only in `view` mode.
- Sandbox, permission and whitelist rules updated (`*.livespace.dev` is whitelisted for every account).
- **Host harness** rewritten for protocol 2: modes, config-form rendering, credential errors, fullscreen, toasts, metadata and the protocol 1 fallback.
- Backend skill: per-account settings come from `config`, not env vars. The SDK's previewed backend token and secret fields are mentioned but not built on.

## 1.0.0

Initial release.

- `plugin-builder` router skill (purpose capture + type routing).
- `plugin-iframe-widget` client-only builder (postMessage contract, Livespace API handshake, UI/i18n reference, local host harness).
- `plugin-iframe-widget-backend` backend-enabled builder (Node / Symfony / Python stack layouts, JIRA Cloud notes).
