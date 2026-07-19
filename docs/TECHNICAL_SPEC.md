# Technical Spec: Voite

Technical design for Voite. See [FUNCTIONAL_SPEC.md](./FUNCTIONAL_SPEC.md) for the product/functional spec these decisions support.

## Architecture Overview

A single Spring Boot modular monolith on Java 21, with Server-Sent Events (SSE) for real-time push and Postgres as the sole datastore.

The workload does not justify services: one Event Manager and 5,000 to 10,000 spectators per event, manual teardown, no historical data. Nothing is a global singleton: state machines, snapshots, and SSE channels are keyed by event, so concurrent events from different Promoters work by construction. The sizing assumption, however, is one full-house event per node; if two Promoters book the same night, either re-run the load test at the combined connection count or give the second event its own VM, both open because of the per-event keying. The hard part is correctness of the three state machines (Event, Bout, Round) and fan-out latency, not scale. A monolith keeps every state transition in one process, so rules like "Start Round implicitly closes voting for the previous round" are method calls, not distributed coordination.

V1 scope (see Out of Scope in the functional spec): Open Scoring, the Promoter self-service app, and the EM's live vote count during open voting are deferred to V2 to fit a part-time build. None of these cuts changes this design: an official round score in V2 is just another domain event through the existing push contract, the Promoter app reuses the `card` module the App Owner tooling drives in V1, and the live count is periodic count events over the EM's existing SSE channel.

### Modules

One deployable, with module boundaries enforced by the build (Gradle/Maven multi-module, or Spring Modulith):

- `roster`: the Promoter's persistent fighter data, the only dataset teardown does not delete
- `card`: event/bout/round configuration, Draft-phase editing, weblink and QR code generation; in V1 driven by App Owner internal tooling, in V2 by the Promoter self-service app (see Out of Scope in the functional spec)
- `runtime`: the three state machines, the only writer of live event state; every transition emits a domain event on an internal bus
- `voting`: vote intake, the 2-second grace timer, tallying
- `push`: subscribes to domain events and fans out to connected spectators over SSE
- `auth`: Spring Security; session login for the Event Manager, spectators anonymous

The internal event bus (Spring `ApplicationEventPublisher`) is the escape hatch for the ~100,000 target: if it materializes, swap the bus for Redis pub/sub and run stateless fan-out nodes, without redesigning the domain.

### Snapshot-and-events contract

Every spectator view is defined as: current snapshot on connect, then incremental events. This single contract covers late join, reconnect, mid-event resume after End Event, and score corrections (a correction is just another event that updates state in place). The `runtime` module owns the snapshot; `push` serves it on every new SSE connection and streams events after it.

Ordering is guaranteed by a single monotonically increasing version number per event, stamped by `runtime` on every state transition. The snapshot carries the version it reflects; every pushed event carries the version that produced it. Clients discard events at or below their snapshot version (closing the race between snapshot read and stream attach) and treat a version gap as a lost event: drop the stream and reconnect, which yields a fresh snapshot. Client correctness thus never depends on delivery guarantees in `push`.

## Resolved Questions

### Real-time delivery: SSE

SSE, not WebSockets or polling. Spectator traffic is almost entirely one-directional: the server pushes state, the phone renders it. The only upstream message is the vote, which is a plain `POST /vote` with normal HTTP semantics (a late vote gets a clean "voting closed" response, per the functional spec).

- Browser `EventSource` gives automatic reconnect for free and tolerates venue Wi-Fi and proxy quirks better than WebSockets
- No event replay: every (re)connect is answered with the full current snapshot, so `Last-Event-ID` is ignored and no replay buffer exists. The snapshot is small (one card, its bouts, rounds, tallies) and served from memory, making snapshot-always cheaper than correct replay bookkeeping
- With virtual threads enabled (`spring.threads.virtual.enabled=true`), 10,000 idle SSE connections on one JVM via `SseEmitter` is a non-issue; no reactive rewrite needed
- Latency targets (~2s for EM actions, ~5s for tallies) are comfortably met by push over an open connection
- Heartbeat: the server sends a comment frame (`:ping`) on every open stream every 15 to 25 seconds. It keeps reverse proxies and carrier NAT from killing idle connections at their 30 to 60 second timeouts, and surfaces zombie sockets (phones gone since screen lock) as write failures for cleanup. Because zombies linger until the next write, the open-connection count overestimates the live audience and is never used as the vote-eligible population
- Reconnect: dropped connections are the normal case, not a failure mode; every screen lock, app switch, or Wi-Fi blip kills the stream and `EventSource` re-establishes it. Clients add small random jitter to the reconnect delay and force a fresh connection on `visibilitychange` to visible rather than trusting a possibly half-open socket. Every (re)connect is answered with the full current snapshot, served from memory and never assembled from Postgres per request, so a reconnect herd of 10,000 phones after a venue Wi-Fi blip stays cheap; this herd, not steady state, is the scenario to load-test

### Stop Bout undo: delayed emission

Stop Bout is the one transition not pushed immediately: the functional spec gives the EM a ~10-second undo window. `runtime` applies the stoppage and schedules the domain event's emission for after the window; undo cancels the scheduled emission and reverts the state, and nothing ever reaches spectators. The EM's own channel shows the pending stoppage immediately; only fan-out is delayed. Once emitted, the stoppage is a normal pushed event and undo is gone (mis-entries go through Score Corrections). The pending stoppage is persisted with its deadline, so a crash inside the window re-arms the timer on restart.

### Identity: anonymous client-side UUID

A random UUID minted client-side into `localStorage` on first load, sent with each vote and used to key the spectator's local vote history. This matches the functional spec's settled stance: clearing browser storage or incognito use loses history, with no server-side recovery, and vote integrity is out of scope for the MVP. Switching devices mid-event simply mints a fresh anonymous identity; prior votes are not transferred.

### Connectivity: fail fast on votes

No offline queuing. A vote queued while offline would mostly arrive after the grace period and be rejected anyway, so queuing adds complexity for no payoff. The vote POST either succeeds or fails immediately with a clear message. The read side is handled by `EventSource` reconnection plus the snapshot-on-connect contract.

### Data store: single-node Postgres

The write burst is at most ~10,000 vote rows spread over a voting window, which Postgres absorbs without tuning as plain per-request single-row inserts. No batching layer: a vote is acked only once its row is committed, so a crash never loses an acked vote. If load testing ever falsifies this, batching is a contained change inside `voting`. Tallying after the grace period is one `GROUP BY` query. Redis, Kafka, or a NoSQL store would be gold-plating at this scale.

- Postgres is authoritative for all state, including live event state, so a JVM restart mid-fight-night resumes correctly (the Ended-to-Live resume behavior in the functional spec requires durable state anyway)
- The current event snapshot is additionally held in memory for reads, since every push and every late joiner needs it
- Flyway for migrations; plain JDBC or Spring Data JDBC over JPA, since the schema is small and the state machines want explicit transitions, not lazy-loaded entity graphs
- Teardown is a transactional delete of the event aggregate; the `roster` tables are untouched
- Connection pool: virtual threads remove the request-thread-pool backpressure that traditionally protects the pool, so the Hikari pool is deliberately small (~10 connections) with a connection timeout shorter than the client's patience. Under the vote burst, requests queue briefly at the pool instead of piling up in Postgres; the vote transaction is one insert, so queueing stays in the milliseconds
- Timers survive restarts: the voting-closed timestamp and any pending Stop Bout deadline are persisted with the state change. On startup `voting` tallies any round whose grace period elapsed without a tally and `runtime` re-arms pending Stop Bout emissions. A restart can delay a tally, never lose one

### Auth: session login, event-scoped roles

Spring Security with server-side sessions and form login for the Event Manager app. No self-service signup: the App Owner provisions accounts manually, per the functional spec.

- Roles: `EVENT_MANAGER` only in V1, each account belonging to a Promoter; a `PROMOTER` role is added with the V2 Promoter app
- Scoping: every event belongs to one Promoter; authorization checks ownership, so an Event Manager drives only their Promoter's events (and in V2, a Promoter edits only their own draft cards)
- The App Owner has no app: provisioning, card configuration, and teardown are operational tasks against the database or thin internal admin endpoints
- Single Event Manager per event is a product constraint (see Out of Scope in the functional spec), not enforced by locking; the last write wins if it is ever violated
- Spectators are unauthenticated; the QR/weblink token identifies the event, not the person
- Session lifetime outlives a fight night: server-side session TTL of 24 hours, not the 30-minute default, so the EM is never logged out ringside mid-event. Re-login is the standard form; no special recovery flow

### Anti-fraud: deferred, with a designed-in seam

The MVP stance is settled (any vote submitted while the button is pressable is legitimate, 2-second grace period), with one piece pulled into V1 because it is a correctness fix, not fraud defense: a unique constraint on (round_id, client_uuid). A double tap or a client retry after a network timeout must not count twice; on conflict the endpoint returns the same success response as the original vote, making submission idempotent. This is also step 1 of the old escalation path, so the remaining escalation is:

1. Per-IP rate limiting at the reverse proxy (defeats naive scripting)
2. Device fingerprinting or signed QR-embedded tokens only if the product ever warrants it

The seam is unchanged: every vote carries the client UUID and arrives at a single endpoint.

### Domains & CORS

Two hostnames (see [INFRASTRUCTURE.md](./INFRASTRUCTURE.md)): `app.` serves the static spectator SPA (Cloudflare-proxied), `api.` serves REST and SSE (direct to origin).

- The QR code and weblink encode the `app.` URL with the event token in the path; the SPA then talks to `api.` cross-origin
- The API allows exactly the `app.` origin, nothing wildcarded; spectator calls are credential-less
- The vote POST's JSON content type triggers a CORS preflight; `Access-Control-Max-Age` is set high so each phone preflights once per session, not per vote
- The Event Manager app is served from the `api.` host itself, so it stays same-origin and its session cookie stays first-party

### Scale & latency (previously resolved)

MVP must handle at least 5,000 concurrent spectators per event (10,000 preferred; ~100,000 if the product succeeds, don't gold-plate for that now). Event Manager updates visible on phones within ~2s; aggregated spectator scores within ~5s after voting closes. The single-node design above meets the MVP numbers; the module boundaries and event bus keep the 100,000 path open.

## Stack Summary

- Java 21+, Spring Boot 3.x, Spring MVC with virtual threads (not WebFlux)
- Spring Security, server-side sessions
- Postgres, Flyway, JDBC/Spring Data JDBC
- State machines as plain enums plus transition methods that validate and emit domain events; no state machine framework
- Spectator App: a tiny static SPA (Preact, Svelte, or vanilla JS with `EventSource`); QR-scan-to-usable must be instant on any phone
- Event Manager app: lower traffic and richer forms; htmx with SSE over Thymeleaf keeps the stack almost entirely in Java, or the same SPA toolchain as the Spectator App if preferred (the Promoter app is V2 and reuses whichever choice this lands on)
- Deployment: one VM or container plus Postgres (co-located or managed); a single node covers the MVP and removes sticky-session concerns for SSE
