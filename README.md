# Voite

Real-time spectator scoring for live kickboxing events. Spectators scan a QR code, score each round from their phone alongside the official jury, and see the crowd's verdict pushed back moments later.

The name blends "vote" with a phonetic nod to the Irish pronunciation of "fight".

## What it does

- Spectators join via QR code or weblink: no app install, no account, any online phone
- After each round, voting opens with the standard scores (10-10, or 10-9 / 10-8 in favor of either fighter)
- The crowd's average and vote count are tallied and pushed to every phone, shown alongside the official result once the bout ends
- A single Event Manager drives the night ringside: starting bouts and rounds, opening and closing voting, entering the judges' scorecards
- Fight cards and fighter rosters are configured pre-event; nothing else to run

## Architecture

A Spring Boot modular monolith on Java 21 with Server-Sent Events for real-time push and Postgres as the sole datastore, deployed as Docker Compose (app, Postgres, Caddy) on a single Hetzner VM. MVP target: 5,000 to 10,000 concurrent spectators per event, EM actions visible on phones within ~2 seconds.

## Documentation

- [Functional spec](docs/FUNCTIONAL_SPEC.md): product scope, use cases per role, the event/bout/round lifecycle, scoring rules, and the V1/V2 cut lines
- [Technical spec](docs/TECHNICAL_SPEC.md): architecture, module boundaries, the snapshot-and-events push contract, and resolved design questions
- [Infrastructure](docs/INFRASTRUCTURE.md): provisioning, runtime, backups, failover, load testing, and monitoring

## Status

Design phase. The specs above are the agreed V1 baseline; implementation has not started.
