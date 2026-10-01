# Niilox API — stream health & platform docs

**Watch LiveKit rooms for dead air** — missing camera/mic tracks, idle hosts, and auto-end when nobody answers “Still live?”

| | |
|---|---|
| **API** | `https://api.niilox.com/api/v1` |
| **Status** | [`https://api.niilox.com/health`](https://api.niilox.com/health) → `{"ok":true}` |
| **Developer portal** | [www.niilox.com](https://www.niilox.com) — keys, billing, usage |
| **Support** | [dev@niilox.com](mailto:dev@niilox.com) |

Every request needs **`X-App-ID: <your_tenant>`**. User actions use a session JWT; server integrations use **`niilox_sk_…`** API keys.

> **Start here:** [**Stream health (5 min)**](./STREAM_HEALTH.md) — `monitor/status`, go live, catch `quality_warn` / `still_live_prompt`.

Deep reference: [LIVEKIT_MONITOR.md](./LIVEKIT_MONITOR.md). Platform plumbing (tenant + guest JWT): [Getting started](./GETTING_STARTED.md).

---

## Platform progress

**→ [PLATFORM_STATUS.md](./PLATFORM_STATUS.md)** — backend progress (operator doc).

| Layer | Status |
|-------|--------|
| Go API + Postgres + native auth | **Production** |
| Stream health (still-live + track-presence warns) | **Production** — any provisioned tenant |
| P2P calls, DMs, Rodent / GeoGig | **Production** |
| `@niilox/sdk` + `@niilox/kyc-ng` | **v0.1 beta** |
| driftin.live | **Production** — reference livestream app |

---

## Quick checks

```bash
curl -s https://api.niilox.com/health
# {"ok":true}

curl -s https://api.niilox.com/api/v1/livekit/monitor/status \
  -H "X-App-ID: myapp"

curl -s https://api.niilox.com/api/v1/platform/ping \
  -H "Authorization: Bearer niilox_sk_YOUR_KEY" \
  -H "X-App-ID: myapp"
```

Create a tenant at [www.niilox.com](https://www.niilox.com) → **Create account** → pick an app id.

Guest sign-in (platform test only — hosts need a registered JWT for rooms):

```bash
curl -s https://api.niilox.com/api/v1/auth/guest \
  -H "X-App-ID: myapp" \
  -H "Content-Type: application/json" \
  -d "{}"
```

---

## Documentation

### Stream health (primary)

| Guide | For |
|-------|-----|
| [**Stream health**](./STREAM_HEALTH.md) | **5‑minute path** — first monitor event |
| [**LiveKit monitor**](./LIVEKIT_MONITOR.md) | Still-live auth, reconnect, field meanings |

### Platform & SDK

| Guide | For |
|-------|-----|
| [**Getting started**](./GETTING_STARTED.md) | Tenant, API key, guest JWT |
| [**Developer workflow**](./DEVELOPER_WORKFLOW.md) | P2P, messaging, BYO storage |
| [**Platform status**](./PLATFORM_STATUS.md) | Operator backlog |
| [**Wiring checklist**](./WIRING_CHECKLIST.md) | End-to-end launch checklist |
| [**Developer integration**](./DEVELOPER_INTEGRATION.md) | Day-to-day flows, WebSockets |
| [**API reference**](./API.md) | Every endpoint |

### Auth & messaging

| Guide | For |
|-------|-----|
| [**Native auth**](./NATIVE_AUTH.md) | Google, Apple, magic link, phone SMS OTP, passwords |
| [**SMS API**](./SMS.md) | Bulk SMS, sender config, numbers |
| [**Authentication overview**](./AUTHENTICATION.md) | JWT model, guest tokens, refresh |

### Products on Niilox

| Guide | Tenant | Stack |
|-------|--------|-------|
| [**Stream health**](./STREAM_HEALTH.md) | *yours* | Rooms + LiveKit monitor (any provisioned tenant) |
| [**Peer signaling**](./PEER_SIGNAL.md) | `rodent`, `geogig` | WebRTC ICE + signal |
| [**GeoGig**](./GEOGIG.md) | `geogig` | Gigs, fiat checkout, P2P video, worker safety |
| [**Worker safety**](./WORKER_SAFETY.md) | `geogig` (+ others) | Field-worker safety |

Reference apps: [GeoGig](https://github.com/Dr-M06/geogig) · [Drift](https://driftin.live) (livestream reference)

### Payments & no-code

| Guide | For |
|-------|-----|
| [**Payments**](./PAYMENTS.md) | Token packs, hosted checkout, webhooks |
| [**Mobile payments**](./MOBILE_PAYMENTS.md) | `mobile_iap` / App Store / Play |
| [**Bubble.io**](./BUBBLE_IO.md) | No-code API Connector setup |

### Platform

| Guide | For |
|-------|-----|
| [**Multi-tenant**](./MULTI-TENANT.md) | `X-App-ID`, isolation, provisioning |
| [**Security**](./SECURITY.md) | Auth model, tenant isolation, payments |

---

## First-party tenants

| `X-App-ID` | Product |
|------------|---------|
| `drift` | Drift — live streaming, chat, gifts, VIP rooms (reference app: [driftin.live](https://driftin.live)) |
| `geogig` | GeoGig — local gigs, safety, P2P video glance |
| `rodent` | Rodent — peer sessions, Drop, paid bookings |
| *yours* | Provision via the [developer portal](https://www.niilox.com) |

---

## Credentials (do not mix)

| Credential | Header | Used for |
|------------|--------|----------|
| **Tenant API key** | `Authorization: Bearer niilox_sk_…` | Server / Bubble backend — `/platform/*` |
| **User session JWT** | `Authorization: Bearer <access_token>` | End users — rooms, gigs, chat, wallet |

---

## About this repo

This repository is the **public documentation mirror** for Niilox integrators. It contains **guides only** — no server source code, no backend tree, and no production secrets.

**Canonical repo:** [niilox-communications/niilox-api](https://github.com/niilox-communications/niilox-api)

Legacy mirror (same docs): [Dr-M06/locust-api](https://github.com/Dr-M06/locust-api)

This tree is docs-only. Server source lives in a private monorepo — never commit backend code or secrets here.

- **In-browser docs:** [www.niilox.com/portal/dashboard/docs](https://www.niilox.com/portal/dashboard/docs)
- **About Niilox:** [www.niilox.com/about](https://www.niilox.com/about)

Questions: [dev@niilox.com](mailto:dev@niilox.com) or open an issue on this repo.

**GitHub topics (canonical repo):** set `observability`, `monitoring`, `livekit`, `webrtc` — drop topics that no longer match stream health.
