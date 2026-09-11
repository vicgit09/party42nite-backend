# Party42nite Backend

This is what actually makes the app secure at any real scale. It holds
every API key server-side, rate-limits requests, and caches responses
across ALL users instead of per-device. See the "Why this exists" section
at the bottom if you want the full reasoning.

## Local setup

```bash
npm install
cp .env.example .env
```

Edit `.env`:
- `GOOGLE_API_KEY` — same Google Cloud key the app was using, with Places
  API (New) and Geocoding API enabled. It now lives ONLY here.
- `SPORTSDB_KEY` — `123` (the free tier key) is fine to start.
- `APP_SHARED_SECRET` — generate one with `openssl rand -hex 32`.

```bash
npm start
```

Server runs on `http://localhost:3000`. Test it:

```bash
curl http://localhost:3000/health
```

## Deploying (Railway or Render — both have generous free tiers)

**Railway:**
1. Push this folder to a GitHub repo (or Railway can deploy from a folder
   directly via their CLI).
2. New Project → Deploy from GitHub repo.
3. Add the three environment variables from `.env` in Railway's dashboard
   (Variables tab) — never commit the real `.env` file.
4. Railway gives you a public URL like `https://your-app.up.railway.app`.

**Render:** same idea — New → Web Service → connect your repo → add the
environment variables in the dashboard → deploy. Use `npm start` as the
start command.

Either way, once deployed, note the public URL — the mobile app needs it.

## Migrating the mobile app to use this backend

This is the other half of the fix — the backend alone does nothing until
the app actually calls it instead of calling Google/TheSportsDB directly.
In the Expo app:

1. Add the backend URL and the shared secret to `src/config.js`:
   ```js
   export const BACKEND_URL = 'https://your-app.up.railway.app';
   export const APP_SHARED_SECRET = 'the same value as your backend .env';
   ```
   (Yes, this secret still ships in the app bundle — see `auth.js` for why
   that's still a meaningful improvement over the old setup, not a perfect
   one.)
2. In `src/services/googlePlaces.js`, `geocoding.js`, `sportsFixtures.js`,
   and `googleNewsRss.js`: replace every direct `fetch()` to
   `places.googleapis.com` / `maps.googleapis.com` / `thesportsdb.com` /
   `news.google.com` with a call to the matching backend endpoint instead
   (`${BACKEND_URL}/api/places/nearby`, etc.), adding the header
   `'X-App-Secret': APP_SHARED_SECRET` to each request. The request/response
   shapes are intentionally identical to Google's own API, so this is a
   find-and-replace of the URL and headers, not a rewrite of the parsing
   logic.
3. Remove `GOOGLE_PLACES_API_KEY` from `src/config.js` entirely once the
   migration is done — the app has no reason to hold it anymore.
4. The client-side caches (`placesCache.js`) can stay — they still help
   (avoiding even a round-trip to your own backend for repeat requests
   within a session), they just don't need to be the *only* line of
   defense anymore.

This is genuinely a real chunk of work across several files — worth doing
as its own focused pass rather than rushing it.

## Why this exists

A React Native app with an API key embedded in it cannot be secured
against abuse at real scale — that key is extractable by anyone with the
shipped `.ipa`/`.apk`, and at meaningful user numbers, someone will extract
it. This backend moves every third-party key here, adds rate limiting so
one bad actor can't run up your bill, and caches responses across every
user instead of every device — which also happens to solve the Places API
quota problem the client-side cache could only patch over.

## What this does NOT do (be honest with yourself about scope)

- **Doesn't provide DDoS protection on its own.** Put this behind a CDN/WAF
  (Cloudflare is the easy option) once traffic is real — it absorbs attack
  traffic before it reaches this server at all.
- **The in-memory cache doesn't survive a restart or scale across multiple
  server instances.** Fine for a while; Redis is the upgrade once you're
  running more than one instance.
- **The shared app secret is not cryptographically bulletproof** — see
  `auth.js` for the honest tradeoff. Firebase App Check or per-install
  signed tokens are the real upgrade if this app becomes a real abuse
  target.
- **No user accounts or authentication exist in this app at all** — there's
  no personal user data to breach today, which is one genuine relief. That
  changes the moment accounts/saved-data-in-the-cloud get built.
