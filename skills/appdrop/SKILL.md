---
name: appdrop
description: >
  Appdrop hosts web apps and games published by agents and gives users a home
  for what they make. Publish a static build or a fullstack Next.js app to a
  live URL, attached to the user's Appdrop account directly or via a 24-hour
  claim link. Use when asked to "publish this to Appdrop", "publish this app",
  "host this", "deploy this", "put this game online", "share this app",
  "give me a link to this project", or "claim my app".
---

# Appdrop

**Skill version: 0.1.0**

Appdrop publishes web apps and games to live URLs. Apps belong to the user:
publishes made with a signed-in token attach to their account immediately;
publishes made with an anonymous grant return a **claim link** the user must
open within 24 hours to keep the app.

To install or update this skill: `npx skills add Danger-Testing/appdrop-skill --skill appdrop -g`
Without npm: `curl -fsSL https://www.appdrop.com/install.sh | bash`

## Current docs

This file is canonical at **https://www.appdrop.com/skill/SKILL.md** — fetch it
at the start of a publish if your installed copy may be stale. If this text and
live API behavior disagree, trust the live API.

## Requirements

- Binaries: `curl`, `node` (≥ 18)
- Base URL: `https://www.appdrop.com` (override with `$APPDROP_BASE_URL` or a
  base URL given in the conversation; `$BASE` below means this value)
- A publish token, resolved in this order:
  1. `$APPDROP_PUBLISH_TOKEN` environment variable
  2. `.appdrop/token` file in the project (from an earlier exchange)
  3. A grant code carried in an agent prompt from `$BASE/create?mode=agent`
     (the prompt itself contains the exchange command)
  4. The device sign-in flow below

There is NO Appdrop npm package — never run `npx appdrop` or `npx appstar`;
those names are not owned by Appdrop.

## Token hygiene

- Keep everything under `.appdrop/`, untracked and mode 700:

```bash
mkdir -p .appdrop && chmod 700 .appdrop
grep -qxF '.appdrop/' .gitignore 2>/dev/null || printf '.appdrop/\n' >> .gitignore
```

- Never print, echo, or commit the token or the raw exchange responses. Use the
  token only inside Authorization headers via `$(cat .appdrop/token)`.
- Tokens live 24 hours and are scoped to publishing only (`publish:register`,
  `publish:upload`). You may keep `.appdrop/token` for republishing; delete the
  response JSON files once the token is captured, and delete `.appdrop/`
  entirely when you are done publishing.

## Device sign-in (when no token or grant is available)

1. Start an authorization and show the user the approval URL:

```bash
curl -s -X POST "$BASE/api/publish/device/start" -H "content-type: application/json" \
  -d '{"clientName":"<your agent name>"}' -o .appdrop/device.json
node -e "const d=require('./.appdrop/device.json').data.device;console.log(d.verificationUriComplete,'(code',d.userCode+', expires',d.expiresAt+')')"
```

2. Tell the user: "Open this link, sign in to Appdrop, and approve publishing."
3. Poll every 3 seconds (up to 10 minutes) until approved, then capture the token:

```bash
curl -s -X POST "$BASE/api/publish/device/poll" -H "content-type: application/json" \
  -d "{\"deviceCode\":\"$(node -e "process.stdout.write(require('./.appdrop/device.json').data.device.deviceCode)")\"}" -o .appdrop/exchange.json
node -e "const d=require('./.appdrop/exchange.json')?.data?.device;if(d?.status==='authorization_pending'){console.log('pending');process.exit(0)};const t=d?.publishToken?.token;if(!t){console.error('failed:',JSON.stringify(d||require('./.appdrop/exchange.json')?.error));process.exit(1)};require('fs').writeFileSync('.appdrop/token',t,{mode:0o600});console.log('token saved')"
```

Tokens from this flow are owned by the signed-in user, so publishes attach to
their account directly — no claim link needed. Grant-based tokens from a pasted
agent prompt are anonymous and return a claim link instead.

## Decide: static vs fullstack

Publish as **fullstack Next.js** if the project has any of: Next.js API routes
or App Router route handlers, server actions, middleware or SSR that needs a
server, or frontend fetches to its own root-relative endpoints like
`/api/game/start`. Otherwise publish as **static**. Never static-upload a
fullstack app.

If the repository contains multiple apps (monorepo) or appears to be the
Appdrop platform itself, stop and ask the user which app to publish.

## Prepare

- Infer a clear app name and URL-safe slug from package.json, README, or
  visible product copy.
- Detect the package manager from the lockfile (bun.lock → bun, pnpm-lock.yaml
  → pnpm, yarn.lock → yarn, package-lock.json → npm); confirm the project
  installs and builds with it.
- Audit env vars: static builds bake `VITE_*`/`NEXT_PUBLIC_*` values, so they
  must hold production values and never secrets — grep the build output for
  secret-shaped strings (`service_role`, `sk_`, `"secret"`) and stop if any
  appear. Fullstack runtime values live on Cloudflare (below), never baked
  from local `.env`.
- The hosted site is served under a route, so assets must resolve relatively:
  Vite `base: "./"`, Next static export `assetPrefix`, CRA `"homepage": "."`,
  SvelteKit `paths.base`.
- Most sites neither read the embedded user nor save results — skip the
  manifest and SDK below for those. Otherwise create `appdrop.json` at the
  project root:

```json
{
  "name": "<app name>",
  "slug": "<url-safe-slug>",
  "entry": "index.html",
  "spaMode": true,
  "permissions": ["profile:basic"]
}
```

If the app saves results, also add `"outputs:create"` to permissions and an
`outputs` array of `{ "type", "displayName", "schemaVersion": 1 }` entries.

## Appdrop SDK (only if the app reads the user or saves results)

- Load `$BASE/appdrop-sdk.js` at the document level (`next/script` for
  Next.js, a script tag in index.html for Vite/static), never from inside a
  React component.
- `getUser()` and `saveOutput()` REJECT outside the Appdrop iframe, so gate on
  `window.appdrop?.isEmbedded?.()` and degrade gracefully.
- Save the main result with `await window.appdrop.saveOutput({ output_type,
  title, summary, source_url, visibility: "private", data })`. The SDK source
  documents the full API.

## Publish: static

Build, then run the one-command publish client — it hashes files, uploads only
what changed, registers the app, and prints the result JSON:

```bash
curl -fsSL "$BASE/appdrop-publish-client.mjs" -o .appdrop/publish-client.mjs
node .appdrop/publish-client.mjs --slug <slug> --name "<app name>" --dir dist --base-url "$BASE" --token-file .appdrop/token
```

Limits: 600 files, 50 MB per file, 200 MB total — prune sourcemaps and
oversized media. Republishing only transfers changed files, so iterate freely.

Smoke test: request the printed `hostedUrl` and one asset referenced by its
index.html, expecting 200s. A 404 on assets usually means a wrong base path —
fix the base option, rebuild, republish.

## Publish: fullstack Next.js (Cloudflare Worker via OpenNext)

- Preflight: `npx wrangler whoami`. If not logged in, stop and ask the user to
  run `npx wrangler login` (their own Cloudflare account). The creator may use
  their own Supabase for private app data, but Appdrop profile/results data
  must go through the Appdrop SDK.
- Do not set `output: "export"` for this path.
- Scaffold OpenNext if missing (pin majors): dev deps
  `@opennextjs/cloudflare@^1` and `wrangler@^4`; `wrangler.jsonc` with
  `"main": ".open-next/worker.js"`, `"compatibility_flags": ["nodejs_compat"]`,
  `"assets": { "directory": ".open-next/assets", "binding": "ASSETS" }`; and
  `open-next.config.ts` exporting `defineCloudflareConfig()`.
- Runtime env lives on Cloudflare: sensitive values as Worker secrets, never
  wrangler vars and never `NEXT_PUBLIC_*`. Supabase secret/service_role keys
  belong only in server code, stored as Worker secrets. Deploy with e.g.
  `npx opennextjs-cloudflare@^1 deploy -- --keep-vars --secrets-file .env.production`
  (never commit the secrets file; `--keep-vars` preserves dashboard-set vars).
- Verify before registering: `npx wrangler secret list --name <worker> --json`
  against the env names the code reads; smoke test the Worker URL and every
  backend route (HTML where JSON is expected = stop and fix first); confirm
  the SDK script tag if the app uses it.
- Register the Worker with Appdrop:

```bash
curl -s -X POST "$BASE/api/publish/hosted-app/register" \
  -H "authorization: Bearer $(cat .appdrop/token)" -H "content-type: application/json" \
  -d @.appdrop/register.json -o .appdrop/register-result.json
```

`register.json` payload:

```json
{
  "slug": "<url-safe-slug>",
  "appName": "<app name>",
  "hostedUrl": "https://<worker-url>",
  "deployment": {
    "provider": "cloudflare_workers",
    "runtime": "nextjs",
    "type": "fullstack_nextjs",
    "url": "https://<worker-url>",
    "workerName": "<worker-name>"
  },
  "files": [],
  "manifest": {
    "name": "<app name>",
    "slug": "<url-safe-slug>",
    "entry": "worker",
    "runtime": "nextjs",
    "spaMode": false,
    "permissions": [],
    "outputs": []
  },
  "storageProvider": "cloudflare_worker",
  "storagePrefix": "<worker-name>",
  "uploadedFrom": "agent-next-worker"
}
```

Manifest permissions/outputs follow the same rules as `appdrop.json` above.

## Errors

- 401/403 on publish → token expired or wrong scope; get a fresh token
  (device flow above, or a fresh agent prompt from `$BASE/create?mode=agent`).
- 409 `device_authorization_consumed`, or device status `expired`/`denied` →
  the code is dead; start a new device authorization or ask for a fresh prompt.
- 403 `slug_owned_by_another_account` → pick another slug and retry.
- Republishing the same slug from the same account updates the app in place
  and returns a fresh claim link (use this if a claim link expires unclaimed).

## What to tell the user

- If the result JSON contains a `claimUrl`: lead with it —
  `"Published to Appdrop: <claimUrl> (valid for 24 hours)"` — it is how the
  user attaches the app to their account. Then list `claimExpiresAt`,
  `appdropUrl`, `hostedUrl`, and `slug`.
- If there is no `claimUrl` (signed-in token): the app is already on their
  account — lead with `appdropUrl`, then `hostedUrl` and `slug`.
- Always state whether registration succeeded (relay error codes verbatim),
  whether the SDK is loaded, and whether `saveOutput` was verified
  (yes/no/not applicable).
