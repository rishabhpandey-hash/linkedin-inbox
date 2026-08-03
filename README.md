# LinkedIn Client Inbox

A client-facing inbox and analytics dashboard for LinkedIn outreach campaigns.
Clients sign in with their work email, read the conversations
their campaigns produced, reply from the dashboard, and see how the campaigns are performing.

**Live:** https://rishabhpandey-hash.github.io/linkedin-inbox/

This repo is only the front end: a single self-contained `index.html`. All data access lives in a
Supabase edge function, so the page ships no secrets beyond the public anon key.

---

## How it fits together

```
Outreach platform  ──sync──▶  Supabase (Postgres + edge function "inbox")  ──JSON──▶  this page
   ▲                            │
   └────── reply / webhook ──────┘
```

- **Sync** — the edge function pulls conversations for every mapped campaign hourly via
  `pg_cron`, and within seconds of a reply webhook from the outreach platform.
- **Reply** — a reply is sent from the LinkedIn profile that owns that conversation, so the
  lead sees a message from the person they were already talking to.
- **Analytics** — conversation-level metrics come from the Postgres function `inbox_analytics`;
  invitation/connection/message volumes come from the outreach platform (cached 10 minutes).

Access control: `client_users` is the whitelist (email → client, `can_reply`, `is_admin`, optional
per-campaign scope). Everything is filtered to the signed-in user's client in the function; the
tables are RLS-on with no policies, so only the service role can read them.

---

## Using it as an AI / automation tool

**Start with the self-describing contract:**

```bash
curl -s -H "Authorization: Bearer $JWT" \
  https://qypevxpscdhdrzlelolt.supabase.co/functions/v1/inbox/api/meta
```

It returns every endpoint, its parameters, and notes on what the fields mean.

### REST endpoints

| Method | Path | Notes |
|---|---|---|
| GET | `/api/meta` | Self-describing contract. Start here. |
| GET | `/api/inbox` | `campaign_id`, `filter`, `sort`, `q`, `sender`. Returns rows (max 500) plus exact `counts` and `senders` facets. |
| GET | `/api/thread?id=` | Full message history, oldest first. |
| POST | `/api/reply` | `{conversation_id, body}` — **sends a real LinkedIn message.** Needs `can_reply`. |
| GET | `/api/analytics` | `days=7\|30\|90\|all`, optional `campaign_id`. |
| GET | `/api/export` | `format=csv\|json`, `filter=` — up to 1000 conversations. |
| GET/POST | `/api/admin/*` | Clients, logos, campaign mapping, users. Admins only. |

`filter` is one of `all`, `needs_reply`, `waiting_48`, `engaged`, `answered`, `first_reply`.
`sort` is one of `recent`, `oldest_waiting`, `most_replies`.

Auth is a Supabase JWT in `Authorization: Bearer …`; the account's email must be in `client_users`.

### MCP server (for a client's own AI tooling)

A remote MCP server exposes the same client-scoped data to a client's AI tool, without
handing over any vendor API key:

```
URL:    https://qypevxpscdhdrzlelolt.supabase.co/functions/v1/mcp
Header: Authorization: Bearer <per-client token>
```

Mint and revoke tokens in the dashboard's Admin panel → **Client AI access (MCP tokens)**.
The token is displayed once; only its SHA-256 hash is stored. Tokens are scoped to one
client (optionally a subset of campaigns), are **read-only unless reply is ticked**, can
carry an expiry, and are revocable instantly. Every call is rate limited (60/min) and
logged with its arguments and duration.

Tools: `linkedin_get_account_info`, `linkedin_get_analytics`, `linkedin_list_conversations`,
`linkedin_get_conversation`, and — only for tokens with reply enabled —
`linkedin_reply_to_conversation` (which sends a real LinkedIn message).

Campaign arguments accept a name as well as an id, and list results are paginated with
`limit`/`offset` so an agent never has to pull thousands of rows into its context.

### Driving the page in a browser

The page exposes a stable action surface — no DOM scraping needed:

```js
INBOX.describe()                  // every action, with the meaning of each filter
INBOX.getState()                  // snapshot: client, user, applied filters, counts, rows, analytics
await INBOX.setFilter('waiting_48')
await INBOX.setSort('oldest_waiting')
await INBOX.search('coach')
await INBOX.openThread(id)
await INBOX.sendReply(id, 'text') // real LinkedIn message; requires can_reply
await INBOX.getAnalytics('90')
```

The same snapshot is mirrored into `<script id="inbox-data" type="application/json">`, so a tool
that can only read HTML still gets structured state. Conversation rows carry
`data-conversation-id`, `data-direction`, `data-status`, `data-sender` and `data-waiting-hours`;
KPI tiles carry `data-kpi`; every chart has a "Show table" text equivalent.

---

## Reading the numbers

- `last_direction` — `REPLY` means the lead spoke last (needs an answer); `SENT` means you did.
- `conversation_status` — `replied` (lead replied once) vs `engaged_conversation` (back and forth).
- `waiting_hours` — hours since the lead's last message; only set when they spoke last.
- Analytics `db.range.*` follows the selected period; `db.live.*` is current state and ignores it.
- Day/hour buckets in the reply heatmap are **IST**; all raw timestamps are UTC ISO-8601.
- A period showing **0 invitations** is not a bug — it means the campaign sent no new connection
  requests in that window and is only messaging people who already accepted.

## Editing the front end

`index.html` is the whole app. Push to `main` and GitHub Pages redeploys; the HTML is cached for
about 10 minutes, so append `?v=<something>` when verifying a change.
