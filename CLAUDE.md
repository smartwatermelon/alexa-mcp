# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An MCP server (stdio) that audits and cleans up an Amazon Alexa account — routines (full CRUD),
smart-home devices (orphan deletion), groups, lists, notifications, media/volume/TTS — on top of
`alexa-remote2` and Amazon's **undocumented, reverse-engineered** Alexa API. `ARCHITECTURE.md` is
the authoritative reference for the auth flow, endpoints, and the routine write-API wire format;
read it before touching `alexa.ts` or `routines.ts`.

## Commands

```bash
npm ci
npm run build                      # tsc -> dist/ (no bundler, no tsx; runs on plain Node >= 20)
npm test                           # build + node --test (all test/*.test.mjs)
node --test test/util.test.mjs     # one test file (run `npm run build` first)
node --test --test-name-pattern="double-encodes"   # tests matching a name
npm run doctor                     # config/token/login-URL check, no network
npm run auth                       # one-time interactive proxy login -> .auth/alexa.json
npm run smoke                      # read devices/routines/scenes with the stored token (live account)
```

- Tests import from `../dist/*.js`, so they run against **compiled output** — always rebuild
  before `node --test`. Only pure helpers are tested (`util.ts`, the builders in `routines.ts`);
  nothing in the suite touches the network or needs a token.
- There is no linter; `tsc` with `strict` is the only static check.
- CI (`.github/workflows/ci.yml`) runs build + `node --test` on Node 20, 22, 24.

## Architecture

- `src/index.ts` — the whole MCP server: every tool is a `server.registerTool(...)` call in one
  file. Read tools come first; write tools are registered **only inside `if (ALLOW_WRITE)`**
  (`ALEXA_MCP_ALLOW_WRITE=1`), and most of them additionally require `confirm: z.literal(true)` in
  their input schema. Keep both layers when adding a write tool.
- `src/alexa.ts` — `initAlexa()`: wraps `alexa-remote2` init. Handles the library's quirks (no
  `ready` event; the init callback fires twice during proxy login and again on every token
  refresh → single-resolve `settled` guard). Persists the token on every `'cookie'` event.
- `src/store.ts` — field-agnostic persistence of the full `cookieData`/`formerRegistrationData`
  object to `.auth/alexa.json` (mode 0600). Storing only the cookie would register a new Amazon
  device on each login.
- `src/routines.ts` — routine write-API: builds triggers/sequences/body, validates actions,
  `writeRoutine` (create/update), `setRoutineEnabled` (re-encode + PUT with flipped status — there
  is no status-only endpoint).
- `src/util.ts` — pure, dependency-free helpers (`flattenAppliances`, `collectRoutineTargets`).
  Put new testable logic here rather than inline in `index.ts`.
- `src/auth-cli.ts`, `doctor.ts`, `smoke.ts` — the `npm run` entry points.

### Conventions in `index.ts`

- Tool `inputSchema` is a Zod **raw shape** (plain object of fields), not `z.object(...)`.
- Return via `ok(data)` (JSON text) / `fail(msg)` (`isError`); wrap errors with `hint(e)`, which
  appends the "run `npm run auth`" advice for missing/expired tokens.
- `getAlexa()` is a lazy singleton that resets on failure so the next call retries.
- `alexa-remote2` is callback-style: use `call(cb => alexa.x(cb))`; use `callRetry` for read
  endpoints that intermittently return an empty body ("no JSON"/"no body").
- Endpoints the library doesn't wrap go through `(alexa as any).httpsGet(true, path, cb, { method, data, headers })`.
  Always `encodeURIComponent` ids in paths — `applianceId`s contain `#`, and an unencoded DELETE
  returns success but deletes nothing.
- Destructive tools re-query afterwards to verify the effect (e.g. device delete reports
  `verifiedGone`); device delete refuses if a routine/group references the device unless `force`.
- stdout is the MCP transport: log only with `console.error`.

### Routine write-API rules (details in ARCHITECTURE.md)

- Writes use the **non-`v2`** path `/api/behaviors/automations[/{id}]`; the `v2` path is GET-only.
- The write body is not the read object: trigger `payload` is a **double-encoded JSON string**,
  and every action's `operationPayload` must first be normalized via
  `POST /api/behaviors/operation/validate`.
- `withBackoff` retries only idempotent calls (GET/PUT/DELETE/validate). Never retry the create
  POST — a lost response would duplicate the routine.
- `dryRun` is implemented on `alexa_create_routine`, `alexa_update_routine` and
  `alexa_set_routine_enabled` (returns the planned method/path/body without sending).

## Runtime notes

- The server never starts a login proxy; with no token it fails with a hint to run `npm run auth`.
- `AUTH_PATH` (`.auth/alexa.json`) resolves against the **process cwd**; set
  `ALEXA_MCP_AUTH_PATH` when launching the server from elsewhere.
- Defaults target Germany: `amazon.de`, `de-DE`, marketplace `A1PA6795UKMFR9`
  (`ALEXA_MCP_AMAZON_PAGE`, `ALEXA_MCP_ACCEPT_LANGUAGE`, `ALEXA_MCP_MARKETPLACE_ID`). Full env-var
  list is in README.md.
- `npm run smoke` and any write tool act on a **real Amazon account**; there is no sandbox/mock.
- Routine listing (`getAutomationRoutines`) returns **at most 50** routines regardless of `limit`;
  no working paging parameter was found. Above 50 routines, the list, the broken-reference audit and
  the device-delete reference check silently miss the rest.

## Local install status (this checkout only — not upstream)

This file and the items below live only on branch `local/install`, pushed to the `fork` remote
(smartwatermelon/alexa-mcp). Never push them to `origin` or include them in an upstream PR.

- Installed as a **user-scope, read-only** MCP server `alexa` (no `ALEXA_MCP_ALLOW_WRITE`), with
  `ALEXA_MCP_AMAZON_PAGE=amazon.com`, `ALEXA_MCP_ACCEPT_LANGUAGE=en-US`,
  `ALEXA_MCP_PROXY_BIND=127.0.0.1`, `ALEXA_MCP_AUTH_PATH=~/.local/state/alexa-mcp/alexa.json`.
  Live-verified 2026-11-17: all 15 read tools work.
- `package-lock.json` bumps `alexa-cookie2` 5.0.5 → 5.0.6 (committed only on `local/install`).
- `node_modules/alexa-cookie2/lib/proxy.js` is patched in `onProxyRes` to delete the
  `content-security-policy` and `strict-transport-security` headers. Without it, Amazon's `/ax/claim`
  sign-in page sends `upgrade-insecure-requests`, and the browser posts the email form to
  `https://127.0.0.1:<port>`, which fails. `npm ci` removes the patch; re-apply before `npm run auth`
  with `git apply patches/alexa-cookie2+5.0.6.patch` (saved on branch `local/install`).
- Run `npm run auth` in a separate terminal: it prints the refresh token and device keys to stderr.
- Inside the Claude Code Bash sandbox, the login proxy and live API calls fail with `EPERM`; they
  need the sandbox disabled.
- Do not enable write tools without re-reading the security review: `confirm: true` is set by the
  model, not a human; `alexa_text_command` can do anything a voice command can; and
  `alexa_trigger_routine` has no `confirm`. The API returns one more routine than the app shows
  (a hidden system routine, probably Home Modes/Guard) — do not edit or delete those via the tool.
