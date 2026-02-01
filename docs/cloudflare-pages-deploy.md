# Deploying the Wire webapp to Cloudflare Pages

This guide documents a Cloudflare Pages setup that matches the current build pipeline and output paths in this repo, plus a troubleshooting checklist and ideas to prevent the “output directory not found” error.

## 1) Build output path (critical)

The webapp bundle is written to `apps/server/dist/static` by the webpack config, **not** `dist/apps/webapp`. Cloudflare Pages must be pointed at this directory to serve the site correctly.

- Webpack output path: `apps/server/dist/static`.
- Nx build output for `webapp` is declared as `{workspaceRoot}/apps/server/dist/static`.

> ✅ **Use this as your Cloudflare “Build output directory”:** `apps/server/dist/static`.

## 2) Recommended Cloudflare Pages settings

**Required build command (copy/paste):**

```bash
yarn install --no-immutable && npx nx build webapp --configuration=production && rm -f apps/server/dist/static/min/*.map
```

**Build command (safe, no zip step):**

```bash
yarn install --no-immutable && npx nx build webapp --configuration=production && rm -f apps/server/dist/static/min/*.map
```

**Build output directory:**

```
apps/server/dist/static
```

**Root directory:**

```
/
```

### Why this works

- `npx nx build webapp --configuration=production` runs the webapp webpack build directly and writes assets into `apps/server/dist/static`.
- The root script `yarn build:prod` uses `nx run server:package`, which is meant for server packaging and runs a zip step. Cloudflare Pages expects a directory, not a zip archive.
- `rm -f apps/server/dist/static/min/*.map` removes source maps after the build (optional). This keeps the output folder lean while leaving the static assets intact.

If you must use a dev build for speed:

```bash
yarn install --no-immutable && npx nx build webapp --configuration=development
```

(Still use `apps/server/dist/static` as the output directory.)

## 3) Required environment variables for a successful build

The webpack config reads `.env`/`.env.defaults` and **throws** when required URL variables are missing. Make sure these are defined as **Plaintext** (or Secrets if needed) in Cloudflare Pages.

**Required (when FEDERATION is not set):**

- `APP_BASE` – public URL of the app (e.g., `https://my-wire-webapp.pages.dev`).
- `BACKEND_REST` – REST API endpoint (e.g., `https://prod-nginz-https.wire.com`).
- `BACKEND_WS` – WebSocket endpoint (e.g., `wss://prod-nginz-ssl.wire.com`).

**Common optional / recommended:**

- `NODE_ENV=production` (ensures production defaults in the build).
- `ENABLE_DEV_BACKEND_API=true|false` (default in `.env.defaults` is `false`).

**Node + Yarn versions:**

- This repo declares **Node >= 24.10** and **Yarn >= 4.12** in `package.json`. Configure Cloudflare’s build environment to match (set `NODE_VERSION=24.10.0` or newer, and ensure Yarn v4 is used). If you deploy successfully on a lower Node version, keep it consistent across environments to avoid rebuild drift.

## 4) Troubleshooting “Output directory not found”

If Cloudflare reports:

```
Error: Output directory "dist/apps/webapp" not found.
```

…then the output path is wrong. The webpack config writes to `apps/server/dist/static`, so set Cloudflare’s build output directory accordingly.

### Quick verification commands (local or CI)

```bash
npx nx build webapp --configuration=production
ls -F apps/server/dist/static
```

You should see `index.html`, `auth/index.html`, `unsupported/index.html`, and a `min/` folder containing JS bundles.

## 5) If you still need the zip artifact (optional)

The packaging step is part of the backend deploy flow (`nx run server:package`) and creates a zip at:

```
apps/server/dist/s3/ebs.zip
```

Cloudflare Pages **cannot** serve a zip file directly. If you used this build command and only have the zip, manually unzip it into a folder and point Pages to that folder.

## 6) Build warnings about large assets

The build log warnings about large bundles and Workbox precache limits are **warnings**, not hard failures. The important part for Cloudflare is that the output directory exists and contains the static assets.

If you want to reduce the warnings later, you can tune Workbox `maximumFileSizeToCacheInBytes` in the webpack config or split bundles further, but these are optional optimizations.

## 7) Quick checklist (copy/paste)

- [ ] Build command: `yarn install --no-immutable && npx nx build webapp --configuration=production && rm -f apps/server/dist/static/min/*.map`
- [ ] Output directory: `apps/server/dist/static`
- [ ] Node version: `24.10+`
- [ ] Env vars: `APP_BASE`, `BACKEND_REST`, `BACKEND_WS`
- [ ] (Optional) `NODE_ENV=production`

## 8) Known good Cloudflare Pages settings (example)

> **Public example (non-sensitive):** This matches a successful deployment configuration.

**Build command:**

```
yarn install --no-immutable && npx nx build webapp --configuration=production && rm -f apps/server/dist/static/min/*.map
```

**Build output directory:**

```
/apps/server/dist/static
```

**Root directory:**

```
/
```

**Environment variables (plaintext):**

```
APP_BASE=https://my-wire-webapp.pages.dev
BACKEND_REST=https://prod-nginz-https.wire.com
BACKEND_WS=wss://prod-nginz-ssl.wire.com
ENABLE_DEV_BACKEND_API=true
NODE_VERSION=20
YARN_ENABLE_IMMUTABLE_INSTALLS=false
```

If you raise `NODE_VERSION` to meet the repository’s declared `engines` requirements, keep the rest of the config the same so the build remains reproducible.
