# Client Extensions (REMOVED in v2.3.4)

> **⚠️ FEATURE REMOVED — v2.3.4.** The client extension feature no longer exists in Marinara Engine, in any form. All extension CSS/browser/server payload execution is gone; the Settings surface, the client hooks, the scoped `marinara` API contract, and the `/api/extensions` routes were all deleted (`extensions.routes.ts` and `extension-transfer.ts` no longer exist on `main`). There is **no legacy or compatibility mode**. On the **first v2.3.4 startup**, the server **permanently erases every retained server extension record and every `extension-storage:*` setting**, and the UI-state migration removes any browser-local extension records. Do not recommend, design, or debug extensions — route users to the surviving surfaces below.

This file is kept only to answer **"where did extensions go?"** and to make older references to the feature intelligible.

## ⚠️ Data Loss on Upgrade

Anything an extension stored via `marinara.storage` (the server-backed `extension-storage:<id>` key/value bags, previously documented as durable and cross-device) is **destroyed on the first v2.3.4 startup**. There is no export step, no grace period, and no in-app recovery path. The only ways to recover extension code or storage are:

- a **pre-2.3.4 backup** of the app database, or
- a **previous zip export** of the extension folder (the old `marinara-extensions.json` envelope format).

If a user upgraded without either, the data is gone.

## Where Did Extensions Go? (Migration Routing)

| If the extension did… | Use instead |
| --- | --- |
| Styling / CSS tweaks / themes | Native **Settings → Appearance** controls, plus server-synced **custom themes** (`/api/themes`). **Settings → Addons now hosts themes only.** |
| Text transforms (rewriting prompts or model output) | **Regex Scripts** (per-character and per-preset; see `custom-tools.md`). |
| Model-invoked actions (the AI *does* something) | **Custom tools** — see `custom-tools.md`. |
| Per-turn behavior (runs alongside generation) | **Agents** — see `agents.md`. |
| Third-party distributable functionality | **Downloadable agent packages**, via the official Download Agents catalog or the new **disabled-by-default custom GitHub agent repositories** in Agents Manager (v2.3.4, #3861). |
| Arbitrary client DOM/JS | **No replacement.** Fork the engine or submit an upstream PR. |

**Professor Mari** no longer creates or edits extensions — her instructions for them were removed with the feature. She can still build **themes, agents, and custom tools**.

## What the Feature Was (Historical, pre-2.3.4)

Client extensions were user-installed CSS + JavaScript payloads — roughly "userscripts + userstyles with a proper API surface." Each enabled extension's CSS was injected as a `<style>` tag, and its JS was executed as a Blob-imported ES module with a scoped `marinara` API object. That API offered `addStyle` (inject CSS), `addElement` (add DOM nodes), `apiFetch` (call the engine's own `/api/*` routes, with a denylist for `/extensions` and `/admin`), auto-cleaned event listeners/observers/timers (`on`, `observe`, `setInterval`, `setTimeout`, `onCleanup`), and — from v2.2 — `marinara.storage`, a server-persisted per-extension key/value bag. Extensions were persisted server-side via `/api/extensions` CRUD routes, managed under Settings → Addons, and could travel as zip packages. Typical uses were style tweaks, debug overlays, small UI additions (token counters, favorites bars), and external integrations via `fetch`. All of that is gone in v2.3.4.

`packages/client/src/components/layout/CustomThemeInjector.tsx` still exists, but **only as the custom-theme CSS injector** — the extension-loading half (the Blob ES-module import and the `__marinaraExtensionApis` registry) was deleted. Do not treat it as an extensions source of truth.
