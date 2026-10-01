# LiveKit stream monitor

Niilox watches live broadcast rooms for idle streams and dead air. When a host has been alone or has had no viewers for too long, the API prompts them over the room WebSocket. The host confirms with a heartbeat REST call; otherwise the room ends automatically.

**Start here:** [STREAM_HEALTH.md](./STREAM_HEALTH.md) — 5‑minute quickstart (`monitor/status`, first WS event). Use this page for auth, reconnect, and field details.

Self-serve check: `GET /api/v1/livekit/monitor/status` with your `X-App-ID`. If the route is missing or `features` is empty, contact **dev@niilox.com**.

## When prompts fire

| Signal | Typical threshold | What it measures |
|--------|-------------------|------------------|
| Host alone in the SFU room | ~3 minutes | LiveKit participant count ≤ 1 — not chat silence |
| Host publishing, zero viewers | ~5 minutes | Active publisher tracks, `viewer_count == 0` — a live camera with no audience, not a frozen upstream signal |

Chat activity and client-side timers are **not** inputs. Track-presence warnings (`room:quality_warn`) are separate from idle still-live prompts.

When viewers join again, the API clears the prompt (`room:still_live_ack` without a new deadline).

## Still-live flow

1. API broadcasts WebSocket `room:still_live_prompt` with `deadline_at` (unix) and `prompt_seconds`.
2. Host taps **Still live** in your UI → `POST /api/v1/rooms/{roomID}/live-heartbeat`.
3. API broadcasts `room:still_live_ack` and extends the deadline.
4. No heartbeat before the prompt window (default **120 s**) → API ends the room and broadcasts `end` with `reason: idle_timeout`.

## REST

All routes require `X-App-ID` (tenant). Heartbeat requires a registered host JWT.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/livekit/monitor/status` | tenant header | Integration metadata for your tenant |
| POST | `/api/v1/rooms/{roomID}/live-heartbeat` | host JWT | Reset still-live countdown (rate-limited per room, 30/min) |

### Heartbeat auth & tenant isolation

The heartbeat handler checks, in order:

1. **`X-App-ID`** — room must exist in that tenant (`rooms.app_id`).
2. **Bearer JWT** — `sub` must match `rooms.host_id` for an active (not ended) room.
3. **Guest JWTs** are rejected (`403`) by `RequireRegistered`.

A host JWT from tenant A cannot heartbeat a room in tenant B, even if they guess the room UUID. Wrong tenant, wrong host, or ended room → **404** `room not found or unauthorized` (no cross-tenant leak).

### Heartbeat idempotency

| Situation | Behavior |
|-----------|----------|
| No active still-live prompt | **Safe no-op** — `{ "ok": true, "active": false }`. Retries and double-taps are harmless. |
| Active prompt | **Not idempotent** — each successful heartbeat **resets** the auto-end timer and broadcasts `room:still_live_ack` with a new `deadline_at`. Duplicate retries extend the window; debounce in your UI if needed. |

### Heartbeat rate limit

`POST /rooms/{roomID}/live-heartbeat` is **rate-limited per room** (30 requests/minute per room ID, sliding window). Excess calls return **429** with `Retry-After`. Integrators should debounce the still-live button and avoid retry loops on success.

### Status response (example)

```json
{
  "ok": true,
  "app_id": "your_app",
  "stream_monitor": {
    "still_live_seconds": 120,
    "still_live_seconds_note": "Configured host-response window (seconds). Same value as prompt_seconds on WebSocket events — not a live countdown.",
    "features": [
      "track_presence_warnings",
      "idle_still_live_prompt",
      "auto_end_on_timeout"
    ]
  },
  "integrate": {
    "heartbeat": "POST /api/v1/rooms/{roomID}/live-heartbeat",
    "websocket": "room:still_live_prompt → host responds with heartbeat",
    "webhook": "POST /webhook/livekit (room_finished, track_published)"
  }
}
```

#### `still_live_seconds` vs `prompt_seconds`

| Field | Where | Meaning |
|-------|-------|---------|
| `still_live_seconds` | `GET /livekit/monitor/status` | **Configuration metadata** — how long the host has to respond after a prompt (from `LIVE_STILL_PROMPT_SEC`, default 120). |
| `prompt_seconds` | WebSocket `room:still_live_prompt` / `room:still_live_ack` | **Per-event echo** of the same configured window, sent with each prompt for UI labels. |
| `deadline_at` | WebSocket payloads | **Live countdown anchor** — unix timestamp when the room auto-ends if there is no heartbeat. Always use this for timers in your UI. |

## WebSocket events

| Event | Audience | Payload |
|-------|----------|---------|
| `room:still_live_prompt` | room (host replay: host only on reconnect) | `{ room_id, deadline_at, prompt_seconds, detail, replayed? }` |
| `room:still_live_ack` | room | `{ room_id, deadline_at?, prompt_seconds? }` — timer extended or cleared |
| `room:quality_warn` | room | `{ identity, kind, detail }` — missing camera/mic, muted tracks |
| `end` | room | `{ room_id, reason? }` — `idle_timeout` when auto-ended |

### Reconnect behavior

The auto-end timer runs server-side regardless of WebSocket connectivity. If the host drops mid-countdown (spotty WiFi, OBS restart):

1. On reconnect, the API **re-sends** `room:still_live_prompt` to the host only, with the **original** `deadline_at` and `replayed: true`.
2. Persist `deadline_at` in local UI state as a fallback.
3. `GET /livekit/monitor/status` does **not** expose per-room prompt state — it is tenant-wide configuration only.

## LiveKit webhooks

Niilox configures LiveKit webhooks for hosted rooms:

- `room_finished` — sync room state when the SFU closes
- `track_published` — start stream monitoring when a publisher goes live

(You do not need your own LiveKit project for the hosted monitor path documented in [STREAM_HEALTH.md](./STREAM_HEALTH.md).)

## Client integration (React / LiveKit)

```typescript
// Room WebSocket
ws.onmessage = (e) => {
  const ev = JSON.parse(e.data)
  if (ev.type === 'room:still_live_prompt') showModal(ev.payload)
  if (ev.type === 'room:still_live_ack') updateDeadline(ev.payload.deadline_at)
  if (ev.type === 'end' && ev.payload?.reason === 'idle_timeout') teardown()
}

// Host confirms still live (debounce if your button can double-fire)
await fetch(`${API}/rooms/${roomId}/live-heartbeat`, {
  method: 'POST',
  headers: { Authorization: `Bearer ${token}`, 'X-App-ID': appId },
})
// { ok: true, active: true } when a prompt was reset; { ok: true, active: false } when none active
```

See also [STREAM_HEALTH.md](./STREAM_HEALTH.md) for the 5‑minute path and event cheat-sheet.
