---
name: "telegram-serverless"
description: "Build, port, debug or deploy Telegram bots on Telegram Serverless (tgcloud): SDK db/api/fetch, schema and migrations, handlers, endpoints, Mini Apps, tgcloud CLI, measured sticker/custom-emoji limits."
---

# Telegram Serverless (tgcloud)

Use for any work on a bot that runs (or should run) on Telegram Serverless:
writing/porting code, reviewing it, deploying with the `tgcloud` CLI, debugging.
Reference version: `@tgcloud/cli` 0.2.0 (project layout with `tgcloud/` folder).
Official docs: https://core.telegram.org/bots/serverless. Inside a scaffolded
project the authoritative copies are `AGENTS.md` and `docs/tgcloud-sdk.md` —
if the user's project has newer versions, prefer them over this file.

Reply to the user in their language (usually Russian).

The platform is new and changes quickly. Treat this file as the baseline; when an answer
depends on a detail that may have changed, say so and check the docs' "Recent changes".
**Never invent APIs, limits, prices or features** — say "not documented" (or "measured"
with the date, for §11–§13). Code runs **only on demand**: when an update of a handled
type arrives or the Mini App calls an endpoint.

## 1. What the platform is

- Bot backend runs as **plain JavaScript ES modules** in a **V8 isolate** on
  Telegram's infrastructure, next to the Bot API and a per-bot **SQLite** DB.
- **JavaScript only.** No Python, no aiogram, no Pillow.
- **No npm packages, no filesystem, no `node_modules`, no build step** at runtime.
  Only `sdk` (+ `sdk/db`, `sdk/api`, `sdk/fetch`) and the project's own modules.
  A pure-JS library can only be used by vendoring its source into `tgcloud/lib/`.
- **No network** except the SDK's `fetch`.
- Webhook is **managed by the platform**: subscribed update types = existing,
  non-empty files in `handlers/`. Never call `setWebhook` yourself.
- Serverless must be enabled per bot in @BotFather (bot → Serverless). CLI access
  token: BotFather → bot → Serverless → CLI Access → Access token (it is NOT the
  bot token). When enabled, any old polling bot for the same token must be stopped.
- **Not documented (assume unknown, design defensively):** per-invocation
  wall-time/CPU/memory limits, cron/scheduled jobs, queues, background tasks,
  `waitUntil`, retries of failed updates, whether module-level globals survive
  between invocations. Only the DB is guaranteed persistent. Observed limits: §12.
- Documented limits: upload 50 MB/file (10 MB photo); `getFile` download 20 MB;
  `fetch` response total 30 MB; batch insert bounded by SQLite variable limit.
- Production `console.*` output is **not viewable** (only in `npx tgcloud run` and
  BotFather test runs) — keep your own log table (§12).
- Docker/VPS rules for the runtime do not apply (no container, logs, RAM to
  manage) — but the CLI tooling on the user's VPS still runs in Docker (§9).

## 2. Project layout

```text
my-bot/
├── tgcloud/                 ← only .js files here are deployed
│   ├── schema.js            DB tables as NAMED exports, one file
│   ├── handlers/            one level only; file name = Bot API update type
│   │   ├── message.js
│   │   ├── callback_query.js
│   │   └── inline_query.js / chat_member.js / chat_join_request.js / …
│   ├── endpoints/           one level only; Mini App backend, POST /api/<name>
│   └── lib/                 shared modules, subfolders allowed
├── tgcloud.jsonc            project config ("static" block for Mini App hosting)
├── docs/tgcloud-sdk.md      SDK reference (local only)
├── AGENTS.md                rules for AI assistants (local only)
├── package.json             devDependency @tgcloud/cli
└── .tgcloud/                CLI machine state + credentials — NEVER read/edit; gitignored
```

Module name = path inside `tgcloud/` without `.js` (`handlers/message`, `lib/cart`).
A full `push` mirrors the folder, **including deletions**; a targeted `push <files>` never
deletes others. A `.js` file in `tgcloud/` outside these locations is flagged and not deployed.
Endpoint names: letters, digits, `_`, not starting with a digit.

## 3. Module rules that bite

- Project imports: **relative path WITH `.js`** — `import { users } from '../schema.js'`.
  `'../schema'` fails. Nothing outside `tgcloud/` is reachable.
- SDK imports by name:
  ```js
  import { db, api, fetch, InputFile, FormData, BotApiError, EndpointError } from 'sdk'
  import { table, integer, text, json, eq, and, sql } from 'sdk/db'
  ```
- **Handler**: `export default async function (payload, ctx)`. `payload` is the
  unwrapped update part: `handlers/message` gets the `Message`,
  `handlers/callback_query` the `CallbackQuery`, etc. Full `Update` = `ctx.update`.
  Handlers are opt-in: a missing/empty file means that update type is not received.
- **Endpoint**: `export default async function (input, ctx)`; `input` = JSON body
  (object); `ctx.initData` = verified Mini App init data (`ctx.initData.user` is the
  caller; there is no `ctx.user`). Return value → `{ ok: true, result }`.
  Refuse with `throw new EndpointError('text', { code: 'X' })` → 400; any other
  exception → generic 500 (message only in logs). Invalid init data → 401, code never runs.
  On the client a generic failure arrives as `{ type: <not ENDPOINT_ERROR>, message:
  'Unexpected response from endpoint <name> (HTTP 500)' }` — wrap endpoints (§12).
- Secrets/config: no env-var mechanism is documented. Bot token is handled by the
  platform (`api` is pre-authenticated). Get the bot username via `api.getMe()`.
  Non-secret config → `lib/config.js`; runtime-editable config → a DB table.
- `atob`/`btoa`/`Buffer`/`CompressionStream` are not documented in the runtime: feature-
  detect (`typeof globalThis.btoa === 'function'`) and keep a pure-JS fallback.

## 4. Database (`db`)

Every call is **async — always `await`** (a missing `await` returns the builder).

### Schema (`tgcloud/schema.js`)

```js
import { table, integer, text, real, blob, boolean, json, index, uniqueIndex, unique, check, primaryKey, sql } from 'sdk/db'

export const users = table('users', {
  id:      integer('id').primaryKey({ autoIncrement: true }),
  tgId:    integer('tg_id').unique(),
  name:    text('name').notNull(),
  lang:    text('lang').default('en'),
  isAdmin: boolean('is_admin').default(false),
  prefs:   json('prefs'),
  created: integer('created_at', { mode: 'timestamp' }).default(sql`(unixepoch())`),
}, (t) => ({
  createdIdx: index('idx_users_created').on(t.created),
}))
```

- Types: `text`, `integer`, `real`/`float`, `numeric`, `blob` (Uint8Array),
  `boolean` (0/1 ↔ true/false), `json` (auto stringify/parse).
- `opts.mode`: `boolean`, `json`, `timestamp` (unix s ↔ Date), `timestamp_ms`, `bytes`.
- Column name arg is optional (defaults to JS key).
- Modifiers: `.primaryKey({ autoIncrement })`, `.notNull()`, `.unique()`, `.default(v | sql\`…\`)`,
  `.generatedAlwaysAs(sql\`…\`, { mode: 'virtual'|'stored' })`, `.constraint('COLLATE NOCASE')`,
  `.deprecated('reason')` (terminal).
- Table callback: `primaryKey({ columns: [t.a, t.b] })`, `unique('n').on(...)`,
  `check('n', sql\`…\`)`, `index('n').on(...)`, `uniqueIndex`, expression and partial
  (`.where(sql\`done = 0\`)`) indexes.
- Table modifiers: `.strict()`, `.withoutRowid()`, `.constraint('CHECK (x > 0)', 'chk_x')`, `.deprecated()`.
- **No foreign keys**: `.references()` / `foreignKey()` THROW at deploy. Enforce in code
  (insert parents first, delete children first, sweep orphans with
  `LEFT JOIN … WHERE parent.id IS NULL`).
- **Drops only via `.deprecated()`**; deleting a declaration drops nothing.
  Type changes are manual via `db.run(...)`. Params don't work in DDL (DEFAULT/CHECK) — use literal SQL.

### Queries

```js
await db.select().from(todos)                                  // awaitable = .all()
await db.select().from(todos).where(eq(todos.id, 1)).get()     // first row | null
await db.$count(todos /*, where? */)
await db.select({ id: todos.id, n: sql`count(*)` }).from(todos)
  .where(and(eq(todos.userId, uid), eq(todos.done, false)))
  .groupBy(todos.userId).having(sql`count(*) > ${1}`)
  .orderBy(desc(todos.priority), asc(todos.id)).limit(10).offset(20).all()

await db.insert(todos).values({ userId: 1, text: 'A' }).run()   // → { rowsAffected, lastInsertRowid, rows: [] }
await db.insert(todos).values([{ text: 'A' }, { text: 'B' }]).run()
await db.insert(users).values(v).onConflictDoNothing({ target: users.tgId }).run()
await db.insert(users).values(v).onConflictDoUpdate({ target: users.tgId, set: { name: 'Ann' } }).run()
await db.update(todos).set({ done: true }).where(eq(todos.id, 1)).run()   // .set() required
await db.delete(todos).where(eq(todos.id, 1)).run()

// .returning() / .returning({ id: todos.id }) → result IS the row array (converted)
const rows = await db.update(jobs).set({ lock: tok }).where(cond).returning()
```

- Builders are awaitable; `.run()` is optional but explicit is clearer.
- Terminals: `.all()`, `.get()`, `.values()`, `.run()`. `.where(a, b)` ≡ `and(a, b)`.
- Increment: `set: { seen: sql\`${t.seen} + 1\` }`.
- Operators (`sdk/db`): `eq ne gt gte lt lte like notLike isNull isNotNull and or not
  between notBetween inArray notInArray count sum avg min max asc desc`.
- `sql` tag: values → named params, column refs → identifiers, nested `sql` spliced;
  `sql.raw('...')` for literals.
- Raw: `db.run / db.all / db.get / db.values` take `sql\`…\`` or `(query, { ':id': 5 })`.
  Raw results are NOT mode-converted (json is a string, booleans 0/1, timestamps seconds).
  Use `db.values` for joins with duplicate column names.

### Migrations

`push` deploys code and NEVER changes the database; `npx tgcloud migrate` applies schema
changes after confirmation. Statuses: **safe** (applied together), **warning** (drops/slow
ops, confirmed one by one), **manual** (e.g. a column type change — new column, copy, swap),
**undocumented** (in the DB, not in the schema). Remove a table/column: `.deprecated('why')`
→ `migrate` → then delete the declaration. Always run migrate after changing `schema.js`. Forgetting it shows up at runtime as
`Error: Failed to prepare query [1] no such table: <name>` — catch `/no such table/` in
admin commands and answer "run migrate".

## 5. Bot API (`api`)

```js
const me = await api.getMe()                       // unwrapped result
await api.sendMessage({ chat_id, text, parse_mode: 'HTML', reply_markup })
await api.answerCallbackQuery({ callback_query_id: cq.id, text: 'Done' })
```

- Proxy: every Bot API method (incl. future ones) works; params are ONE object in
  snake_case; result is unwrapped (no `.ok/.result`).
- Failure throws `BotApiError` with `.code` (400/403/429/5xx…), `.description`
  (e.g. `"Bad Request: STICKERSET_INVALID"`), `.method`, `.parameters`
  (`retry_after`, `migrate_to_chat_id`). Inspect `.code`/`.description`; rethrow the rest.
  **5xx happen** (seen: `getStickerSet: 502 Bad Gateway`) — treat `code >= 500` as transient,
  and as "outcome unknown" for writes (reconcile before retrying, §10a).
- **Files up**: `new InputFile(bytes, 'name.ext', { type })` anywhere in params, at any
  depth (also inside arrays like `sendMediaGroup.media`, `InputSticker.sticker`).
  Many uploads in ONE invocation fail with a non-BotApiError `Upload: too many files` (§12).
- **Files down**: by `file_id` only — `await api.getFileContent(file_id)` → Uint8Array;
  `await api.getFileStream(file_id)` → `{ file, body, bytes() }`. 20 MB cap; bad id → BotApiError 400.
- Prefer re-sending an existing `file_id` over download+upload (e.g. `InputSticker.sticker`
  accepts a `file_id` string).

## 6. HTTP (`fetch`) and logging

- `const res = await fetch(url, { method, headers, body })`; `res.ok/status/statusText/url/headers`,
  `await res.json()` / `res.text()` / `for await (const c of res.body)`. Body readable once.
- Body helpers: `fetch.body.json(o)`, `fetch.body.form(o)`, `fetch.body.text(s)`; binary
  bodies: `Uint8Array`, `InputFile`, `FormData` (`import { FormData } from 'sdk'`).
- Redirects followed; HTTP errors resolve with `ok === false`; only network errors reject.
- `console.log/info/warn/error` — shown in `npx tgcloud run` output, tagged `[file:line]`;
  NOT visible for production traffic → log to a DB table as well (§12).

## 7. Mini App hosting

- `tgcloud.jsonc`: `{ "static": { "source": "dist", "spa": true, "immutable": ["/assets/*"] } }`
  (`"static": "dist"` shorthand; `"static": false` removes the site). Served at
  `https://app<app_id>.tgcloud.ai/`, endpoints under `/api/`. A plain folder of
  hand-written files (no build) works: `"static": { "source": "web", "spa": false }`.
- `push` never builds — build first (`npm run deploy` = build + push). `static` also takes
  `headers` and `redirects`; never put files under `/api/`.
- Front-end is normal browser code with npm; it reaches the backend only via
  `Telegram.WebApp.Serverless.call('<name>', input, (err, result) => …)` from the official
  `telegram-web-app.js` (never hand-roll fetch + initData). Don't set
  `X-Frame-Options`/restrictive `frame-ancestors` (Telegram Web uses an iframe).
- `err.type === 'ENDPOINT_ERROR'` → your `EndpointError` (`err.parameters` = your object);
  anything else is the platform (e.g. `UNEXPECTED_HTTP`, HTTP 500) → retry a few times with
  backoff, but only for idempotent endpoints.
- Keep endpoint responses small: send big binaries in pieces (≤128 KB raw per call,
  base64) — a ~700 KB base64 response coincided with bare HTTP 500s; chunking fixed it.
- Heavy client-side work (image/video processing) belongs in the Mini App, since the
  backend has no native libraries. WebView specifics: §13.
- After `push`, users must close and reopen the Mini App (WebView caches the page).

## 8. CLI (`npx tgcloud …`)

| Command | Purpose |
|---|---|
| `npm create @tgcloud/bot <dir>` | scaffold + install CLI, optionally link the bot |
| `init` | scaffold in an existing folder (detects a front-end) |
| `login` | link project to a bot (interactive, hidden token prompt) |
| `add handlers/<type>` / `add endpoints/<name>` / `add lib/<x>` | scaffold a module (`add handlers` lists valid names) |
| `status` / `diff [file]` | local vs snapshot (offline) |
| `push [files…] [--force]` | deploy changed modules (+ static build) atomically; rejected if someone deployed since last sync |
| `fetch` / `pull [--force]` / `reset [file]` | sync snapshot / working dir with cloud; discard local changes |
| `migrate [--yes] [--safe] [--local] [--dry-run]` | apply schema changes |
| `run <module> '<json5 payload>' [--ctx '<json5>']` | execute a handler/endpoint in the cloud WITHOUT deploying, shows console output |
| `webhook` / `webhook sync [--drop-pending]` | show / repair the managed webhook (`allowed_updates` from handlers) |
| `upgrade [--dry-run]` | migrate a pre-0.2.0 layout into `tgcloud/` |
| `completion [shell]` | shell completion |

Node.js 18+. `npm create @tgcloud/bot` never overwrites files. `run` executes LOCAL code on
the platform without deploying; the payload is JSON5 and must be an object (pass saved real
payloads with `"$(cat msg.json5)"`). Global flags: `--debug` (log HTTP), `--no-pager`.
Auth: env `TGCLOUD_TOKEN` (CI; never written to disk; takes priority) or the token saved by
`login` in `.tgcloud/` (format `app<id>:<secret>`). There is **no logs command**.
Optimistic concurrency: if the cloud revision moved ahead, `push` is rejected → `fetch`/`pull`;
`push --force` only deliberately.

Example test run:
```bash
npx tgcloud run handlers/message '{ message_id: 1, from: { id: 123, is_bot: false, first_name: "me" }, chat: { id: 123, type: "private" }, text: "/start" }'
npx tgcloud run endpoints/getProfile '{}' --ctx '{ initData: { user: { id: 1 } } }'
```

Standard deploy: `push` → `migrate` (if schema changed) → `webhook` (check subscriptions).

Known CLI 0.2.0 bug: `push` can crash AFTER a successful deploy with
`TypeError [ERR_INVALID_ARG_TYPE] … Received type boolean (false)` in
`writeSnapshotFile` (server returned `false` for a module). The cloud is fine; repair the
local snapshot with `npx tgcloud login` (re-sync) or `npx tgcloud pull --force`.

## 9. The user's setup: CLI on a Debian VPS via Docker

The user runs the CLI on their VPS without installing Node on the host, via a
throwaway container. Use this wrapper in instructions (`tgc npx tgcloud push`):

```bash
# token once, hidden, file mode 600
install -m 600 /dev/null ~/.tgcloud.env
read -rsp 'Access token: ' T && printf 'TGCLOUD_TOKEN=%s\n' "$T" > ~/.tgcloud.env && unset T; echo

# in ~/.bashrc
tgc() {
  docker run --rm -it \
    --user "$(id -u):$(id -g)" -e HOME=/tmp \
    --env-file ~/.tgcloud.env \
    -e NPM_CONFIG_UPDATE_NOTIFIER=false -e NPM_CONFIG_FUND=false \
    --memory 512m --log-driver none \
    -v "$PWD":/work -w /work \
    node:22-alpine "$@"
}
```

- Run commands from the project folder (it is the only mount). Don't suggest
  `npm install -g npm@…` — the container is ephemeral.
- Never print, log or commit the CLI token; never put `.tgcloud/` or `~/.tgcloud.env`
  into archives sent anywhere.
- Moving files into the project: `scp` to the VPS, unzip inside the container
  (`tgc unzip -o -q x.zip`).
- **Ship updates as a FLAT zip of the whole project** (no root folder, without
  `node_modules/` and `.tgcloud/`) plus a `VERSION` file the user can `cat` to confirm.
  A zip with a root folder unpacks into a nested folder, `push` then sees no changes and
  old code keeps running — this happened repeatedly. Partial zips cause mixed versions.

## 10. Assessing and migrating an existing bot (do this first)

Give a verdict: **fits / fits with changes / does not fit**, with reasons. Blockers:
1. **Language** — non-JS bots (aiogram, python-telegram-bot, Go…) need a full JS rewrite.
2. **Dependencies** — npm libs (telegraf, grammY, zod, openai SDK, sharp…) → own code +
   `fetch` to REST APIs. Native processing (images, ffmpeg) can't run on the backend — move
   it to the Mini App (canvas, WebCodecs, §13) or an external service.
3. **Background work** — timers, polling external APIs, scheduled broadcasts, cron, queues:
   no scheduler; only button-driven batches (below) or a hybrid with an external worker.
4. **Secrets** — third-party API keys have no documented storage; never hardcode them
   (modules are in git and deployed). Flag as an open question.
5. **Data** — Postgres/Redis-specific features, foreign keys, files > 20/50 MB, HTTP
   responses > 30 MB.

| Before | On Telegram Serverless |
|---|---|
| polling / own webhook server | platform-managed webhook (`npx tgcloud webhook`) |
| update-type router | one file per type in `handlers/` |
| command / callback_data routing | own dispatcher in `lib/` |
| FSM in memory / Redis | `states` table (user_id PK, state, data json, updated_at) |
| SQL DB / ORM | `schema.js` + `db`, no FKs |
| `BOT_TOKEN` env | not needed; `api` is pre-authorised |
| requests / axios | `fetch` from `sdk` |
| Mini App backend + initData check | `endpoints/` (verification built in) |
| front-end hosting | `tgcloud.jsonc` → `static` |

Steps: (1) a separate **staging bot** (one project = one bot) with Serverless on;
(2) port the schema → `push` → `migrate`; (3) rewrite handlers, test each with
`npx tgcloud run` and saved real payloads; (4) move data — no import tool: a temporary
endpoint called chunk by chunk via `run`, verified on staging, removed afterwards;
(5) cut over: stop the old service (it steals updates via `getUpdates`/`setWebhook`), enable
Serverless on prod, `push`, `migrate`, `npx tgcloud webhook` shows **in sync**; (6) verify
commands, buttons, inline mode, 429 handling, and idempotency — updates may be redelivered,
use upserts / `update_id`; (7) keep the old deployment ready for rollback until stable.

## 10a. Design patterns for this runtime

**Long work (> a few seconds)** — no background jobs or cron are documented:
1. Store the job in a DB table (one row per user/job: progress, lock token, lockedUntil).
2. Each invocation works within a time budget (≈18–20 s; ~22 s ran fine), saving
   progress after every unit.
3. Then edit a progress message with an inline **▶️ Continue** button; each press
   (`callback_query`) runs the next batch.
4. **Lock** with one atomic statement:
   ```js
   const rows = await db.update(jobs).set({ lockToken: tok, lockedUntil: now + LOCK_MS })
     .where(sql`${jobs.userId} = ${uid} AND (${jobs.lockedUntil} IS NULL OR ${jobs.lockedUntil} < ${now})`)
     .returning()
   const job = rows[0] || null   // null → someone else is running it
   ```
   Lock expiry covers a killed isolate. Release only if the token still matches.
5. **Idempotent resume**: reconcile with the real external state before continuing
   (e.g. count stickers already in the target set) — an invocation can die after the
   side effect but before the progress write, and an API call can succeed while its
   response is lost. Before a write, store the item ids as `inflight`; on the next run
   compare with reality. Before retrying a failed call, check whether it already took effect.
6. Handle 429 via `e.parameters.retry_after`: stop the batch, tell the user how long to wait.
7. Answer `answerCallbackQuery` immediately, before the long work.
8. Cap file uploads per invocation (≤6) — see `Upload: too many files` (§12).

**State** — keep everything in the DB: dialog steps (FSM) in a `sessions` table with
TTL, throttling/slots/queues as rows. Never rely on module globals beyond caching
harmless values (like `getMe().username`).

**Porting from Python/aiogram**: full rewrite in JS. Map: routers/filters → `if`s in
`handlers/message.js`; FSM → sessions table; `FSInputFile` local images → upload once
and keep `file_id`s (no filesystem); in-memory lists → tables; `asyncio.sleep`
loops/broadcasts → batched jobs with Continue; Pillow/native libs → Mini App canvas,
vendored pure JS (watch CPU), or an external service via `fetch`.

**Groups**: commands arrive as `/cmd@BotName` — ignore commands addressed to other
bots; carry the owner id in `callback_data` (`cont:<userId>`) and reject presses from
others. Some user-scoped methods (e.g. `createNewStickerSet`) fail with
`PEER_ID_INVALID` until the user has started the bot in private — show a hint.

## 11. Stickers and custom emoji — measured (2026-10-28)

Measured on the live Bot API with a `/probe` admin command (method below). Prefer these
over the docs where they differ; re-measure if Telegram changes behaviour.

| Custom emoji upload (`InputFile`) | Accepted | Refused (`Bad Request: file is too big`) | Docs say |
|---|---|---|---|
| static PNG 100×100 | 131 072 B | 132 096 B → **limit 128 KiB** | not stated |
| video WEBM (VP9) 100×100 | 65 536 B | 66 560 B → **limit 64 KiB** | 256 KB (wrong for emoji) |
| animated `.tgs` | 65 536 B | 66 560 B → **limit 64 KiB** | 64 KB |

`.tgs` complexity (measured 2026-10-29; refusal text is **`Bad Request: wrong file type`**,
not "too big"):

| `.tgs` content | Accepted | Refused | Safe rule |
|---|---|---|---|
| points in ONE contour (`sh` vertices) | 504 | 512 | **≤ 500** |
| contours of 4 points each | 2336 (≈9.3k points) | 2368 | **≤ 2300 contours and ≤ 9000 points total** |
| unpacked Lottie JSON size | 2 MB (search top) | — | not limited (the 64 KiB is the gzip file) |

The contour test can't tell "contours" from "total points" apart — respect both bounds.
A traced photo (1217 contours, a 933-point contour) was refused even though rlottie and
`tgs_check` accept it: Telegram's server check is stricter than the renderer.

- Sticker → emoji by `file_id`: animated `.tgs` (512×512 canvas) is accepted as-is;
  static and video stickers (512 px) are refused with `STICKER_PNG_DIMENSIONS` /
  `STICKER_VIDEO_DIMENSIONS` — they must be re-encoded to exactly 100×100. Don't try
  them by `file_id` at all (wasted calls, noisy errors) — send them straight to conversion.
- One custom emoji set can mix static, video and animated stickers (per-sticker `format`).
- `deleteStickerSet` / `deleteStickerFromSet` work for sets the bot created.
- Video emoji: VP9, ≤30 fps, no audio; keep duration ≤ 2.9 s as a margin to the 3 s rule
  (exact boundary not measured). Target ≤ 63 KiB and re-encode smaller if Telegram still
  says `file is too big`.
- Readability: a 100×100 static emoji makes small text unreadable; a vector `.tgs`
  (traced artwork, 512×512) stays sharp at any size. Photos trace badly — keep them PNG.
- Building `.tgs` by tracing: after each detail tier count contours/points; if over the
  limits go to a simpler tier (fewer colours, coarser curves); a single contour > 500 points
  → keep evenly spaced vertices as straight segments; if nothing fits, send a PNG.
- Emoji set limits: 200 emoji per set; `createNewStickerSet` takes 1–50 initial stickers;
  set name `[a-z][a-z0-9_]*_by_<bot>`, ≤ 64 chars, no `__`; title ≤ 64 chars.
- Using custom emoji in messages needs Telegram Premium (creating sets does not).

**How to measure a limit (`/probe` pattern):** admin-only command; create a scratch set
with a tiny anchor sticker; for each format binary-search the size (start 64 KiB, double
until refused, then bisect to 1 KiB) by `addStickerToSet` with generated files of an
exact byte size, deleting each accepted probe; on `file is too big` → upper bound; any
other error → stop that format and report the text; ≤6 uploads per invocation, ▶️
Continue between steps; delete the scratch set and write the result to the log table.
Exact-size valid files without codecs: PNG — stored (uncompressed) deflate + a private
ancillary chunk (`prVt`) of random bytes; WEBM — a small prebuilt VP9 clip with an
unknown-size Segment + a Matroska `Void` element (ID `0xEC`, 8-byte size) appended;
`.tgs` — Lottie JSON with a random `nm`, gzip with stored blocks. Random padding keeps
every file unique (identical stickers are silently ignored).
Complexity probes (count units, 2 % precision): `.tgs` with N 4-point squares, and with one
contour of N points; keep the file small with a tiny greedy LZ77 + fixed-Huffman deflate in
pure JS (1217 contours → 3 KB). Unpacked-size probe: one long run of `a` in `nm` encoded as
length-258/distance-1 copies (2 MB JSON → 14 KB file). Treat ANY 400 at a size above an
accepted one as the bound and report its text — Telegram doesn't always say "too big".
Keep the files Telegram refuses (`rejects` table, `/rejects`, `/reject <id>` sends it back
as a document) so they can be analysed offline; this is how the complexity limit was found.

## 12. Platform behaviour observed in production

- `Upload: too many files` — thrown (not a `BotApiError`) after roughly 10–15 `InputFile`
  uploads in one invocation; the next invocation works. Keep ≤6 uploads per invocation.
- Invocations of ~22 s wall time completed; budget long work at 18–20 s.
- Bot API 5xx (`502 Bad Gateway`) occur occasionally — retry later, reconcile writes.
- **Own log table + `/logs`** (production `console.*` is invisible): table `logs(id, ts,
  level, src, userId, msg, detail)`, pruned on every write (e.g. 3 days and ≤1000 rows);
  wrap every handler/endpoint (`guardHandler` / `guardEndpoint`) to catch, log and return
  "Внутренняя ошибка #<id>" (endpoints → `EndpointError` with code `INTERNAL`); an
  admin-only `/logs [N]` and `/logs #id`; a Mini App `report` endpoint so errors only the
  browser saw (platform 500s, decoder failures, Telegram rejections) reach the journal.
  Log Telegram rejections server-side too — a silent fallback hides the real cause
  (this is how the 64 KiB video limit was found).
- `ADMIN_IDS` in `lib/config.js`; when a non-admin calls an admin command, reply with
  their id so a misconfigured id is obvious.

## 13. Mini App in Telegram's WebViews (video/image work)

- A `<video>` that is not on screen may not play or advance frames in Telegram's
  WebViews (headless Chromium hides this): don't capture frames by playback
  (`requestVideoFrameCallback`) or trust the WEBM header duration (often wrong in
  sticker files). Demux the WEBM in JS and decode with WebCodecs `VideoDecoder`;
  seeking a `<video>` only as a fallback.
- **Close every `VideoFrame` right in the decoder `output` callback.** Hardware decoders
  (Android, Windows) hand out frames from a small pool; keeping frames open stalls
  `flush()` forever. Configure `hardwareAcceleration: 'prefer-software'` when supported,
  put timeouts on decode/encode and a hard cap per item with a fallback path.
- VP9 alpha in WEBM is a second VP9 stream in `BlockAdditions > BlockMore >
  BlockAdditional` (BlockAddID 1); decode it with a second decoder and use its Y plane
  (`frame.copyTo`) as alpha. WebCodecs can't encode VP9 alpha — encode the alpha mask as
  a second VP9 stream and mux it the same way (webm-muxer needs a small patch:
  BlockAdditions + ReferenceBlock for delta frames).
- `VideoEncoder` with VP9 exists in Chromium-based clients (Android, Desktop on
  Windows/Linux, Telegram Web) but not on iOS/macOS — feature-detect
  (`VideoEncoder.isConfigSupported`) and tell users where video conversion works.
- Test WebView-specific failure modes locally with shims: block `HTMLMediaElement.play`,
  wrap `VideoDecoder` with a 4-frame output pool, and a decoder that never outputs.
- Image downscaling: browser canvas scaling is soft; a JS Lanczos3 in linear light with
  premultiplied alpha + trimming transparent margins gives visibly sharper 100×100 emoji.

## 14. Testing locally

There is no local runtime. Two options:
- `npx tgcloud run …` against the cloud (real DB and Bot API — side effects are real).
- A local mock for logic tests: a Node ESM loader (`module.register` resolve hook)
  that maps `sdk` → a mock exporting `api` (Proxy over a fake in-memory Bot API),
  `InputFile`, `BotApiError`, `EndpointError`, `db`; and `sdk/db` → a tiny Drizzle-like
  builder (in-memory or `node:sqlite`) implementing only the chains the bot uses, and
  throwing `no such table` for missing tables. Use a fake clock (`Date.now` advanced per
  API call) to exercise time budgets, and inject failures (429, 5xx, lost responses, bad
  file_ids, concurrent presses, `file is too big` above a configured size). Silence the
  bot's own warn/error output per test and print it only for failing tests.
- Mini App: run `web/` in Playwright Chromium with a mocked `Telegram.WebApp.Serverless.call`
  and the WebView shims from §13; validate outputs with real tools (`ffprobe`/ffmpeg for
  WEBM incl. alpha, `tgs_check.py` from python-lottie for `.tgs`, Pillow for PNG). For
  `.tgs`, also load/render with rlottie (`pip install rlottie-python`) — the renderer Telegram
  uses — but it is NOT the server check (see §11 complexity); very complex files render slowly.

## 15. Coding conventions and answer format

- Thin handlers → logic in `lib/` (`router.js`, `state.js`, `repo/*.js`, integrations).
- Upsert + `.returning()` instead of select-then-insert; always `await` db and api calls.
- Log sparingly to the log table (§12); no full update dumps. Keep `AGENTS.md` current.
- Answers: (1) **verdict / short answer** first; (2) **plan** with exact CLI commands (§9
  wrapper); (3) **complete files** for key modules when building, not fragments;
  (4) **pitfalls and open questions** stated explicitly (undocumented features, limits,
  secrets, scheduling).

## 16. Review checklist

- [ ] Only `sdk`, `sdk/db` and relative `.js` imports; nothing outside `tgcloud/`.
- [ ] Every DB call awaited; no `.references()`/`foreignKey()`.
- [ ] Schema change → `migrate` in the deploy steps; drops via `.deprecated()`.
- [ ] No long loops/sleeps in one invocation; progress persisted; resume idempotent; lock in place.
- [ ] ≤6 file uploads per invocation; 5xx treated as transient.
- [ ] `BotApiError` handled by `.code`/`.description`; 429 respected.
- [ ] `answerCallbackQuery` called promptly.
- [ ] HTML/Markdown user content escaped.
- [ ] Handlers/endpoints wrapped; errors and Telegram rejections land in the log table.
- [ ] Custom emoji within §11 limits (PNG 128 KiB, WEBM/TGS 64 KiB; `.tgs` ≤ 500 points per
      contour, ≤ 2300 contours, ≤ 9000 points).
- [ ] No secrets in code; CLI token never exposed.
- [ ] Handler files exist only for update types actually needed (they define the webhook subscriptions).
- [ ] Update shipped as a flat full-project zip with `VERSION`.
