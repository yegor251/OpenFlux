# Reconnect: what happens when the tunnel dies

The contract: **a client process never requires a user action to bring the
tunnel back.** Restarts of the node (the updater), short network outages,
Wi-Fi↔LTE switches and device sleep are survived on their own; the user may
see "reconnecting", never "restart the app". Recovery has three layers, each
with its own clock, and they compose:

1. **Carrier level** — every transport re-establishes its own connection.
2. **Session level** — the Session detects a silent peer and rehandshakes.
3. **Process/platform level** — the app keeps the process alive and restarts
   it when the OS kills it; everything below runs unchanged inside.

## Detection: when is the data plane dead

A carrier is not counted as working merely because its socket is up: only
**traffic heard from the peer** proves a carrier works both ways. Quiet
carriers are pinged (LinkPing every 10 s) and a pong refreshes the clock.

| Bound | Value | What fires |
|---|---|---|
| Keepalive ping | every 10 s of silence per carrier | LinkPing; pong refreshes the carrier's clock |
| Session reset (keepalived peer) | 30 s (`linkTimeout`) of silence on **every** carrier | challenge reset, new handshake |
| Session reset (never keepalived) | 3 min (6×`linkTimeout`) of total silence | the same; covers peers predating keepalive and sessions younger than the first pong |
| First pong | sent immediately after the handshake | keeps the 30 s bound armed from the session's first seconds |
| Lost handshake reply | retried on every repeat | an initial hello is answered every time it is offered |
| Negotiation timeout (cold start) | 20 s, then the client keeps offering | strict `--negotiate` fails the start; with fallbacks the client keeps trying |

The reset generates a fresh challenge, so the next handshake cannot be
poisoned by state from the dead session (sequence numbers reset on both
sides together). On carriers that serve one peer at a time (direct) a fresh
client always displaces the dead session's connection; it cannot be locked
out by a stale one.

## Backoff

- **Carrier redial (direct client):** 200 ms, ×1.4 per failed attempt, cap
  15 s, reset to 200 ms on a working connection. The client never gives up.
- **Document carriers** (yandex, mailru, …): their own reconnect backoff,
  plus re-join with fresh tokens when the carrier refuses them.
- **Handshake pace:** a hello every 250 ms until established, 2 s after the
  first 20 s of trying.
- The client never exits and never stops offering the handshake. There is no
  "gave up" state; the app shows reconnecting for as long as the session is
  down.

## Network died vs node died

The client does not have to tell them apart, and usually cannot: both look
like silence, and the same recovery path serves both.

- **Network switch (Wi-Fi↔LTE):** sockets die, transports redial, the
  session survives on whichever carrier starts reaching the peer again.
  Routing prefers carriers the peer was heard on; a carrier that is
  connected but silent for the peer is never preferred over one that works.
- **Node restart:** same silence; when the node returns, its fresh session
  accepts the client's challenge (and vice versa) - replacement requires the
  echo of a challenge minted for the new sender, so replayed ciphertext
  cannot displace a live session.
- **Node down for a long time:** the client cycles carrier redials and
  handshake offers for as long as the outage lasts and recovers within one
  redial of the carrier coming back (measured: seconds; see below).

## What the app sees

- `IsConnected()` - the session is established and at least one carrier
  reaches the peer. During an outage it drops when the sockets die (or, for
  a silent half-open hang, when the bounds above fire).
- `Mode()` - `""` (nothing yet), `"session"` or `"classic"` (a
  classic-fallback client talking to an old exit).
- `ActiveTransport()` / `ActiveTransports()` - what traffic uses now.
- CLI logs mark each stage: `[SESSION] exit silent on every carrier ...:
  handshaking again`, `handshake OK`, carrier redials.

UI guidance: show "connected" only while `IsConnected()`; while it is false,
show reconnecting. The core recovers without configuration.

## Platform notes

- **CLI / Desktop:** nothing to configure. Sleep/wake looks like a network
  outage: the bounds fire, recovery is automatic. A desktop app should keep
  the core process running and restart it if it exits.
- **Android:** the VPN service keeps the process alive; doze may pause it,
  and on wake the silence bounds fire once and the tunnel returns. Wi-Fi↔LTE
  is a socket-level event handled by carrier redial. Idle keepalive costs
  about 6 small packets per minute - no wakelocks are needed for recovery.
- **iOS (Network Extension):** recovery works the same while the extension
  runs; the system may kill the extension, restarts it, and the core starts
  a fresh handshake - cold start, same bounds. The platform gives no
  always-on guarantee; that is an OS constraint, not a tunnel one.

## Measured acceptance (direct l3 channel, real node)

- Node restart under a live client (session older than the first pong):
  tunnel back in **24–40 s** without user action.
- Node restart within the session's first seconds (before any pong): with
  the current core, recovery on the ordinary path; on cores older than the
  keepalive fix this case stayed a zombie until the client was restarted.
- **5-minute node outage, client process untouched** (2026-10-22): the
  client rehandshaked once, on the 30 s bound, and the tunnel came back
  **~10 s** after the node returned - one carrier redial plus one
  handshake, nothing else.

A session-level outage is invisible to established TCP inside the tunnel
beyond its own retransmissions: once packets flow again, connections
continue.
