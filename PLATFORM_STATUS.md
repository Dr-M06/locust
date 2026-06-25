# Niilox platform — progress overview

**Last updated:** June 2026  
**Production API:** `https://api.driftin.live` · **Portal:** [www.niilox.com](https://www.niilox.com)

High-level status for integrators. Internal runbooks, migration lists, and deployment details are not published here — contact **dev@niilox.com** for partner onboarding.

---

## At a glance

| Layer | Status | Notes |
|-------|--------|-------|
| **Go API** | **Production** | Multi-tenant, native auth |
| **Postgres** | **Production** | Primary + replica |
| **LiveKit (livestream)** | **Production** | Drift tenant |
| **Peer / P2P signaling** | **Production** | GeoGig + Rodent tenants |
| **Developer portal** | **Production** | Keys, billing, docs at www.niilox.com |
| **`@niilox/sdk`** | **v0.1 beta** | Monorepo install; not on npm yet |

---

## Products on Niilox

| Tenant | Product | Capabilities |
|--------|---------|--------------|
| `drift` | Drift | LiveKit streaming, chat, gifts, VIP |
| `geogig` | GeoGig | Gigs, safety, P2P video |
| `rodent` | Rodent | Peer sessions, bookings, drop-off |
| `rabbaly` | Rabbaly | Creator storefront |
| *yours* | Your app | Provision via the portal |

---

## SDK (`@niilox/sdk` v0.1)

TypeScript client with modules for auth, rooms, seats, gifts, payments, gigs, peer, DMs, platform, and more.

**Not on npm yet** — install from monorepo or request access via **dev@niilox.com**.

Sensitive flows (capped events, worker safety, paywall handling) are documented in the private integration guide.

---

## What's next (public roadmap)

- npm publish for `@niilox/sdk`
- Integration test suite
- Full GeoGig SDK migration
- Niilox KYC and USSD (announcing soon)

For detailed engineering backlog or ops status, contact **dev@niilox.com**.
