# Stream health — 5‑minute quickstart

Watch LiveKit broadcast rooms for **dead air**: missing camera/mic tracks, idle hosts, and auto-end when nobody responds.

> **Honest scope today:** monitoring runs on **Niilox rooms** (`POST /api/v1/rooms` → SFU watch). It is not yet a standalone “bring your own LiveKit only” sidecar. Any provisioned tenant can call the monitor APIs; [driftin.live](https://driftin.live) is the reference livestream app.

| | |
|---|---|
| **API** | `https://api.niilox.com/api/v1` |
| **Status** | `https://api.niilox.com/health` → `{"ok":true}` |
| **Deep dive** | [LIVEKIT_MONITOR.md](./LIVEKIT_MONITOR.md) (still-live edge cases, auth, rate limits) |

---

## 1. Confirm the API and your tenant (~30 s)

```bash
curl -s https://api.niilox.com/health
# {"ok":true}

curl -s https://api.niilox.com/api/v1/livekit/monitor/status \
  -H "X-App-ID: myapp"
```

Expected shape:

```json
{
  "ok": true,
  "app_id": "myapp",
  "stream_monitor": {
    "still_live_seconds": 120,
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

If `features` is empty or the route 404s, your environment is behind — ping **dev@niilox.com**. Otherwise you are good to integrate without a sales ticket.

---

## 2. Go live in a room (~2 min)

You need a **registered** host JWT (guest tokens cannot create rooms or heartbeat).

```bash
# Example: guest is fine for watching; use portal sign-in / magic link for a host JWT.
# Create room (registered user):
curl -s https://api.niilox.com/api/v1/rooms \
  -H "Authorization: Bearer HOST_JWT" \
  -H "X-App-ID: myapp" \
  -H "Content-Type: application/json" \
  -d '{"title":"monitor test","category":"vibe"}'
```

Save `id` (room UUID). Connect the host client to LiveKit with the returned token, and open the room WebSocket:

```
wss://api.niilox.com/ws/rooms/{roomID}?token=HOST_JWT
```

Creating a room starts the media watcher (`WatchRoom`). Publishing camera/mic (or leaving them off) is what produces quality events.

Full room lifecycle → [API.md](./API.md) (Rooms).

---

## 3. Catch an event (~2 min)

### Quality warn (missing / muted tracks)

With no camera for ~15 s after join, or after losing a track mid-stream, the API broadcasts:

| Product language | WS event | Typical `kind` | Ships today? |
|------------------|----------|----------------|--------------|
| Black / no picture | `room:quality_warn` | `BLACK` | **Yes** — track *presence* (no camera track), not decoded black frames |
| Silent / no audio | `room:quality_warn` | `SILENT` | **Yes** — missing or all-muted mic tracks |
| Frozen frame | — | — | **Roadmap** — decoded frame detection (Cargo feature), not on by default |
| Idle / alone | `room:still_live_prompt` | — | **Yes** — host alone ~3 min or publishing with zero viewers ~5 min |

```typescript
ws.onmessage = (e) => {
  const ev = JSON.parse(e.data)
  if (ev.type === 'room:quality_warn') {
    // { identity, kind: 'BLACK'|'SILENT', detail }
    console.log('quality', ev.payload)
  }
}
```

### Still-live prompt → heartbeat → idle end

1. Server sends `room:still_live_prompt` with `deadline_at` (unix) and `prompt_seconds`.
2. Host UI calls heartbeat before the deadline:

```bash
curl -s -X POST "https://api.niilox.com/api/v1/rooms/ROOM_ID/live-heartbeat" \
  -H "Authorization: Bearer HOST_JWT" \
  -H "X-App-ID: myapp"
# { "ok": true, "active": true }  — timer reset
# { "ok": true, "active": false } — no prompt was active (safe no-op)
```

3. Server sends `room:still_live_ack` (extended or cleared).
4. No heartbeat → room ends with WS `end` and `reason: "idle_timeout"`.

```typescript
if (ev.type === 'room:still_live_prompt') showModal(ev.payload) // use deadline_at for the timer
if (ev.type === 'room:still_live_ack') updateDeadline(ev.payload.deadline_at)
if (ev.type === 'end' && ev.payload?.reason === 'idle_timeout') teardown()
```

Debounce the Still live button (30 req/min per room). Details → [LIVEKIT_MONITOR.md](./LIVEKIT_MONITOR.md).

---

## Event cheat-sheet

| You want to detect | Listen for | Act |
|--------------------|------------|-----|
| Host camera never published / lost | `room:quality_warn` `kind=BLACK` | Warn in UI |
| Host mic never published / all muted | `room:quality_warn` `kind=SILENT` | Warn in UI |
| Host alone or zero viewers too long | `room:still_live_prompt` | Show “Still live?” |
| Host confirmed | after heartbeat → `room:still_live_ack` | Clear / extend timer |
| Host abandoned the stream | `end` `reason=idle_timeout` | Tear down lobby card |

Chat silence and client-side timers are **not** inputs.

---

## What this is not (yet)

- **BYO LiveKit credentials / webhook-only monitor** with no Niilox rooms — product roadmap, not documented as available.
- **Decoded black / frozen / silence** from pixels and PCM — detectors exist in `media-svc` behind a heavy build feature; production default is **track-presence** only.
- Open-sourcing the Rust watcher — separate decision from this hosted API.

---

## Next

| Doc | For |
|-----|-----|
| [LIVEKIT_MONITOR.md](./LIVEKIT_MONITOR.md) | Auth isolation, reconnect replay, field meanings |
| [API.md](./API.md) | Room CRUD, join tokens, WS paths |
| [Getting started](./GETTING_STARTED.md) | Tenant + API key + guest JWT (platform plumbing) |
