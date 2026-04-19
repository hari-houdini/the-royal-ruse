---
name: debug-realtime
description: Diagnose and fix Supabase Realtime channel issues in GDScript. Use when game events are not arriving at clients, WebSocket connections are dropping, or channel subscriptions are silently failing. Triggers include "Realtime not working", "events not arriving", "WebSocket issue", "channel not connecting", "broadcast not received".
---

# Skill: Debug Supabase Realtime Issues

Supabase Realtime uses WebSocket connections to broadcast messages between clients. Failures are often silent — the WebSocket appears connected but events do not arrive. Follow this diagnostic checklist in order.

## Step 1 — Verify the Supabase Project Settings

In Supabase Dashboard → Settings → API:
- [ ] **Realtime is enabled** for the project (toggle under Database → Replication)
- [ ] The table emitting events has **Realtime enabled** (if using DB changes — not needed for broadcast)
- [ ] The anon key is correct in `res://.env`

## Step 2 — Check WebSocket Connection State in GDScript

```gdscript
# Add this temporarily to your adapter to log connection state:
func _process(_delta: float) -> void:
    if _websocket:
        var state: int = _websocket.get_ready_state()
        match state:
            WebSocketPeer.STATE_CONNECTING: print("WS: CONNECTING")
            WebSocketPeer.STATE_OPEN:       print("WS: OPEN")
            WebSocketPeer.STATE_CLOSING:    print("WS: CLOSING")
            WebSocketPeer.STATE_CLOSED:     print("WS: CLOSED — code: %d" % _websocket.get_close_code())
```

## Step 3 — Verify Channel Subscription Message

Supabase Realtime requires a specific subscription message to join a channel. Log the raw WebSocket traffic:

```gdscript
func _process(_delta: float) -> void:
    if not _websocket or _websocket.get_ready_state() != WebSocketPeer.STATE_OPEN:
        return
    _websocket.poll()
    while _websocket.get_available_packet_count() > 0:
        var packet: PackedByteArray = _websocket.get_packet()
        var message: String = packet.get_string_from_utf8()
        print("WS RECEIVED: ", message)  # Log all incoming messages
```

Expected heartbeat message every 30s:
```json
{"event":"heartbeat","payload":{},"ref":null,"topic":"phoenix"}
```

## Step 4 — Verify Channel Join Message Format

The correct Supabase Realtime channel join format:

```gdscript
func _subscribe_to_channel(channel_name: String) -> void:
    var join_msg := {
        "event": "phx_join",
        "topic": "realtime:%s" % channel_name,   # Must be prefixed with "realtime:"
        "payload": {
            "config": {
                "broadcast": { "self": false },
                "presence": { "key": "" }
            }
        },
        "ref": "1"
    }
    _websocket.send_text(JSON.stringify(join_msg))
```

Common mistakes:
- Missing `realtime:` prefix on topic — channel silently fails to join
- Missing `config` key in payload — treated as malformed join
- Using wrong event name (`join` instead of `phx_join`)

## Step 5 — Verify Broadcast Message Format

```gdscript
# Correct broadcast format for Supabase Realtime:
var broadcast_msg := {
    "event": "broadcast",
    "topic": "realtime:session:%s" % session_code,
    "payload": {
        "event": your_event_name,    # The inner event name your listeners use
        "payload": your_data_dict    # The actual data
    },
    "ref": null
}
_websocket.send_text(JSON.stringify(broadcast_msg))
```

## Step 6 — Check RLS on the Channel

If using DB-backed channels (Postgres Changes), RLS must allow the user to read the row being changed. Pure broadcast channels (used in this project) are not affected by RLS.

For broadcast channels, verify the channel name matches exactly on all clients:
- Sender: `"realtime:session:RUSE42"`
- Receiver: `"realtime:session:RUSE42"` — any typo = no messages

## Step 7 — Check Free Tier Limits

In Supabase Dashboard → Reports → Realtime:
- Messages sent per day approaching 2M/month limit?
- Concurrent connections approaching limit?

Free tier: 2M Realtime messages/month, 200 concurrent connections.

## Step 8 — Test with Supabase JS Client in Browser Console

Isolate whether the issue is GDScript-specific or a Supabase configuration issue:

```javascript
// Paste this in your browser console at https://your-project.supabase.co
const { createClient } = supabase
const sb = createClient('https://YOUR_PROJECT.supabase.co', 'YOUR_ANON_KEY')
const channel = sb.channel('session:TEST99')
channel.on('broadcast', { event: 'test_event' }, (payload) => {
  console.log('RECEIVED:', payload)
}).subscribe()

// Then from another tab:
channel.send({ type: 'broadcast', event: 'test_event', payload: { msg: 'hello' } })
```

If this works but GDScript does not: the issue is in the WebSocket implementation, not Supabase config.

## Common Fixes Checklist

| Symptom | Most Likely Cause | Fix |
|---------|------------------|-----|
| WS connects but no messages | Wrong channel name format | Add `realtime:` prefix |
| WS immediately closes | Auth token expired or invalid | Refresh JWT before connecting |
| Only sender receives messages | `self: true` missing in broadcast config | Set `"self": true` if sender also listens |
| Messages arrive then stop | WebSocket not polled in `_process()` | Add `_websocket.poll()` in `_process()` |
| Connection drops every ~60s | No heartbeat sent | Send heartbeat every 30s |
| Channel join silently fails | Missing `config` in join payload | Add config block to phx_join |
