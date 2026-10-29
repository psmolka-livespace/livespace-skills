# plugin-builder

Build a **Livespace plugin** — a custom extension embedded inside a Livespace profile (**Deal, Company, Person, or Space**) or opened from the **apps menu**. An account administrator adds it (HTTPS URL on a whitelisted domain), fills in its settings form, and configures where it shows, its size, whether it may open fullscreen or in a dialog, and who can see it.

This plugin is a small **router + two builder sub-skills**:

| Skill | Role |
|---|---|
| `plugin-builder` | Intake + routing. Captures *what the plugin should do*, picks the plugin type, hands off. Contains no build detail. |
| `plugin-iframe-widget` | Builds a **client-only** iframe widget on the Livespace plugin SDK (`LsPlugin`, protocol v2): live record context, admin settings forms, host toasts, fullscreen/dialog layouts, design tokens, and the Livespace API. Ships a local host harness for testing. |
| `plugin-iframe-widget-backend` | Adds a **backend** (Node / PHP-Symfony / Python) for plugins that must keep third-party secrets server-side. |

## How it flows

```
plugin-builder  ──(purpose captured, type = iframe widget)──▶  plugin-iframe-widget
                                                                     │
                                        (needs server-side secrets?) │
                                                                     ▼
                                                     plugin-iframe-widget-backend
```

## Platform docs

This plugin documents the Livespace plugin platform against the official public docs:

- **Plugin SDK docs** (authoritative, versioned with each SDK release): <https://plugin-sdk.feature.livespace.dev/v1/llms.txt>, page by page starting at <https://plugin-sdk.feature.livespace.dev/v1/docs/index.md>
- Platform overview and security model: <https://plugin.docs.livespace.io/>
- Full API reference (Postman collection): <https://api-docs.livespace.io/>

The builder skills read the SDK docs first and defer to them; their reference files are summaries.

## Usage

Once installed, just ask Claude to build a Livespace plugin (e.g. *"build a Livespace plugin that shows this deal's commission"*). The `plugin-builder` skill triggers, captures the purpose, and routes to the right builder.
