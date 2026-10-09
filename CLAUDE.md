# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Homebridge plugin (published as `homebridge-people-x`, platform name `PeopleX`) that exposes per-person presence sensors to HomeKit, based on pinging each person's phone and/or receiving geofence webhooks. Presence history is shown in the Eve app via `fakegato-history`. Forked from PeteLawrence/homebridge-people. Supports Homebridge 1.6+ and 2.x, Node 22/24.

## Commands

- `npm run lint` — ESLint (flat config in `eslint.config.mjs`, `--max-warnings=0`). The config lints `.js` as CommonJS; Node globals are listed by hand in `languageOptions.globals`, so add any new ones there. Custom rules must stay after the recommended configs or they get overridden.
- `npm run fix` — ESLint with `--fix`.
- There is no build step and no test suite. The plugin is plain CommonJS JavaScript in a single file (`index.js`); the TypeScript-related devDependencies are only used by the ESLint config. CI (`.github/workflows`) only runs `npm install`.
- To try changes manually, link the plugin into a local Homebridge install (`npm link`) and configure a `PeopleX` platform as shown in `README.md`.

## Architecture (`index.js`)

Three constructor-function "classes" registered in `module.exports`:

- **`PeoplePlatform`** — reads config, initialises `node-persist` storage (in `cacheDirectory`, default Homebridge persist path), creates one `PeopleAccessory` per `config.people` entry plus optional "Anyone"/"No One" `PeopleAllAccessory` instances, then starts an HTTP webhook server on `webhookPort` (default 51828).
- **`PeopleAccessory`** — one per person. Exposed as a `MotionSensor` (not an occupancy sensor) with custom Eve characteristics (LastActivation, Sensitivity, Duration) so Eve renders motion history; history entries are written to a `FakeGatoHistoryService('motion')`. Current state lives in `stateCache`.
- **`PeopleAllAccessory`** — `OccupancySensor` for "Anyone" / "No One", derived from the `stateCache` of all people; refreshed whenever any person's state changes via `setNewState`.

### Presence logic (ping + webhook interplay)

State is derived from two timestamps persisted per target in node-persist: `lastSuccessfulPing_<target>` and `lastWebhook_<target>`.

- A person is "active" if their last successful ping is within `threshold` minutes.
- The ping loop (`PeopleAccessory.ping`, every `pingInterval` ms; disabled with `-1`) only runs/applies while the last webhook is older than `threshold` — i.e. a recent webhook overrides ping results. A ping only changes state if it happened after the last webhook.
- Webhook requests `GET /?sensor=<name>&state=true|false` match a person by **name** (case-insensitive). They are queued per target and applied after `ignoreReEnterExitSeconds` (debounce: a newer webhook for the same target cancels the pending one). Applying a webhook records `lastWebhook_<target>` and calls `setNewState` directly.

Config values `threshold`, `pingInterval` and `ignoreReEnterExitSeconds` can be set per person and fall back to the platform-level values.

Note: the README says `nooneSensor` defaults to `false`, but the code defaults both `anyoneSensor` and `nooneSensor` to `true`.

## Style

Follow the ESLint rules: 2-space indent, single quotes, semicolons, trailing commas on multiline, `curly: all`, strict equality (`eqeqeq: smart`, so only `== null` is allowed). Existing code uses `var` and prototype-based classes; match it when editing.

Keep the plugin CommonJS: don't add `"type": "module"` to `package.json` (the ESLint config is `.mjs` for that reason).
