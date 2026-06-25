# Bubble.io integration

Build a live-streaming app on Bubble using the Locust API (`api.driftin.live`). This guide uses Bubble’s **API Connector** and **backend workflows** so secrets stay off the client.

## Before you start

1. Create a tenant at https://www.niilox.com (`/portal` → create account).  
2. Note your **`app_id`** (e.g. `myapp`).  
3. Create an **API key** (`drift_sk_…`) under **API keys** — use it only in **backend** workflows.  
4. Read [Getting started](./GETTING_STARTED.md) if you have not called the API yet.

## Architecture on Bubble

```text
┌─────────────────────────────────────────────────────────┐
│  Bubble page (browser)                                   │
│  - User JWT in custom state / cookie (after login)       │
│  - API Connector calls with dynamic Authorization        │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│  Bubble backend workflow (optional, recommended)         │
│  - drift_sk_… for /platform/* only                       │
│  - Never expose API key to “Current user” on page        │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
              https://api.driftin.live/api/v1
              Header: X-App-ID: myapp
```

| Call type | `Authorization` header | Where to run in Bubble |
|-----------|--------------------------|-------------------------|
| Platform / usage | `Bearer drift_sk_…` | Backend workflow only |
| Rooms, chat, gifts, wallet | `Bearer <user_jwt>` | Page or backend (JWT from login) |

## Step 1 — API Connector (shared settings)

1. **Plugins** → **API Connector** → **Add another API**.  
2. Name: `Locust` (or your brand).  
3. **Use as**: Data (for GET) and Action (for POST/PATCH).

### Shared headers (every call)

Add these as **shared headers** on the API group:

| Name | Value |
|------|--------|
| `X-App-ID` | `myapp` (your tenant id) |
| `Content-Type` | `application/json` |

Do **not** put the API key in shared headers if any call runs from the browser with a user JWT — use per-call headers instead.

### Base URL

All product calls use:

```text
https://api.driftin.live/api/v1
```

Liveness (no `/api/v1`, no auth): `https://api.driftin.live/health` → `{"ok":true}`.

### Verify from terminal (before Bubble)

```bash
curl -s https://api.driftin.live/health

curl -s https://api.driftin.live/api/v1/platform/ping \
  -H "Authorization: Bearer drift_sk_YOUR_KEY" \
  -H "X-App-ID: your_app_id"
```

Replace `your_app_id` with the id from the developer portal (e.g. `dream_api`). **No spaces** in the header value. The key must be created under that same tenant.

---

## Step 2 — Calls to add in API Connector

Below, **Authorization** is set per call unless noted. In Bubble, check **“Add a header”** and use **dynamic** value when the user is logged in.

### 0. Health (optional — no secrets)

Use this only to confirm Bubble can reach the API. **Do not** put `Authorization` or `X-App-ID` on this call.

| Field | Value |
|-------|--------|
| Method | GET |
| URL | `https://api.driftin.live/health` |
| **Data type** | **JSON** |
| **Use as** | Data |

**Initialize** → body `{"ok":true}`.

If you still see *“returns a non-object and you picked JSON”*, the production API may not be on the latest build yet — use **platform ping** (below) or set **Data type** to **Text** and expect body `ok`.

### A. Platform ping (test API key — use this to Initialize)

| Field | Value |
|-------|--------|
| Method | GET |
| URL | `https://api.driftin.live/api/v1/platform/ping` |
| Header | `X-App-ID` = `your_app_id` (can be a **shared** collection header) |
| Header | `Authorization: Bearer drift_sk_YOUR_KEY` |
| **Data type** | **JSON** |

**Initialize call** in the editor. Success body includes `"auth":"api_key"` and `"app_id":"your_app_id"`.

Run this from a **backend workflow** first (**uncheck** “Attempt to make the call from the browser”) so you never publish the key on a page.

### B. Guest session (browse without sign-up)

| Field | Value |
|-------|--------|
| Method | POST |
| URL | `https://api.driftin.live/api/v1/auth/guest` |
| Body type | JSON |
| Body | `{}` |

Response fields to save on the user or in `app` custom state:

- `token` → use as `Authorization: Bearer <token>`  
- `user_id`, `name`

### C. Password login

| Field | Value |
|-------|--------|
| Method | POST |
| URL | `https://api.driftin.live/api/v1/auth/password/login` |
| Body | `{"email":"<email>","password":"<password>"}` |

Response:

```json
{
  "token": "<access_jwt>",
  "refresh_token": "<opaque>",
  "user_id": "<uuid>"
}
```

Store `token` and `refresh_token` in secure custom states (or user fields). Use `token` on all user API calls.

### D. Magic link (email)

**Send link**

| Method | POST |
| URL | `https://api.driftin.live/api/v1/auth/magic/send` |
| Body | `{"email":"user@example.com"}` |

User clicks the link in email (opens your Bubble page with `?token=…` in the URL).

**Verify** (on page load when `token` query param present)

| Method | POST |
| URL | `https://api.driftin.live/api/v1/auth/magic/verify` |
| Body | `{"token":"<from_url>"}` |

Same response shape as password login.

### E. Refresh session

| Method | POST |
| URL | `https://api.driftin.live/api/v1/auth/refresh` |
| Body | `{"refresh_token":"<stored_refresh>"}` |

Use when a call returns **401** — update stored `token` and retry once.

### F. Current user profile

| Method | GET |
| URL | `https://api.driftin.live/api/v1/users/me` |
| Header | `Authorization: Bearer <user_jwt>` |

### G. List rooms (lobby)

| Method | GET |
| URL | `https://api.driftin.live/api/v1/rooms` |
| Header | `Authorization: Bearer <user_jwt>` |

Optional query: `?category=music` (`fun`, `music`, `sports`, `gossip`, `vibe`, `adult`).

Use **Get data from external API** on a repeating group; map JSON list to your UI.

### H. Join room (get LiveKit token)

| Method | POST |
| URL | `https://api.driftin.live/api/v1/rooms/[room_id]/join` |
| Header | `Authorization: Bearer <user_jwt>` |

Replace `[room_id]` with a dynamic path segment in Bubble (use `[]` for dynamic parts in API Connector).

Response includes LiveKit `token` and `room_name` — pass these into your video plugin (LiveKit-compatible client).

### I. Send chat message

| Method | POST |
| URL | `https://api.driftin.live/api/v1/rooms/[room_id]/chat` |
| Header | `Authorization: Bearer <user_jwt>` |
| Body | `{"text":"Hello"}` |

### J. Token packs (store)

| Method | GET |
| URL | `https://api.driftin.live/api/v1/tokens/packs` |
| Header | `Authorization: Bearer <user_jwt>` |

Use fields `token_checkout` / `fiat_checkout` and `payment_modes` to show the right purchase UI.

### K. Buy tokens (Stripe redirect)

| Method | POST |
| URL | `https://api.driftin.live/api/v1/tokens/checkout` |
| Header | `Authorization: Bearer <user_jwt>` |
| Body | `{"pack_id":"<id from packs>"}` |

Response includes a **checkout URL**. In Bubble:

1. Run API Connector action.  
2. **Navigate to external website** → URL = `checkout_url` from response (or open in new tab).

After payment, Stripe redirects to your configured success URL; refresh `GET /users/me` for updated balance.

### L. Fiat VIP room (card, no token wallet)

| Method | POST |
| URL | `https://api.driftin.live/api/v1/payments/fiat/room-access` |
| Header | `Authorization: Bearer <user_jwt>` |
| Body | `{"room_id":"<uuid>"}` |

Redirect user to returned checkout URL (same pattern as token checkout). Requires **Stripe Connect** on your tenant — set in portal **Billing**.

### M. Fiat gift

| Method | POST |
| URL | `https://api.driftin.live/api/v1/payments/fiat/gift` |
| Header | `Authorization: Bearer <user_jwt>` |
| Body | `{"room_id":"<uuid>","gift_id":"<gift_id>"}` |

### N. VIP access with tokens

| Method | GET | `.../rooms/[room_id]/access` |
| Method | POST | `.../rooms/[room_id]/access` |

On **402**, show top-up (call **K**) or fiat flow (**L**) depending on `payment_modes`.

---

## Step 3 — Bubble workflows (recommended patterns)

### Login page

1. User submits email + password.  
2. **API Connector** → `password/login`.  
3. **Set state** `jwt` = result’s `token`, `refresh` = `refresh_token`.  
4. **Navigate** to lobby.

### Lobby repeating group

1. **On page load** → if `jwt` is empty, run `auth/guest` or redirect to login.  
2. **Get data from API** → `GET /rooms` with header `Authorization: Bearer jwt`.  
3. Display `title`, `thumbnail_url`, `viewer_count`, etc.

### Watch room

1. **Join** → `POST .../join` → store LiveKit token in state.  
2. Initialize your video element / plugin with that token.  
3. Poll or refresh chat with `GET .../chat` unless you add WebSockets (below).

### Paywall button

1. `GET .../access` — if `has_access` is no, show price.  
2. On click: if `fiat_checkout` → **L**; else `POST .../access` (tokens) or **K** if balance low (402).

---

## Step 4 — Payments in your app

Configure token packs and card checkout in the [developer portal](https://www.niilox.com) → **Billing**, then use the payments module from [Payments](./PAYMENTS.md). Do not hardcode prices in Bubble — read packs from the API.

---

## Step 5 — WebSockets (realtime)

Bubble has limited native WebSocket support. Options:

| Approach | Best for |
|----------|----------|
| **Polling** `GET /rooms/{id}/chat` every few seconds | Simple MVP |
| **Bubble plugin** that supports WebSocket client | Chat + gifts live |
| **Small middleware** (Cloudflare Worker / your server) that fans out WS → webhook to Bubble | Production scale |

WebSocket URLs (when you use a custom client or plugin):

```text
wss://api.driftin.live/ws/rooms/{roomID}?token=<jwt>
wss://api.driftin.live/ws/me?token=<jwt>
```

Events: `chat`, `gift`, `presence`, `room:seats`, `end` — see [Developer integration](./DEVELOPER_INTEGRATION.md).

---

## Step 6 — Google sign-in on Bubble

Google OAuth returns to the **API**, then redirects to **your** app URL with `?token=`.

Typical pattern:

1. Host a small page on your domain (or Bubble custom domain) at `/auth/finish`.  
2. `POST /auth/google/init` with body including your redirect / portal hint (see [NATIVE_AUTH.md](./NATIVE_AUTH.md)).  
3. On `/auth/finish`, read `token` from query string and store as user JWT.

For B2B tenants, use `portal: "developer"` in the init body when signing in through the Locust developer flow.

---

## Security checklist (Bubble)

- [ ] `drift_sk_…` only in **Backend workflows** → **API Connector** with “Hide from client” / server-only  
- [ ] User JWT in custom state; clear on logout via `POST /auth/logout`  
- [ ] `X-App-ID` matches your portal tenant on every call  
- [ ] Do not log full JWT or API key in Bubble debugger for production apps  
- [ ] Stripe: user completes checkout on Stripe-hosted page (never collect card numbers in Bubble for Locust flows)

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `401 Unauthorized` | Refresh JWT; check `Authorization` spelling |
| Wrong tenant’s data | Set `X-App-ID` to your `app_id`, not `drift` |
| `402 payment_required` | Top up tokens or use fiat checkout |
| `fiat checkout disabled` | Portal → Billing → enable hybrid/fiat + Connect |
| CORS error in browser | API must allow your Bubble app origin — contact support with your Bubble URL |
| API Connector “not initialized” | Click **Initialize call** after editing each endpoint |
| *“returns a non-object and you picked JSON”* on `/health` | Use **JSON** after API deploy, or **Text** on older API; prefer **`/platform/ping`** for key tests |
| `401` on `/platform/ping` | Missing/wrong `Authorization: Bearer drift_sk_…`, or key not for this `X-App-ID` |
| `400 unknown app` | Wrong `X-App-ID` or tenant not created in portal |

---

## Quick reference

| Task | Method | Path |
|------|--------|------|
| Test key | GET | `/platform/ping` |
| Guest | POST | `/auth/guest` |
| Login | POST | `/auth/password/login` |
| Profile | GET | `/users/me` |
| Rooms | GET | `/rooms` |
| Join | POST | `/rooms/{id}/join` |
| Chat | POST | `/rooms/{id}/chat` |
| Token checkout | POST | `/tokens/checkout` |
| Fiat VIP | POST | `/payments/fiat/room-access` |
| Fiat gift | POST | `/payments/fiat/gift` |

Full list: [API.md](./API.md).  
Wiring order: [WIRING_CHECKLIST.md](./WIRING_CHECKLIST.md).
