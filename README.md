# README.md

# STEVE — Smart Terminal Executive & Versioning Environment

STEVE is a homely, deterministic, developer-friendly executive that turns intent into reproducible work: **design → research → plan → build → test → deploy**, with strict receipts and replayability.

STEVE is not “a chatbot that runs commands.” It is an execution system with:
- a **daemon control plane** (`steved`) that enforces policy, budgets, and determinism
- a thin client (`steve`) for CLI/TUI interaction
- container-first execution and evidence-first planning

## Status

Early-stage. The core scaffolding is under active construction. Expect breaking changes.

## Core Workflow

State machine:

`DESIGN → RESEARCH → PLAN → BUILD → TEST → DEPLOY`

Loops:
- `DESIGN ↔ RESEARCH`
- `PLAN ↔ RESEARCH`
- `BUILD ↔ TEST`

High-level semantics:
- **PLAN** produces a frozen, hash-addressed plan artifact.
- **BUILD** consumes the frozen plan to produce build artifacts.
- **TEST** verifies artifacts and produces a test certificate (pass/fail + evidence).
- **DEPLOY** requires a passing test certificate (or an explicit, recorded override).

## Design Goals

- **Determinism and replay**: runs produce receipts; actions are auditable and re-runnable.
- **Homely UX**: friendly interaction surface without sacrificing technical rigor.
- **Safe execution**:
  - container execution is defined by STEVE
  - host/system execution requires explicit user confirmation and is always recorded
- **Small control plane**: `steved` targets a strict memory envelope (256MB) via streaming, bounded queues, and disk spooling.

## Architecture (Planned)

- `steved` (Rust + Tokio): daemon; single-writer DB; scheduler; policy engine; secrets; connectors.
- `steve` (Rust): CLI/TUI client; talks to daemon via unix socket.
- SQLite (`steve.db`) for canonical truth + content-addressed storage (CAS) on disk for blobs/artifacts.
- Per-run event/log stores to keep the canonical DB lean.

## License

STEVE is **GPLv3** licensed. See `LICENSE` for details.

## Contributing

Contributions are welcome, especially around:
- daemon scaffolding (`steved`)
- deterministic receipts / provenance
- container policy enforcement
- streaming log/event ingestion
- state machine correctness tests

See `DEVPATH.md` for roadmap, definitions, and development path.
