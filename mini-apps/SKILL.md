---
name: mini-apps
description: "FIRST DRAFT, needs review. Many rules are too prescriptive; treat them as defaults to question. Build lightweight team apps as a single index.html (Preact + htm, no build) synced through TinyBase on a Cloudflare Worker, with an optional zero-dependency server.mjs. Optionally uses Giga for credentials and tools."
---

> **⚠️ First draft: needs review before relying on it.**
> This was written in one go from a single project (Magnet). Several rules are more prescriptive than they should be, and some details (room protection, hosting, per-person vs team credentials) haven't been settled. Treat everything below as **sensible defaults, not requirements**: deviate whenever the app needs it, and flag anything that seems wrong so the skill can be corrected.

# Mini Apps

The goal is the lightest possible app that a team can actually use.

- **Ideal:** one `index.html` that runs from anywhere (any static host, or a file on disk). No build step and no npm install.
- **Data** lives in TinyBase and syncs through a small Cloudflare Worker. So "no backend" really means "no backend we write".
- **Add `server.mjs` only when you must:** calling an API that blocks browsers (CORS), holding a secret, or signing something. Keep it one file with no dependencies.

## 1. Files

```
app/
  index.html     the whole front end: HTML, CSS, one <script type="module">
  server.mjs     optional: node server.mjs → http://localhost:3000
  md/            content as Markdown (guides, hubs), loaded at runtime
  .env           optional server keys, gitignored
  .oauth.json    OAuth tokens (server only, only if you use Giga), gitignored
```

- **Order inside `index.html`:** imports → constants/config → store and sync → data helpers → small components → pages → routing → `App` → `render`.
- **Section markers:** `// ---------- Name ----------` for sections, plus a short comment explaining *why* wherever a choice isn't obvious.

## 2. Front end

- **Libraries, loaded from esm.sh with pinned versions:**
  - `preact@10.x` and `preact/hooks`
  - `htm` (`const html = htm.bind(h)`)
  - `marked` for Markdown
  - `tinybase@10.x`
  - For TinyBase's React-based packages (e.g. the Inspector), alias React to Preact: `?alias=react:preact/compat,react-dom:preact/compat&deps=preact@<version>`.
- **Components are functions using hooks.** No classes, no state library: TinyBase is the state.
- **A `useTable(name)` hook** subscribes to a table and returns rows as `[{ id, ...row }]`. Derive everything else during render.
- **Hash routing, one route per view:**
  - Examples: `#/`, `#/settings`, `#/p/<slug>`, `#/p/<slug>/<tab>`, `#/p/<slug>/<tab>/<item>/<subtab>`.
  - One `parseRoute()` regex, a `useRoute()` hook listening to `hashchange`, and `go(hash)` for navigation.
  - Query strings after `?` hold view options, e.g. `?page=<id>`.
  - Every screen and sub-tab has its own URL, so it can be linked and survives a reload.
- **Remember per-browser conveniences in `localStorage`** (last period picked, last feed URL, "which team member am I"). Never shared data.

## 3. Layout and UI patterns

- **Top bar, one row:**
  - **Left:** a back arrow, the context name, and **icon-only navigation** with `data-tip` tooltips. The active icon is underlined.
  - **Right:** context controls (e.g. a period picker), an initials avatar menu ("who am I"), a `⋯` menu (settings, config, import, how it works), and a **sync dot**. The dot shows text only when something is wrong.
- **Details open in a side panel** on the right, opened from slim list rows, not on a new page. Escape closes it.
- **Creating things happens in a modal:** a `+ Add` button opens a small form. Don't put big forms at the top of list pages.
- **Sub-sections use one compact header row:** back arrow, small bold name, text tabs on the right, and a cog for settings (which opens a modal).
- **Lists:** a search box and filter chips above the rows, newest first, loading about 50 at a time with a "Show more" button.
- **Empty states always say what to do next.**
- **Multi-step flows** (e.g. an import): *load → search/filter → review (✕ to exclude, ↺ to undo) → tag → confirm*. Never import everything blindly.
- **Charts are hand-written SVG** (bars and lines). No chart library.

## 4. Styles and content

- **Load base-styles before your own `<style>`:** `https://cdn.html-first.com/base-styles-<version>.css`. It uses `@layer base, components`.
- **Markdown content** is rendered with `marked` inside `.ui-styled-text`.
- **Your CSS:**
  - Define design tokens on `:root` (`--ink`, `--muted`, `--line`, `--card`, `--bg`, `--accent`, `--radius`).
  - Give every feature's classes a **feature prefix** (`bm-row`, `dewey-item`, `hub-tabs`). Generic names collide: a bookmarks `.brow` once broke the leaderboard's `.brow`.
- **Long-form content goes in `md/*.md`,** fetched at runtime:
  - Use `{{PLACEHOLDER}}` tokens for values that come from the code, so the docs never go out of date.
  - Build tables of contents from `#`, `##` and `###` headings, with matching heading IDs.
  - Name hubs after their file (`SEO-hub.md` → "SEO hub"), so the doc's own `#` title can be anything.

## 5. Data: TinyBase

- **Store:** `createMergeableStore()`. A CRDT with **last-write-wins per cell**:
  - Offline edits to *different* cells both survive.
  - With *the same* cell, the later edit silently wins.
  - Text inside a cell is never merged.
- **Local copy: IndexedDB,** via `createIndexedDbPersister(db, '<app>')`. Call `load()` once, then `startAutoSave()`.
  - **Never localStorage.** The store, including its deletion history, soon passes the ~5 MB limit, and saves then fail silently.
  - Log persister errors with `onIgnoredError` rather than swallowing them.
- **Sync:** `createWsSynchronizer(db, new WebSocket('wss://<worker>/<room>'), timeoutSecs, onSend, onReceive, onIgnoredError)`.
  - One room (URL path) per app.
  - Reconnect with backoff, with the status going `connecting → synced → offline`.
- **Table conventions:**
  - Flat rows with an explicit owner key (`projectId`), plus `createdAt`.
  - Many-to-many relationships get **join tables** with composite IDs (`bookmark_tags` with id `${bookmarkId}|${tagId}`), so writes are idempotent.
  - Cells can hold objects (`metadata`), but an object merges as one value, so keep fields that are edited independently in separate cells.
  - Use deterministic IDs for derived data (e.g. `[ownerId, source, interval, date].join('|')`), so re-fetching overwrites rather than duplicates.
- **Deletions are kept forever** as "tombstones", so every device learns about them. Avoid bulk insert-then-delete of throwaway data: 13,000 deleted metric rows once made up most of an 8 MB store.
- **Migrations run after sync:** fix old shapes in a transaction once synced data has arrived.
- **The synced store has no authentication. Anyone with the room URL can read it:**
  - Never store secrets in plain text.
  - Never store personal data (lead names, emails, message bodies). Store counts and IDs.
  - Encrypted team keys are fine (see §7).
- **Expose `window.<app> = { db }`** for debugging from the console.

### Sync gotchas, learned the hard way

- **The dot saying "synced" only means downloading worked.** A browser that reconnects with changes made *offline* may never upload them: the server's catch-up pull times out on large stores and fails silently.
- **Include a "Sync now" action** (⋯ menu, plus `window.<app>.syncNow()`):
  1. Download the server's copy through a temporary store.
  2. Diff it cell by cell.
  3. Re-send what's missing as **ordinary edits** (`setCell`) over a second, upload-only connection.
  4. Verify, logging each step to the console.
- **`applyMergeableChanges` is never re-broadcast** (it's how sync avoids loops), so it can't be used to push data.

### The Worker

The sync server is [tonyennis145/tinybase-cloudflare-worker](https://github.com/tonyennis145/tinybase-cloudflare-worker): TinyBase's WebSocket server running in a Cloudflare Durable Object, with each room saved in the Durable Object's SQLite storage. Its README covers deploying it (Cloudflare dashboard or `wrangler deploy`) and running it locally. The whole Worker:

```js
import { createMergeableStore } from 'tinybase';
import { createDurableObjectSqlStoragePersister } from 'tinybase/persisters/persister-durable-object-sql-storage';
import { WsServerDurableObject, getWsServerDurableObjectFetch } from 'tinybase/synchronizers/synchronizer-ws-server-durable-object';

export class TinyBaseDurableObject extends WsServerDurableObject {
  createPersister() { return createDurableObjectSqlStoragePersister(createMergeableStore(), this.ctx.storage.sql); }
}
export default { fetch: getWsServerDurableObjectFetch('TINYBASE') };
```

- **One Worker serves every app.** Each URL path is its own room (its own Durable Object), created on first connect. Give each app its own room name.
- **Use the same TinyBase version in the Worker and the app.**
- **The Worker has no authentication,** so a room is readable by anyone who knows its URL (see the store rules above).

## 6. server.mjs (only when needed)

- **Node 18+ and `node:` built-ins only:** `http`, `crypto`, `fs`, `async_hooks`. No packages.
- **Small helpers:**
  - `httpError(status, msg)`
  - `json(res, status, data)`
  - `readBody(req)`
  - `need(value, name)`
- **Per-request context via `AsyncLocalStorage`:** values the browser sends in headers (keys, credential choices), readable anywhere without passing them around.
- **All routes at the bottom** in one `handle(req, res)`:
  - A flat list of `if (method === … && pathname === …) return json(res, 200, await fn(...))`.
  - One try/catch that turns errors into `{ error }` JSON.
- **Serve only `index.html` and `md/*.md`,** with a strict filename check, `cache-control: no-store` and nothing else. Never `.env` or other files.
- **Return data ready to use;** business logic can live in the browser. The server exists for secrets, CORS and signing, not for app logic.

## 7. Credentials and API keys

- **A Config page** (reached from the `⋯` menu) has one card per external service: what it's for, what in the app uses it (`usedBy`), and where its key comes from.
- **Keys come from `.env` on the server,** or from **team keys** pasted on the Config page:
  - Encrypt pasted keys with a team code: AES-GCM, with a key derived via PBKDF2.
  - Keep them in a synced `secrets` table, decrypted only in the browser.
  - The browser sends them to the server per request in a header. The server falls back to `.env`, and never saves a pasted key to disk.
- **Every call to a keyed service goes through one function,** `keyedFetch(provider, url, opts)`:
  - Each provider declares its auth scheme once: `PROVIDER_AUTH[p] = { header: (key) => ({ 'x-api-key': key }) }`.
  - It returns a fetch-like `{ ok, status, json(), text(), via }`, where `via` says which key was used ("team key", ".env key").
  - Log every call with its source, e.g. `[apify] POST api.apify.com via .env key: HTTP 201`. When something fails, that line answers "which key was it using?".
- **Test a key with a free, read-only call** (an account-info endpoint) before building on it.
- With Giga connected, each card can use a Giga credential instead of a key. See **Optional: Giga integration** at the end.

## 8. AI features

- **Pattern:** research (deterministic code + APIs) → the AI generates → **the AI grades it against a rubric** → a person approves the batch → code publishes or sends → tracking scores the result.
- **Graders return JSON** (`response_format: json_object`): a score per factor, a rationale, and the matched ICP. The rubrics live in one object on the server.
- **Batch work** goes through a shared runner (concurrency ~3) and a job list showing queued / grading / done / failed / skipped. Skip duplicates before running.

## 9. Testing and ways of working

- **Syntax-check the whole front end** by pulling out the module script and running `node --check` on it.
- **Browser tests use headless Chrome** (`puppeteer-core` with the system Chrome):
  - **Replace `WebSocket` with a stub** (`evaluateOnNewDocument`) so tests never write to the real room.
  - For sync tests, redirect to a **throwaway room** (`/<room>-test-<timestamp>`).
  - Screenshot the result and look at it.
- **Test against real integrations with read-only or cheap calls,** and say what each one costs.
- **Time-box investigations.** After ~5 minutes or 2–3 disproved theories, stop and report what's known, what's ruled out and the options. Say how long any test over a minute will take.
- **Back up a file before a big rewrite,** and confirm before anything destructive or shared (deleting synced data, changing the shared room).
- **Search before naming a CSS class or route,** to avoid collisions.

## Optional: Giga integration

Skip this section if the app doesn't use [Giga](https://getgiga.com). With it, people sign in to Giga once and use credentials they already keep there, instead of pasting API keys into each app. Giga makes the call and adds the secret itself, so the app never sees it.

The **Config** page then reads, in order:
1. **Connect Giga** (OAuth).
2. **Team code**, only if pasted keys are used.
3. **One card per provider,** each with an **API key | Giga credential** switch.

### Sign-in (generic for any MCP server)

- **Routes:** `/oauth/:provider/start` and `/oauth/:provider/callback`, plus `/api/mcp/:provider/{status,tools,call,disconnect}`.
- **Discover everything from the MCP URL:**
  1. `/.well-known/oauth-protected-resource`
  2. then the authorization server's metadata
  3. register the app automatically (dynamic client registration)
  4. PKCE (S256) with a public client
  5. refresh the token automatically, and retry once on a 401
- **Tokens stay in `.oauth.json` on the server** (file mode 600, gitignored). The browser only gets a connection ID.
- **A synced `connections` table** holds non-secret details (account, status) so the team can see who's connected.
- **The MCP transport** is JSON-RPC over HTTP POST:
  - `initialize`, then `notifications/initialized`, `tools/list` and `tools/call`
  - keep the `mcp-session-id`
  - accept either JSON or a one-shot event stream

### Using credentials: one function

- **Each provider declares:**
  - **`slugs`:** which Giga `integration_slug`s count as this provider. Compare them ignoring punctuation (`scrape-creators` = `scrapecreators`).
  - **`usedBy`:** shown on its card.
  - **Its auth scheme** (from §7), plus the matching Giga injection rule: `PROVIDER_AUTH[p] = { header: (key) => ({…}), injection: 'header.x-api-key={token}' }`.
- **Choices are one synced value,** e.g. `{ apify: 'apify-magnet', deepseek: '' }`:
  - a handle means use that Giga credential;
  - `''` means an API key was chosen on purpose;
  - no entry means not chosen yet.
- **The browser sends the choices with every request** (e.g. an `x-<app>-giga` header).
- **`keyedFetch` (from §7) gains two routes,** and picks one of three:
  1. **Giga API credential** → `http_request_with_credential` (pass the provider's `injection_rule`).
  2. **Giga MCP connection** → `invoke_tool`, **only if** the provider declares an `mcp(url) → { tool_id, arguments }` mapping *and* its MCP tools mirror the REST API and return JSON. ScrapeCreators does; Zernio doesn't.
  3. **API key** (team key, or `.env`) → the server makes the call itself.
- **Show `via` in the UI** ("fetched via Giga · apify-magnet") as well as in the log.
- **Auto-select:**
  - Only for a provider with no API key anywhere and no choice yet.
  - Prefer plain credentials over Pipedream ones, then healthy ones, then the newest.
  - Never while the team keys are locked: a locked vault looks like "no key", and the choice syncs to everyone.
- **Which credentials work** (from `list_credentials`; it never returns secrets):
  - **Plain API key / OAuth credentials:** work with `http_request_with_credential`.
  - **Pipedream-connected credentials:** only work if Giga's Pipedream plan includes its Connect proxy. Otherwise you get a 403 ("proxy API is not available on your current plan").
  - **MCP credentials:** can't be used for plain requests (`credential_not_http_proxyable`). Use `invoke_tool`, or add the service to Giga again as an API key.
  - **Google:** each OAuth credential only has the scopes it was granted. Use Giga's native Search Console integration (`webmasters.readonly`). Keep Search Console and GA4 as **separate providers**, because they need different scopes.
- **Refresh tokens rotate, so refresh one at a time.** Each refresh returns a new refresh token and retires the old one, and re-using a retired one makes Giga revoke the sign-in. Requests often hit an expired token together, so keep one in-flight refresh per connection and let the others wait for it. If a refresh is rejected, mark the connection expired and show **Sign in again**.
- **Giga's MCP endpoint currently rejects requests from web pages (no CORS),** so using Giga needs `server.mjs` as a thin proxy. If Giga allows the app's web address, the browser can sign in with PKCE and call Giga directly, and the app needs **no server at all**.

## Open questions for review

1. **Protecting rooms:** anyone with a room URL can read it. Should rooms need a secret token?
2. **Room names:** how should new apps choose one (and should it be hard to guess)?
3. **Hosting:** should there be a default static host (e.g. Cloudflare Pages)?
4. **Giga credential choices:** per person, or shared by the team (as here)?
5. **Which rules are too prescriptive** (UI patterns, "never", "always") and should be softened or removed?
