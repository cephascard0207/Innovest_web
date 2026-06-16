# IGM Group — Deployment Guide

## What this project is
- **Framework:** TanStack Start v1 (React 19, Vite 7, file-based routing in `src/routes/`)
- **Rendering:** SSR + server functions (`createServerFn`) + server routes (`src/routes/api/`)
- **Backend:** Lovable Cloud (Supabase under the hood) — auth + DB
- **Default deploy target:** Cloudflare Workers (via Nitro preset baked into the Vite config)
- **NOT** a plain Vite SPA, and **NOT** Next.js

## Can it run on GoDaddy?
GoDaddy offers three product lines. Only one of them can host this app properly:

| GoDaddy product | Works? | Why |
|---|---|---|
| Shared / cPanel hosting (Linux, PHP/Apache) | ❌ No | No Node.js runtime; cannot run SSR or server functions. |
| VPS / Dedicated Server (full root Linux) | ✅ Yes | You install Node 20+ and run the built server yourself. |
| Website Builder / Managed WordPress | ❌ No | Not a custom-code host. |

If you only have **shared hosting**, you cannot deploy this exact codebase there. You'd have to either (a) upgrade to a GoDaddy VPS, (b) deploy elsewhere (Cloudflare/Vercel/Netlify) and point your GoDaddy domain at it via DNS, or (c) rewrite the app as a static SPA and lose the server-side features.

---

## Build commands (any host)

```bash
# 1. Install dependencies (Node.js 20+ required, plus bun OR npm)
bun install            # or: npm install

# 2. Production build
bun run build          # or: npm run build

# 3. Output:
#    .output/          ← Nitro server bundle (server.js + assets) — for Node/Workers hosts
#    .output/public/   ← static assets (only useful if you somehow run SPA-only)
```

`bun run dev` runs the local dev server on http://localhost:8080.

---

## Environment variables

Create a `.env` file at the project root (already git-ignored):

```env
# Public (sent to the browser, bundled at build time)
VITE_SUPABASE_URL=https://YOUR-PROJECT.supabase.co
VITE_SUPABASE_PUBLISHABLE_KEY=YOUR_PUBLISHABLE_KEY
VITE_SUPABASE_PROJECT_ID=YOUR_PROJECT_ID

# Server-only (runtime, never bundled into client)
SUPABASE_URL=https://YOUR-PROJECT.supabase.co
SUPABASE_PUBLISHABLE_KEY=YOUR_PUBLISHABLE_KEY
SUPABASE_SERVICE_ROLE_KEY=YOUR_SERVICE_ROLE_KEY   # secret — server only
```

You can copy your current values from the Lovable Cloud panel inside Lovable, or from your Supabase dashboard → Project Settings → API.

---

## Deployment recipes

### A) GoDaddy VPS / Dedicated Server (recommended if you must use GoDaddy)

SSH into the VPS, then:

```bash
# 1. Install Node 20 and pm2
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs git
sudo npm i -g pm2

# 2. Pull the source
git clone <your-repo-or-upload-the-zip> /var/www/igm
cd /var/www/igm
npm install

# 3. Add the .env shown above
nano .env

# 4. Build
npm run build

# 5. Run the SSR server (Nitro outputs a Node-compatible server)
PORT=3000 pm2 start .output/server/index.mjs --name igm
pm2 save && pm2 startup

# 6. Front it with nginx + your domain + HTTPS (Let's Encrypt / certbot)
```

Then point your GoDaddy DNS A record at the VPS IP.

### B) Cloudflare Workers (default, easiest, free tier covers this site)

```bash
npm install
npm run build
npx wrangler deploy        # uses the wrangler.toml / nitro output
```

Then in GoDaddy DNS, add a CNAME from `www` → your Workers URL.

### C) Vercel / Netlify

Both auto-detect TanStack Start. Push the repo, set the env vars above in their dashboard, and deploy. Point GoDaddy DNS at their nameservers or use a CNAME.

### D) GoDaddy shared hosting (static-only fallback — LIMITED)

This is **only** viable if you accept losing auth, server functions, and any `/api/*` routes. You would need to refactor the app to a pure SPA build first. Not recommended; mentioned only for completeness.

---

## Files included in the package
- All source under `src/`
- `package.json`, `bun.lockb` / `package-lock.json`
- `vite.config.ts`, `tsconfig.json`, `components.json`
- Public assets and uploaded media
- This `DEPLOYMENT.md`

**Excluded** (regenerated on `install` / `build`): `node_modules/`, `.output/`, `dist/`, `.git/`, `.env`.
