# Appdrop: fullstack Next.js publish (Cloudflare Worker via OpenNext)

Part of the `appdrop` skill — read SKILL.md first for token setup, the
static/fullstack decision, prep, and reporting. `$BASE` is the Appdrop base
URL from SKILL.md. The Worker runs on the creator's own Cloudflare account;
Appdrop only registers its URL.

## Preflight

- Run `npx wrangler whoami`. If not logged in, stop and ask the user to run
  `npx wrangler login` (their own Cloudflare account).
- The creator may use their own Supabase for private app data, but Appdrop
  profile/results data must go through the Appdrop SDK, never directly through
  the creator's Supabase.
- Do not set `output: "export"` in next.config for this path.

## Scaffold OpenNext (if missing — pin majors)

- Dev dependencies: `@opennextjs/cloudflare@^1` and `wrangler@^4`.
- `wrangler.jsonc` at the project root:

```json
{
  "name": "<cloudflare-worker-name>",
  "main": ".open-next/worker.js",
  "compatibility_date": "2025-03-25",
  "compatibility_flags": ["nodejs_compat"],
  "assets": { "directory": ".open-next/assets", "binding": "ASSETS" }
}
```

- `open-next.config.ts`:

```ts
import { defineCloudflareConfig } from "@opennextjs/cloudflare";
export default defineCloudflareConfig();
```

## Runtime env and secrets

- Worker runtime variables live on Cloudflare, never accidentally baked from
  local `.env`. Sensitive values are Worker secrets — never wrangler vars and
  never `NEXT_PUBLIC_*`.
- Supabase: a publishable/anon key may be browser-side with RLS; a secret or
  legacy service_role key belongs only in server code, stored as a Worker
  secret, under the exact env names the code reads.
- Deploy uploading runtime secrets, e.g.
  `npx opennextjs-cloudflare@^1 deploy -- --keep-vars --secrets-file .env.production`
  (never commit the secrets file; `--keep-vars` preserves dashboard-set values).

## Verify before registering

- `npx wrangler secret list --name <worker-name> --json` — compare against the
  env names the backend code actually reads.
- Smoke test the deployed Worker URL and every backend API route the app
  needs. HTML where JSON is expected = stop and fix hosting/env first.
- If the app uses the Appdrop SDK, confirm the script tag is in the deployed
  HTML; if it saves results, verify the result flow calls
  `window.appdrop.saveOutput`. If you cannot complete a signed-in end-to-end
  saveOutput test, say exactly why and mark it not fully verified.

## Register with Appdrop

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
    "description": "<one-sentence app description>",
    "entry": "worker",
    "frame": { "mode": "responsive" },
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

Manifest permissions/outputs follow the same rules as `appdrop.json` in
SKILL.md. Use `"frame": { "mode": "contained", "width": 460, "height": 932 }`
when the useful app surface is a fixed portrait/canvas shape rather than a
responsive page, or `"frame": { "mode": "mobile", "width": 460, "height": 932 }`
for phone-first apps that should appear as a phone-sized Appdrop surface. The
app fills that surface edge to edge, so render the phone UI full-bleed.
The response `data.publish` contains the values to
report (claimUrl when anonymous, appdropUrl, hostedUrl, slug) — report them per
SKILL.md.
