# Admin settings — configuration forms

Load this file when the plugin needs per-account settings: an account id, a region, a toggle, a default view. **Do not build a settings screen.** The plugin declares the fields it needs; Livespace renders the form, validates it, stores the values and hands them back as `config`.

> **Authoritative source:** <https://plugin-sdk.feature.livespace.dev/v1/docs/config-forms.md>.

## Declaring a form (SDK)

Call it once, in the global scope — not inside an event handler, and not after an `await`:

```js
LsPlugin.createConfigForm()
  .addSection("Połączenie", "Jak łączymy się z Twoim kontem")
  .addTextInput("accountId", "ID konta", { required: true, maxLength: 32 })
  .addSelect("region", "Region", [
    { value: "eu", label: "Europa" },
    { value: "us", label: "USA" },
  ], { default: "eu" })
  .addCheckbox("compact", "Widok kompaktowy", { default: false })

LsPlugin.on("config", (config) => {
  if (config === null) return renderSetupPrompt()
  render(config.accountId, config.region, config.compact)
})
```

If the form is built asynchronously, call `LsPlugin.flushConfigForm()` when it is ready. A plugin that throws before reaching `createConfigForm()` answers nothing, and the admin sees an empty settings screen with no explanation.

## Field kinds

| Builder | Options | Stores |
|---|---|---|
| `addTextInput(key, label, opts?)` | `required`, `placeholder`, `minLength`, `maxLength`, `pattern`, `helpText`, `default` | string |
| `addNumberInput(key, label, opts?)` | `required`, `min`, `max`, `step`, `helpText`, `default` | number |
| `addTextarea(key, label, opts?)` | `required`, `placeholder`, `maxLength`, `helpText`, `default` | string |
| `addCheckbox(key, label, opts?)` | `helpText`, `default` — never `required` | boolean |
| `addSelect(key, label, choices, opts?)` | `choices` is the **third argument**, `[{ value, label }]`, non-empty, up to 100; `required`, `helpText`, `default` | string |
| `addSection(title, description?)`, `addHelpText(text)` | layout only | — |

Without the SDK, the `config-form` payload is `{ fields: [...] }` where each field is `{ kind, key, label, …options }` with `kind` one of `text`, `number`, `textarea`, `checkbox`, `select` (choices as `options: [{ value, label }]`), or a layout node `{ kind: "section", title, description? }` / `{ kind: "helpText", text }`.

Limits: 50 fields per form and 100 choices per select — anything beyond is dropped, not rejected. Keys must be unique; a duplicate throws at declaration time. Validation runs when the admin saves, so still read `config` defensively — a config saved earlier predates any constraint added later.

## Rules

- **Always answer in `config` mode**, even with no settings. The SDK sends an empty form after a short grace period if nothing was declared; the raw protocol has to send `config-form` with `fields: []` itself.
- **`undefined` ≠ `null` ≠ `{}`.** `undefined`: not delivered yet. `null`: never configured — render a setup prompt. `{}`: configured, everything left blank — don't show the setup prompt again.
- **`preview` mode re-sends `config` on every keystroke** so the admin watches the plugin react. A plugin that reads `config` only once looks broken while it is being configured.
- **Config values are not secret.** Anyone who can open the plugin's settings can read them, and they travel to the iframe in a `postMessage`. Never ask for an API key or token here — a third-party secret means a backend (`plugin-iframe-widget-backend`).

Secret fields (`addSecretInput`), encrypted for the plugin's own backend, appear in the published SDK docs as a **preview**: the Livespace services behind them are not live, and the admin screen does not show them yet. Don't build on them until the SDK docs drop the preview label.
