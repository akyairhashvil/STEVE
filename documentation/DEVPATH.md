# DEVPATH.md
# STEVE Development Path (DEVPATH)
Version: 0.1 (living document)
License: GPLv3 (STEVE is GPLv3 from inception)

This document is the canonical roadmap + project definition for STEVE.
It enumerates (1) vocabulary, (2) invariants, (3) architecture decisions, (4) data model strategy,
(5) planned features (exhaustive catalog), and (6) milestone roadmap with exit criteria.

STEVE is designed as serious local infrastructure: deterministic, auditable, container-first,
and resource-polite in its control plane.

---

## 0) Project Summary

STEVE = Smart Terminal Executive & Versioning Environment

STEVE is a local executive that moves work through a controlled lifecycle:

DESIGN → RESEARCH → PLAN → BUILD → TEST → DEPLOY

with explicit iteration loops and strict rules (axioms) enforced mechanically.

The system is daemon-first:
- `steved` (daemon) is authoritative and policy-enforcing.
- `steve` (client) is a thin CLI/TUI that talks to `steved`.

All code execution happens in containers.
Host/system actions are gated by explicit user confirmation.

Control plane resource target:
- `steved` and supporting control-plane processes must remain within a strict memory budget
  (target: 256MB RSS), via bounded queues, streaming pipelines, and disk spooling.
- Execution plane (containers/jobs) is not subject to STEVE-imposed caps; host may impose caps.

---

## 1) Definitions (Vocabulary)

### 1.1 Control Plane vs Execution Plane
- Control Plane: `steved` + workers that perform orchestration, policy enforcement, provenance,
  DB writes, context assembly, indexing, caching, logging pipelines, and connectors.
- Execution Plane: build/test/research tool execution in containers.

Constraint:
- STEVE enforces strict budgets for the control plane.
- STEVE does not impose caps on container/job RAM/CPU (but can still define policy flags,
  isolation/hardening, and scheduling concurrency).

### 1.2 Stages / Modes (State Machine)
Primary chain:

DESIGN → RESEARCH → PLAN → BUILD → TEST → DEPLOY

Allowed loops:
- DESIGN ↔ RESEARCH
- PLAN ↔ RESEARCH
- BUILD ↔ TEST

Stage semantics:
- DESIGN: define intent, contracts, axioms, acceptance criteria, and a design manifest.
- RESEARCH: gather external evidence (cited, cached, hashed), producing a research manifest.
- PLAN: transform design + research into an executable DAG and a frozen plan artifact.
- BUILD: execute plan steps in containers, producing build artifacts and receipts.
- TEST: verify outputs and invariants, producing a test certificate (pass/fail + evidence).
- DEPLOY: export/ship verified artifacts; host actions require explicit user confirmation.

### 1.3 Manifest
A structured, versioned schema artifact emitted by stages when “frozen.”
Manifests must be machine-parseable and validated.

Types:
- Design Manifest
- Research Manifest
- Plan Manifest (Frozen Plan)
- Build Receipt / Provenance Manifest
- Test Certificate
- Deploy Receipt

### 1.4 Frozen Plan
An immutable, hash-addressed plan artifact:
- references inputs (design/research manifests)
- defines an executable DAG (nodes, edges, policies, budgets, environment pins)
- is consumed by BUILD/TEST/DEPLOY

### 1.5 Receipt / Provenance
A machine-checkable record of what occurred:
- inputs (hashes, versions)
- steps executed
- container image digests
- environment and toolchain
- outputs (hashes)
- policy decisions
- user prompts and confirmations (host actions)
- timestamps (with normalization rules)

### 1.6 Test Certificate
Immutable artifact emitted by TEST:
- pass/fail
- evidence pointers
- failure classification (typed)
- policy verdict summary
- optional explicit override record (must be noisy and auditable)

### 1.7 Internal Versioning (VCS-in-DB)
STEVE maintains internal revision semantics:
- content-addressed objects
- immutable trees/snapshots
- revisions/refs/branches/tags
- retention + garbage collection

Git is a supported interoperability target (export/bridge), not necessarily the canonical store.
(Implementation may use Git as a backend internally if chosen, while preserving STEVE semantics.)

### 1.8 “Host Execution Prompt Rule”
Any action outside containers on the host system must:
- prompt the user explicitly
- record the user decision in receipts
- support dry-run preview

### 1.9 “Homely UI”
STEVE must feel warm and approachable, without sacrificing rigor.
UX includes:
- friendly messaging
- later: role-based creature “crew”
- gentle wellbeing nudges (configurable, non-intrusive)

---

## 2) Non-Negotiable Invariants (“Physics”)

### 2.1 Determinism & Replayability
- Runs produce receipts.
- Replays must reconstitute the same inputs and execution environments
  (subject to pinned images and cached research artifacts).
- Non-deterministic steps must be declared and recorded explicitly.

### 2.2 Evidence-First Engineering
- Research outputs must be cached and referenced by hash.
- Plan artifacts cite design + research artifacts by stable IDs.
- Builds must not silently depend on live web unless explicitly allowed and recorded.

### 2.3 Container-Only Code Execution
- No user code runs on the host.
- Containers are the only execution surface for build/test/research tooling.
- Host interactions are limited to gated export/deploy and are explicitly confirmed.

### 2.4 Strict Control-Plane Resource Discipline
- Bounded queues everywhere.
- Streaming pipelines: never “load the universe into RAM.”
- Logs/events spool to disk with backpressure.
- DB access must avoid unbounded result sets.
- Token counting costs must be budgeted.

### 2.5 Single Writer to Canonical DB
- `steved` is the sole writer to the canonical SQLite DB(s).
- Clients are readers + intent submitters.
- Writes are serialized via a writer queue with short transactions.

### 2.6 Security Posture
- Default-deny networking for containers (policy-defined).
- Hardened container runtime flags (seccomp/AppArmor/SELinux where available).
- Image trust model (digest pinning; signature/SBOM validation in later milestones).
- Secrets never appear in logs/artifacts; redaction is mandatory.

### 2.7 UX Truthfulness
- Cute messaging must not obscure actual operations.
- Every “friendly line” must map to a real internal action.
- Error messages must be translated into human-readable explanations (with typed taxonomy).

---

## 3) Architecture Decisions (Current)

### 3.1 Language / Runtime
- Rust for core components (`steved`, `steve`).
- Tokio for async I/O orchestration.
- Dedicated blocking thread(s) for SQLite write path (and other truly blocking work).

### 3.2 Daemon-First Topology
- `steved`: authoritative engine (policy, state machine, scheduling, DB writes, receipts, caches).
- `steve`: thin client (CLI/TUI) over unix socket RPC.

### 3.3 Storage Strategy
- Canonical DB: SQLite in WAL mode (one writer).
- Content-addressed storage (CAS) on disk for blobs/artifacts.
- Per-run directories for high-churn logs/events.

Optional topology:
- `steve.db`: canonical truth (projects, runs, manifests, refs, provenance pointers).
- `events.db` (optional): append-only events (or chunked event files).
- `runs/<run_id>/...`: churn store; cheap retention.

### 3.4 Scheduling Strategy
- Local per-instance scheduler (in `steved`) plus a global coordination daemon/lock protocol
  for multi-instance fairness and shared caches (later milestone).
- Token-bucket rate limiting for control-plane operations (web fetch, log processing, API calls).
- Backpressure-first design.

### 3.5 Network / Container Policy
- Default-deny container networking.
- Research containers may receive scoped allowlists.
- Build/test containers are offline unless explicitly granted.
- Host mounts and privileged flags are rejected by policy (unless explicitly allowed with prompts).

### 3.6 UX Strategy
- CLI-first with a TUI planned (terminal-native).
- Guided workflow for beginners (progressive disclosure) planned.
- Homely messaging and wellbeing nudges are planned; must be non-intrusive and configurable.
- Creature “crew” with roles (owl as communicator, etc.) is a later layer.

---

## 4) System Components (Subsystem Map)

### 4.1 `steved` Core
Responsibilities:
- State machine enforcement
- Project/session management
- DB writer queue and migrations
- Receipt/provenance generation
- Scheduling, budgets, backpressure
- Container orchestration (policy-validated)
- Secrets store & redaction pipeline
- Research caching and evidence store
- Error taxonomy and translation pipeline
- Inspectability endpoints (status, active manifests, logs pointers, budgets)

### 4.2 `steve` Client
- CLI commands and scripting interface
- TUI (planned)
- Guided workflow mode (planned)
- Human-readable output formatting (including translated errors)

### 4.3 Container Runtime Integration
- Docker Engine inside Linux/WSL (initial target).
- Policy enforcement wrapper:
  - image digest pinning
  - network rules
  - mounts allowlists
  - hardened runtime flags
- Checkpoints/snapshots (planned, for long-running builds and replay).

### 4.4 Connectors (Frontier Systems + APIs)
- Provider abstraction exists, but v1 implements one backend.
- Supports API-based “advisor” models (OpenAI API and others).
- LLM is an advisor; engine enforces rules.

### 4.5 Observability / Telemetry
- Structured logs (JSON lines) with correlation IDs.
- Run timeline events stored as structured data (not printf).
- Metrics for scheduler/budgets.

---

## 5) Data Model (Canonical Entities)

### 5.1 Identifiers
- ProjectID, SessionID, RunID, ArtifactID, ManifestID
- Hash IDs for CAS objects (sha256/blake3; selection TBD)
- Deterministic ID derivation rules (documented)

### 5.2 Canonical DB Entities (minimum set)
- projects
- sessions
- runs
- transitions (state machine events)
- manifests (typed, versioned, pointers to CAS)
- artifacts (metadata, pointers to CAS, lineage)
- refs/branches/tags (internal versioning)
- receipts (typed, pointers to CAS/event logs)
- policies (snapshotted configurations per run)
- prompts/confirmations (host gating events)
- errors (typed + translation records)
- caches/indexes metadata (FTS planned)

### 5.3 CAS Layout (on disk)
- `cas/<hash>`: immutable blobs
- Optional: pack/chunk format for efficiency

### 5.4 Run Directory Layout
- `runs/<run_id>/logs/` (chunked)
- `runs/<run_id>/events/` (chunked or `events.db`)
- `runs/<run_id>/tmp/` (ephemeral)
- `runs/<run_id>/exports/` (staged artifacts before gated export)

Retention:
- Run directories are the primary “delete unit.”
- Canonical DB references are cleaned via GC.

---

## 6) Feature Catalog (Exhaustive Planned Features)

This catalog is grouped by domain. Each feature includes:
- Priority: P0 (must for usable v1), P1 (important), P2 (later), P3 (nice)
- Notes and dependencies

### 6.1 Control Flow & State Machine
F-CF-001 (P0): Explicit state machine definition (DESIGN→RESEARCH→PLAN→BUILD→TEST→DEPLOY)
- Includes allowed loops: DESIGN↔RESEARCH, PLAN↔RESEARCH, BUILD↔TEST
- Guards: prerequisites per transition; validation errors typed

F-CF-002 (P0): Transition triggers and commands
- Explicit user commands and programmatic triggers
- Freeze semantics: manifests are emitted on explicit freeze or on successful stage completion

F-CF-003 (P0): Forking / branching runs
- Ability to fork a run lineage at any stage
- Branch refs tracked in internal versioning graph

F-CF-004 (P1): Stage-local submodes (concept doc)
- DESIGN submodes: Intake, Contracts, Axioms, Tests, Freeze
- RESEARCH submodes: Scope, Search, Fetch, Extract, Summarize, Freeze
- PLAN submodes: Decompose, Schedule, Budget, Assign, Gate, Enqueue, Freeze
- BUILD submodes: Exec, Verify, Package, Export, Prove, Merge
- TEST submodes (planned): Run, Classify, Explain, Certify
- DEPLOY submodes (planned): Preview, Prompt, Execute, Verify, Receipt

F-CF-005 (P1): Inspect mode / debug view (read-only)
- Inspect current stage, manifests, context tiers, budgets, and intermediate state pointers

F-CF-006 (P1): Dry-run mode for any stage that would execute actions
- Especially DEPLOY and host-side effects
- Outputs planned commands, container flags, expected outputs

---

### 6.2 Planning & DAG Engine
F-PL-001 (P0): DAG representation
- Nodes: actions (build steps), gates (policy checks), fetch steps, packaging steps, tests
- Edges: dependencies
- Typed nodes with explicit inputs/outputs

F-PL-002 (P0): Plan Freeze artifact (Frozen Plan)
- Hash-addressed, references design/research manifests, environment pins, and policy snapshot

F-PL-003 (P1): Split plan concerns internally (recommended by review)
- Decompose: tasks graph
- Schedule: resource ordering and DAG formation
- Gate: explicit policy nodes

F-PL-004 (P1): Plan simulation / “preflight”
- Validate environment pins, check for missing dependencies, obvious conflicts before BUILD

F-PL-005 (P2): Surgical plan patching
- Modify DAG locally after a failure without full redesign (tracked and auditable)

---

### 6.3 Execution / Build / Test / Deploy
F-EX-001 (P0): Container-only execution harness
- Build and test steps run in containers
- Policy-validates container flags before launch

F-EX-002 (P0): Build outputs
- Single Linux binary OR zipped multi-file project
- Always includes README + provenance manifest + test reports + citations (if used)

F-EX-003 (P0): TEST stage and Test Certificate
- Runs tests
- Produces certificate artifact and stores pass/fail plus evidence pointers

F-EX-004 (P0): BUILD↔TEST loop implementation
- Fast iteration cycle with clear failure classification

F-EX-005 (P0): DEPLOY stage gating
- DEPLOY requires passing test certificate, else explicit recorded override

F-EX-006 (P0): Host Execution Prompt Rule enforcement
- Any host action requires explicit confirmation and is recorded
- Includes export, deploy, external writes outside workspace, and privileged actions

F-EX-007 (P1): Checkpoints / resume for long builds
- Save intermediate state; resume instead of restart
- Implementation options: container snapshot/commit/checkpoint-restore (runtime dependent)

F-EX-008 (P1): Replay
- `steve replay <run_id> [--step <node>]`
- Reconstruct container environment and rerun deterministically using cached artifacts

F-EX-009 (P1): Export adapters (artifact export flexibility)
- Flat directory, zip, binary, container image, deb/rpm, etc. (pluggable)
- Checksums and SBOM included in provenance

F-EX-010 (P2): Port allocation broker for tests that expose services
- Prevent collisions across concurrent runs/instances

---

### 6.4 Container Security & Policy Engine
F-SEC-001 (P0): Default-deny networking policy
- Offline by default for build/test
- Research containers can have scoped allowlists

F-SEC-002 (P0): Container runtime hardening baseline
- Drop caps, no-new-privileges, read-only rootfs where possible, restricted mounts
- seccomp profile support; AppArmor/SELinux integration where available

F-SEC-003 (P1): Image supply chain policy
- Digest pinning mandatory for reproducibility
- Policy around allowed base images and registries

F-SEC-004 (P1): Signature/SBOM verification (sigstore/cosign style)
- Validate image signatures
- Record SBOM pointers in provenance receipts

F-SEC-005 (P1): WSL safety layer
- Detect WSL/Docker specifics and warn
- Strong recommendation: place CAS inside WSL ext4, not /mnt/c

F-SEC-006 (P2): Host interface module (explicit boundary)
- Central API for host prompts, exports, git interactions, and file writes
- Improves auditability and testability

---

### 6.5 Secrets Management & Redaction
F-SC-001 (P0): Secrets vault (encrypted at rest)
- Local encrypted storage
- Access controlled by `steved`

F-SC-002 (P0): Ephemeral injection into containers
- Secrets exposed only for job lifetime
- Avoid writing secrets to disk inside run artifacts

F-SC-003 (P0): Log redaction pipeline
- Prevent secrets in logs, receipts, exports
- Sentinel tests to ensure zero leakage

F-SC-004 (P1): Secret-scoped policies
- Per-project secrets
- Least privilege injection by plan declaration

---

### 6.6 Research Engine (Web + Evidence)
F-RS-001 (P0): Research artifact caching
- Store fetched content locally (not just hash of URL)
- Serve from cache for reproducible builds

F-RS-002 (P0): Cited summaries and evidence objects
- Extract bounded excerpts
- Store citations and evidence pointers to cached content

F-RS-003 (P1): Live research vs frozen research
- Frozen research default for reproducibility
- Refresh command for intentional updates

F-RS-004 (P1): JS-heavy research support via headless browser container
- Sandboxed chromium container
- Rate-limited and treated as expensive

F-RS-005 (P1): Link rot handling
- Local mirroring ensures replay even when URLs die
- Citation records include retrieval timestamp and hash

F-RS-006 (P1): Legal/ToS policy disclosure
- Explicit policy and user warning flows for scraping risks

F-RS-007 (P2): Research signing (optional)
- Signed cache artifacts / attestations for integrity (sigstore-style)

---

### 6.7 Context Engine (Memory, Summaries, Retrieval)
F-CTX-001 (P0): Tiered context strategy with overflow handling
- Tier 1: critical (current manifests, current task)
- Tier 2: supporting (relevant decisions, relevant research)
- Tier 3: background (older chat, old logs)
- Overflow: shed tier 3 first, then tier 2; never shed tier 1 without explicit user awareness

F-CTX-002 (P0): Streaming context assembly
- Generator/pull-based assembly
- Avoid building giant strings until boundary

F-CTX-003 (P0): Disk spooling for intermediate context chunks
- Keep memory bounded

F-CTX-004 (P1): Decision extraction
- Mechanisms:
  - explicit user “mark decision” support
  - optional LLM-based decision extraction (opt-in; cost-aware)

F-CTX-005 (P1): SQLite FTS / indexed search retrieval
- Retrieve relevant snippets on demand
- Hybrid “hot window” strategy (small in-RAM cache for active files)

F-CTX-006 (P2): Summarization policy & lifecycle
- When to summarize, how to store, how to avoid losing nuance
- Optional multi-summary levels

---

### 6.8 Error Handling & Human Translation
F-ERR-001 (P0): Structured error taxonomy
- Typed failures: container crash, compile failure, test failure, network, policy denial,
  resource exhaustion (control plane), timeout, invalid manifest, etc.

F-ERR-002 (P0): Human-readable error translation templates
- Plain-language explanations and recommended next steps

F-ERR-003 (P1): Optional “Explain This Error” LLM pass (opt-in)
- Use cheap model; strict cost controls; never required

F-ERR-004 (P1): Correlation IDs across transitions and container runs
- For debugging and supportability

---

### 6.9 Scheduling, Multi-Instance, and Shared Resources
F-SCH-001 (P0): Local scheduler with backpressure
- Control-plane token buckets: job launches, web fetches, log processing, API calls
- Worker pool caps for control-plane tasks (not execution plane)

F-SCH-002 (P1): Global coordination daemon / broker for multi-instance
- Manage fairness across instances
- Shared caches coordination
- Port allocation
- Disk quota enforcement
- Crash recovery and leases

F-SCH-003 (P1): Two-tier scheduling architecture
- Per-instance local queue + global coordinator quotas

F-SCH-004 (P1): Disk I/O governor (fsync budget)
- Token bucket for heavy DB writes and churn operations
- Prevent IOPS thrash on consumer SSDs

F-SCH-005 (P2): Preemption and prioritization
- Interactive tasks prioritized over background maintenance

---

### 6.10 Internal Versioning & Git Interop
F-VCS-001 (P0): Content-addressed objects + immutable snapshots
- Objects: blob/tree/commit-like semantics in DB
- Dedupe by hash

F-VCS-002 (P0): Refs and history
- Revisions, refs, tags
- Retention policies + GC

F-VCS-003 (P1): Git export bridge (`steve gitify` / `steve export --git`)
- Convert internal history to Git commits
- Preserve tags/branches where possible

F-VCS-004 (P2): Bidirectional sync (Git ↔ STEVE)
- Import from Git repo into internal store
- Sync back out while preserving provenance

F-VCS-005 (P2): “Git-as-backend” option (implementation choice)
- Use Git plumbing internally while keeping STEVE semantics and UI
- Decision TBD; must not compromise receipts and manifests

---

### 6.11 CI/CD Parity & Workflow Generation
F-CI-001 (P1): Local CI/CD parity mode
- Containerized steps, pinned images, offline by default

F-CI-002 (P1): GitHub Actions workflow generation
- Translate plan/manifests into GH Actions
- Include pinned images and cache usage
- Secrets injection support (mapped to GH secrets)

F-CI-003 (P2): SLSA-style attestations
- Provenance receipts compatible with supply-chain tooling

---

### 6.12 Observability, Inspectability, and “Doctor”
F-OBS-001 (P0): Structured logging (JSON lines) in control plane
- Correlation IDs, event types, timestamps

F-OBS-002 (P0): Inspect endpoints / commands
- Inspect: active state, manifests, DAG, budgets, current logs pointers

F-OBS-003 (P1): `steve doctor` stress suite
- Many concurrent clients
- Log flood tests
- DB growth tests
- Budget invariants (256MB)
- WSL filesystem placement warnings

F-OBS-004 (P2): Visual DAG debugger in TUI
- Show running/failed nodes
- Click/enter to view logs and evidence

---

### 6.13 UX: CLI, TUI, Guided Workflow
F-UX-001 (P0): CLI-first interface
- Scriptable commands for all stages

F-UX-002 (P1): TUI (terminal UI)
- Mode/stage visualization
- Logs viewer
- DAG viewer
- Inspect mode

F-UX-003 (P1): First-run guided workflow (wizard)
- Describe intent → generate design → review → approve research scope → plan → build/test → deploy
- Progressive disclosure hides DAG/manifests until needed

F-UX-004 (P2): Non-coder “translator” intake agent
- Interviews user to produce contracts/axioms/tests
- Aggressive clarification before freezing design

F-UX-005 (P2): Localization-ready message IDs
- Messages stored as IDs with variants (cute layer is configurable)

---

### 6.14 Homely Layer: Creature Crew & Wellbeing Nudges
F-HOME-001 (P1): Homely messaging layer (cute but truthful)
- One friendly line + one factual detail line
- Not spammy; progress-triggered updates

F-HOME-002 (P2): Creature crew with roles (not a single mascot)
- Owl responsible for communication UX layer
- Other creatures mapped to roles (planning, testing, building, etc.)
- Pack system and user customization

F-HOME-003 (P1): Wellbeing nudges (“care pings”)
- Gentle reminders: eat, drink water, touch grass, posture, eye breaks
- Non-intrusive, configurable cadence, quiet hours
- UI surface only (no run-log pollution unless explicitly enabled)

F-HOME-004 (P2): “Warmth without interruption” policy
- Never interrupt critical failures or active build/test loops
- Surface nudges when idle or at safe checkpoints

---

### 6.15 Maintenance: GC, Cleanup, Janitor
F-MNT-001 (P1): Garbage collection for CAS and DB references
- Retention policies per project/run
- Safe deletion and compaction strategy

F-MNT-002 (P1): Zombie process/container reaper (“janitor”)
- Kill/cleanup leaked containers
- Prune dangling images (policy-aware)
- Vacuum/optimize DB when safe

F-MNT-003 (P2): Disk quota policy and enforcement
- Per-project quotas
- Global quotas under multi-instance broker

---

## 7) Roadmap (Milestones with Exit Criteria)

Milestones are ordered to ship a usable core early, then expand UX and advanced features.

### M0 — Repo Genesis (P0)
Deliverables:
- README.md, DEVPATH.md, LICENSE (GPLv3)
- Cargo workspace scaffold
- CI: fmt/clippy/test
Exit:
- Clean `cargo test` locally and in CI

### M1 — Daemon Skeleton + IPC (P0)
Deliverables:
- `steved` starts, listens on unix socket
- `steve status` and `steve ping`
- IPC framing + serialization (binary recommended)
Exit:
- Multiple clients can query concurrently
- Bounded IPC queues exist

### M2 — Canonical DB + Writer Queue + IDs (P0)
Deliverables:
- SQLite schema v0
- WAL mode enabled
- Single-writer DB thread + request queue
- Core IDs (ProjectID/RunID/etc.)
Exit:
- DB writes are durable
- No unbounded memory in DB paths

### M3 — State Machine Enforcement (P0)
Deliverables:
- Explicit state machine coded and test-covered
- Transition logging
- Allowed loops enforced
Exit:
- Invalid transitions rejected deterministically
- Transitions queryable and inspectable

### M4 — CAS + Run Directories + Log/Event Spooling (P0)
Deliverables:
- `cas/` store with hash addressing
- `runs/<run_id>/...` structure
- Chunked log spooling with backpressure
Exit:
- Control plane remains stable under log flood
- DB remains lean

### M5 — PLAN Freeze + DAG Artifact (P0)
Deliverables:
- Plan DAG model
- Frozen Plan schema + validation
- `steve plan freeze` creates frozen plan
Exit:
- Frozen plan is hash-addressed and reproducible
- DAG is inspectable

### M6 — Container Harness (BUILD) (P0)
Deliverables:
- Container runner wrapper
- Policy-validation on run (network default-deny in place)
- BUILD executes a minimal hardcoded plan step
Exit:
- Build artifacts stored in CAS
- Receipts record image digest + outputs + logs pointers

### M7 — TEST Certificates + BUILD↔TEST Loop (P0)
Deliverables:
- TEST stage runs tests and emits certificate
- Failure classification typed
- BUILD↔TEST loop operational
Exit:
- Pass/fail gating works
- Certificate is recorded and replayable

### M8 — DEPLOY with Host Prompt Rule (P0)
Deliverables:
- DEPLOY stage with dry-run
- Host prompts required and recorded
- Export verified artifacts only
Exit:
- Deploy receipts exist
- Override path is explicit and auditable

### M9 — Research v0 (P1)
Deliverables:
- Cached content mirroring
- Cited summaries (bounded excerpts)
- Frozen research default
Exit:
- Research artifacts pinned and referenced by plan
- No live refetch during reproducible runs unless explicit

### M10 — Error Taxonomy + Translation + Inspect Mode (P1)
Deliverables:
- Typed error hierarchy
- Human-readable templates
- Inspect commands
Exit:
- Failures are understandable without raw exit codes alone

### M11 — Secrets v0 + Redaction (P1; required for serious integrations)
Deliverables:
- Vault + ephemeral injection + redaction
Exit:
- Sentinel secrets never appear in logs/exports

### M12 — TUI + Guided Workflow (P1/P2)
Deliverables:
- TUI with DAG viewer and logs
- Guided first-run wizard
Exit:
- Beginners can complete a project flow without learning DAG vocabulary

### M13 — Multi-Instance Broker + Shared Caches (P2)
Deliverables:
- Global coordination daemon / lease protocol
- Shared cache coordination
- Port allocation
Exit:
- Multiple instances behave politely, no thrash

### M14 — Supply Chain Enhancements (P2)
Deliverables:
- Image signature checks
- SBOM/attestation generation
Exit:
- Provenance artifacts integrate with external tooling

### M15 — Homely Crew + Wellbeing Nudges (P1/P2)
Deliverables:
- Care pings
- Role-based creature crew (owl communicator, etc.)
Exit:
- Homely UX enhances without interrupting critical loops or polluting logs

---

## 8) Testing Strategy (STEVE tests itself)

STEVE must test:
- State machine correctness: allowed transitions and loops
- Manifest schema validation and migrations
- Receipt completeness and determinism properties
- Host prompt rule enforcement
- Container policy enforcement (default-deny network, forbidden flags)
- Backpressure correctness (bounded queues)
- Log flood stability (no OOM)
- DB stability (no unbounded result sets)
- WSL warnings and filesystem placement guidance
- Secrets leakage tests (redaction correctness)

Tooling:
- Unit tests for pure logic
- Integration tests for daemon+client
- “Doctor” stress tests for resource and concurrency

---

## 9) Non-Goals (for early milestones)
- Full GUI/web UI (TUI is preferred)
- Multi-provider model support in v1 (abstraction only; one backend implemented)
- Complex branching/merging UX in internal VCS before core build/test/deploy works
- Heavy background indexing enabled by default (must remain opt-in and rate-limited)

---

## 10) Open Design Decisions (tracked here)
- Hash algorithm(s) for CAS (sha256 vs blake3 vs both)
- Git backend adoption internally vs pure internal store (must preserve STEVE semantics)
- Events storage: events.db vs chunked files
- Headless browser implementation details and budgets
- Exact connector/plugin model (in-process vs out-of-process helpers)
- Attestation formats for supply chain outputs

End of DEVPATH.
