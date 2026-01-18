# DEVPATH.md
# STEVE Development Path (DEVPATH)
Version: 0.2 (living document)
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
  (target: 256MB RSS total, with 64MB reserved for tokenizer state), via bounded queues,
  streaming pipelines, and disk spooling.
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
A structured, versioned schema artifact emitted by stages when "frozen."
Manifests must be machine-parseable and validated.

Types:
- Design Manifest
- Research Manifest
- Plan Manifest (Frozen Plan)
- Build Receipt / Provenance Manifest
- Test Certificate
- Deploy Receipt

### 1.4 Design Manifest Schema (Canonical)

The Design Manifest is the contract between DESIGN and all downstream stages.
Schema version: 1

```
DesignManifest {
  schema_version: u32,                     // Always 1 for this version
  manifest_id: ContentHash,                // BLAKE3 hash of canonical serialization
  
  // Identity
  project_id: ProjectID,
  name: String,                            // Human-readable project name
  created_at: Timestamp,
  frozen_at: Timestamp,
  
  // Intent (the "what")
  intent: IntentBlock {
    summary: String,                       // One-paragraph description
    detailed: Option<String>,              // Extended prose if needed
    user_stories: Vec<UserStory>,          // Optional structured stories
  },
  
  // Acceptance Criteria (the "done")
  acceptance_criteria: Vec<AcceptanceCriterion> {
    id: String,                            // AC-001, AC-002, etc.
    description: String,                   // Human-readable criterion
    testable: bool,                        // Can this be mechanically verified?
    verification_hint: Option<String>,     // How to test, if known
  },
  
  // Constraints (the "boundaries")
  constraints: Vec<Constraint> {
    id: String,                            // C-001, C-002, etc.
    type: ConstraintType,                  // { Functional, Performance, Security, Legal, Resource }
    description: String,
    hard: bool,                            // true = axiom (non-violable), false = preference
  },
  
  // Axioms (non-negotiable rules, subset of hard constraints)
  axioms: Vec<AxiomRef>,                   // References to constraints where hard=true
  
  // Interfaces (optional, for projects with defined APIs)
  interfaces: Vec<InterfaceSpec> {
    name: String,
    type: InterfaceType,                   // { CLI, API, Library, File, Stream }
    schema: Option<String>,                // JSON Schema, protobuf, or prose
  },
  
  // Dependencies (known at design time)
  declared_dependencies: Vec<DeclaredDep> {
    name: String,
    rationale: String,
    version_constraint: Option<String>,
  },
  
  // Research questions (seeds for RESEARCH stage)
  research_questions: Vec<ResearchQuestion> {
    id: String,
    question: String,
    priority: Priority,                    // { Must, Should, Could }
    answered_by: Option<ResearchManifestRef>,
  },
  
  // Provenance
  intake_session_id: Option<SessionID>,    // Link to intake conversation
  human_approved: bool,
  approval_timestamp: Option<Timestamp>,
}
```

Design Manifests are immutable once frozen. Amendments create new manifests that reference predecessors.

### 1.5 Frozen Plan
An immutable, hash-addressed plan artifact:
- references inputs (design/research manifests)
- defines an executable DAG (nodes, edges, policies, budgets, environment pins)
- is consumed by BUILD/TEST/DEPLOY

### 1.6 Receipt / Provenance
A machine-checkable record of what occurred:
- inputs (hashes, versions)
- steps executed
- container image digests
- environment and toolchain
- outputs (hashes)
- policy decisions
- user prompts and confirmations (host actions)
- timestamps (with normalization rules)

### 1.7 Test Certificate
Immutable artifact emitted by TEST:
- pass/fail
- evidence pointers
- failure classification (typed)
- policy verdict summary
- optional explicit override record (must be noisy and auditable)

### 1.8 Internal Versioning (VCS-in-DB)
STEVE maintains internal revision semantics:
- content-addressed objects
- immutable trees/snapshots
- revisions/refs/branches/tags
- retention + garbage collection

Git is a supported interoperability target (export/bridge), not necessarily the canonical store.
(Implementation may use Git as a backend internally if chosen, while preserving STEVE semantics.)

### 1.9 "Host Execution Prompt Rule"
Any action outside containers on the host system must:
- prompt the user explicitly
- record the user decision in receipts
- support dry-run preview

The prompt-record-execute pattern is implemented as a reusable primitive (see Section 4.7).

### 1.10 "Homely UI"
STEVE must feel warm and approachable, without sacrificing rigor.
UX includes:
- friendly messaging
- later: role-based creature "crew"
- gentle wellbeing nudges (configurable, non-intrusive)

### 1.11 LLM Integration Layer
STEVE uses LLMs for:
- Intake (natural language → structured intent)
- Summarization (context compression)
- Planning (design → task decomposition)
- Code generation (plan → implementation)
- Error explanation (raw errors → human-readable)

LLM calls are:
- budgeted (token limits per stage, per run, per project)
- logged (full request/response stored in DB, redacted for secrets)
- retriable (with backoff and fallback policies)
- streaming-first (responses processed as streams, not buffered)

---

## 2) Non-Negotiable Invariants ("Physics")

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
- Streaming pipelines: never "load the universe into RAM."
- Logs/events spool to disk (via SQLite) with backpressure.
- DB access must avoid unbounded result sets.
- Token counting costs must be budgeted (64MB reserved for tokenizer state).

### 2.5 Single Writer to Canonical DB
- `steved` is the sole writer to the canonical SQLite DB(s).
- Clients are readers + intent submitters.
- Writes are serialized via a writer queue with short transactions.

### 2.6 Security Posture
- Default-deny networking for containers (policy-defined).
- Hardened container runtime flags (seccomp/AppArmor/SELinux where available).
- Image trust model (digest pinning; signature/SBOM validation in later milestones).
- Secrets never appear in logs/artifacts; redaction is mandatory.
- All databases encrypted at rest via SQLCipher.

### 2.7 UX Truthfulness
- Cute messaging must not obscure actual operations.
- Every "friendly line" must map to a real internal action.
- Error messages must be translated into human-readable explanations (with typed taxonomy).

### 2.8 Data Sovereignty
- All user data remains local (no telemetry, no cloud sync unless explicitly enabled).
- SQLCipher encryption ensures data at rest is protected.
- Key management is user-controlled.

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
- Canonical DB: SQLite in WAL mode (one writer), encrypted via SQLCipher.
- Content-addressed storage (CAS) as BLOBs in SQLite (not filesystem).
- Events stored in SQLite (events table or separate `events.db`), enabling encryption.
- Per-run data stored in SQLite with run_id partitioning.

Database topology:
- `steve.db`: canonical truth (projects, runs, manifests, refs, provenance pointers, CAS objects).
- `events.db`: append-only events with rotation/archival policy.
- Both databases encrypted via SQLCipher with the same key derivation.

Rationale for SQLite-only storage:
- SQLCipher provides transparent encryption for all data at rest.
- Single key management surface (no separate file encryption).
- Atomic transactions across related data.
- Simplified backup (copy two files).
- BLOB storage for CAS avoids filesystem overhead on WSL.

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
- Creature "crew" with roles (owl as communicator, etc.) is a later layer (post-v1).

### 3.7 LLM Integration Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        steved                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              LLM Integration Layer                   │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │   Provider  │  │   Context   │  │   Token     │  │   │
│  │  │  Abstraction│  │  Assembler  │  │   Budget    │  │   │
│  │  │   (trait)   │  │  (streaming)│  │   Enforcer  │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │  Response   │  │   Request   │  │  Tokenizer  │  │   │
│  │  │  Streamer   │  │   Logger    │  │   Cache     │  │   │
│  │  │             │  │  (redacted) │  │  (64MB cap) │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                                │
│                            ▼                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Provider Implementations                │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │  Anthropic  │  │   OpenAI    │  │   Local     │  │   │
│  │  │  (v1 impl)  │  │  (planned)  │  │  (Ollama)   │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

Components:
- **Provider Abstraction**: Trait defining send/stream/tokenize operations.
- **Context Assembler**: Streams context from DB without full materialization.
- **Token Budget Enforcer**: Tracks usage per stage/run/project, enforces limits.
- **Response Streamer**: Processes LLM responses as streams, writes to DB incrementally.
- **Request Logger**: Logs all requests/responses (redacted) for debugging and audit.
- **Tokenizer Cache**: Holds tokenizer vocabulary in memory (64MB budget).

v1 implements Anthropic provider only. Abstraction layer exists for future providers.

### 3.8 Multi-Instance Coordination Protocol (Design)

Even though full implementation is M13, the coordination protocol is designed early
to ensure architectural compatibility.

Coordination model:
- **Lease-based coordination** via a lockfile + metadata file in a shared directory.
- Each `steved` instance registers on startup, deregisters on shutdown.
- Lease file contains: instance_id, pid, start_time, claimed_resources.
- Coordination daemon (M13) replaces lockfile protocol with active management.

Shared resources requiring coordination:
- Container image cache (read-shared, write-exclusive).
- Research cache (read-shared, write-exclusive per URL).
- Port allocation pool (exclusive).
- Global job queue priority (advisory in v1, enforced in M13).
- Disk quota accounting.

IPC surface for future coordinator:
```
CoordinatorRequest {
  ClaimResource { resource_type, instance_id, timeout },
  ReleaseResource { resource_type, instance_id },
  QueryStatus { },
  RegisterInstance { instance_id, pid, capabilities },
  DeregisterInstance { instance_id },
  HeartBeat { instance_id },
}

CoordinatorResponse {
  Granted { lease_id, expires_at },
  Denied { reason, retry_after },
  Status { instances: Vec<InstanceInfo>, resources: Vec<ResourceInfo> },
  Ack,
}
```

v1 behavior (pre-M13):
- Instances use lockfile protocol for critical resources.
- Best-effort coordination; may thrash under heavy multi-instance load.
- Explicit documentation of limitations.

### 3.9 Hash Algorithm Decision

**BLAKE3** is the canonical hash algorithm for STEVE.

Rationale:
- Faster than SHA-256 (especially on modern CPUs with SIMD).
- Cryptographically secure (based on BLAKE2 with improvements).
- Designed for content-addressing use cases.
- Single algorithm simplifies implementation.

All content-addressed objects use BLAKE3. SHA-256 may be computed additionally
for interoperability with external systems (e.g., OCI image digests) but is not canonical.

### 3.10 Attestation Format Decision

**SLSA Provenance v1.0** is the canonical attestation format for supply chain outputs.

Rationale:
- Industry-standard format with broad tooling support.
- Integrates with sigstore, cosign, and container registries.
- Provides clear provenance chain for audits.

STEVE generates SLSA provenance for:
- Build artifacts (binaries, packages).
- Container images (when STEVE builds images).
- Exported research bundles.

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
  - network policy (default-deny)
  - seccomp/AppArmor profiles
  - resource limits (memory, CPU, pids)
  - forbidden flag rejection (privileged, host mounts)

### 4.4 LLM Integration Subsystem
Responsibilities:
- Provider abstraction and connection management
- Context assembly (streaming from DB)
- Token counting and budget enforcement
- Request/response logging (with redaction)
- Retry and fallback policies
- Response streaming and incremental DB writes

Components:
- `llm::Provider` trait (send, stream, count_tokens, model_info)
- `llm::ContextAssembler` (streaming context builder)
- `llm::TokenBudget` (per-stage, per-run, per-project tracking)
- `llm::RequestLogger` (audit log with secret redaction)
- `llm::ResponseStreamer` (incremental processing)

Memory budget: 64MB reserved for tokenizer vocabularies and provider state.

### 4.5 Storage Subsystem
Responsibilities:
- SQLCipher key management and initialization
- Schema migrations
- CAS operations (store, retrieve, gc)
- Event logging with rotation
- Backup and restore

Components:
- `storage::KeyManager` (derive key from passphrase, secure memory)
- `storage::CAS` (content-addressed blob storage in SQLite)
- `storage::EventLog` (append-only event storage)
- `storage::Migrator` (schema versioning and migrations)

### 4.6 Scheduling Subsystem
Responsibilities:
- Job queue management
- Resource allocation (containers, network, ports)
- Backpressure enforcement
- Multi-instance coordination (lockfile protocol pre-M13)

Components:
- `sched::JobQueue` (priority queue with bounded size)
- `sched::ResourcePool` (container slots, port ranges)
- `sched::RateLimiter` (token bucket for API calls, web fetches)
- `sched::Coordinator` (lockfile protocol, upgraded to daemon in M13)

### 4.7 Host Prompt Subsystem
Responsibilities:
- Prompt user for host actions
- Record decisions in receipts
- Support dry-run preview
- Timeout and cancellation handling

The prompt-record-execute pattern:
```rust
pub trait HostAction {
    fn describe(&self) -> String;           // Human-readable description
    fn dry_run(&self) -> DryRunResult;      // Preview without execution
    fn execute(&self) -> ExecuteResult;     // Actual execution
    fn record(&self, decision: Decision, receipt: &mut Receipt);
}

pub enum Decision {
    Approved { timestamp: Timestamp, reason: Option<String> },
    Denied { timestamp: Timestamp, reason: Option<String> },
    Timeout { timestamp: Timestamp },
}
```

All DEPLOY actions and any container escape must use this pattern.

### 4.8 Error Taxonomy Subsystem
Responsibilities:
- Classify errors into typed hierarchy
- Generate human-readable explanations
- Optionally invoke LLM for complex error translation
- Track error patterns for debugging

Error hierarchy:
```
STEVEError
├── StateError (invalid transition, frozen manifest violation)
├── BuildError
│   ├── CompileError (with language-specific subtypes)
│   ├── LinkError
│   ├── DependencyError
│   └── ResourceExhausted
├── TestError
│   ├── AssertionFailed
│   ├── Timeout
│   └── CrashError
├── ContainerError
│   ├── ImagePullFailed
│   ├── StartFailed
│   ├── NetworkDenied
│   └── ResourceLimitHit
├── ResearchError
│   ├── FetchFailed
│   ├── ParseError
│   └── QuotaExceeded
├── LLMError
│   ├── ProviderUnavailable
│   ├── TokenBudgetExceeded
│   ├── ResponseInvalid
│   └── RateLimited
└── InternalError (bugs, should never happen)
```

Each error type has:
- Unique code (e.g., `STEVE-BUILD-001`)
- Human-readable template
- Suggested actions
- Links to documentation (when available)

---

## 5) Data Model Strategy

### 5.1 SQLite Schema Design Principles
- All tables have created_at, updated_at timestamps.
- Soft deletes where appropriate (deleted_at).
- Foreign keys enforced.
- Indexes on common query patterns.
- BLOBs for CAS objects (with size limits per object).

### 5.2 Core Tables (Schema v0)
```sql
-- Projects
CREATE TABLE projects (
    id TEXT PRIMARY KEY,                    -- ProjectID (ULID)
    name TEXT NOT NULL,
    created_at INTEGER NOT NULL,            -- Unix timestamp
    updated_at INTEGER NOT NULL,
    deleted_at INTEGER,
    metadata BLOB                           -- JSON, optional
);

-- Runs
CREATE TABLE runs (
    id TEXT PRIMARY KEY,                    -- RunID (ULID)
    project_id TEXT NOT NULL REFERENCES projects(id),
    state TEXT NOT NULL,                    -- Current state machine state
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL,
    finished_at INTEGER,
    metadata BLOB
);

-- State transitions (audit log)
CREATE TABLE state_transitions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    run_id TEXT NOT NULL REFERENCES runs(id),
    from_state TEXT NOT NULL,
    to_state TEXT NOT NULL,
    trigger TEXT NOT NULL,                  -- What caused the transition
    timestamp INTEGER NOT NULL,
    metadata BLOB
);

-- Manifests (all types)
CREATE TABLE manifests (
    id TEXT PRIMARY KEY,                    -- ContentHash (BLAKE3)
    type TEXT NOT NULL,                     -- design, research, plan, build_receipt, test_cert, deploy_receipt
    run_id TEXT NOT NULL REFERENCES runs(id),
    schema_version INTEGER NOT NULL,
    content BLOB NOT NULL,                  -- Canonical serialization
    created_at INTEGER NOT NULL
);

-- CAS objects
CREATE TABLE cas_objects (
    hash TEXT PRIMARY KEY,                  -- BLAKE3 hash
    size INTEGER NOT NULL,
    content BLOB NOT NULL,
    created_at INTEGER NOT NULL,
    last_accessed_at INTEGER NOT NULL,
    ref_count INTEGER NOT NULL DEFAULT 1
);

-- LLM requests (audit log)
CREATE TABLE llm_requests (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    run_id TEXT REFERENCES runs(id),
    stage TEXT NOT NULL,
    provider TEXT NOT NULL,
    model TEXT NOT NULL,
    input_tokens INTEGER NOT NULL,
    output_tokens INTEGER NOT NULL,
    request_hash TEXT NOT NULL,             -- Hash of request (for dedup)
    response_hash TEXT NOT NULL,            -- Hash of response (stored in CAS)
    latency_ms INTEGER NOT NULL,
    created_at INTEGER NOT NULL
);

-- Token budgets
CREATE TABLE token_budgets (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    scope_type TEXT NOT NULL,               -- project, run, stage
    scope_id TEXT NOT NULL,
    budget_tokens INTEGER NOT NULL,
    used_tokens INTEGER NOT NULL DEFAULT 0,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL
);

-- Secrets (encrypted, references only)
CREATE TABLE secrets (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    encrypted_value BLOB NOT NULL,          -- Additional encryption layer
    created_at INTEGER NOT NULL,
    expires_at INTEGER
);

-- Events (in events.db)
CREATE TABLE events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    run_id TEXT,
    event_type TEXT NOT NULL,
    correlation_id TEXT NOT NULL,           -- For tracing across operations
    payload BLOB NOT NULL,                  -- JSON
    created_at INTEGER NOT NULL
);
CREATE INDEX idx_events_run_id ON events(run_id);
CREATE INDEX idx_events_correlation_id ON events(correlation_id);
CREATE INDEX idx_events_created_at ON events(created_at);
```

### 5.3 SQLCipher Configuration
```rust
// Key derivation
let key = argon2id_derive(passphrase, salt, 64KB memory, 3 iterations);

// SQLCipher pragmas
PRAGMA key = 'x\"{hex_key}\"';
PRAGMA cipher_page_size = 4096;
PRAGMA kdf_iter = 256000;
PRAGMA cipher_hmac_algorithm = HMAC_SHA512;
PRAGMA cipher_kdf_algorithm = PBKDF2_HMAC_SHA512;
```

Key management:
- Passphrase prompted on first `steved` start (or via environment variable for automation).
- Key derived using Argon2id.
- Key held in secure memory (mlock'd) during daemon lifetime.
- Key never written to disk unencrypted.

---

## 6) Feature Catalog (Exhaustive)

### 6.1 State Machine (Core)
F-SM-001 (P0): Explicit state machine with typed states and transitions
F-SM-002 (P0): Transition validation (reject invalid transitions)
F-SM-003 (P0): Transition logging to audit table
F-SM-004 (P0): Allowed loops enforcement (DESIGN↔RESEARCH, PLAN↔RESEARCH, BUILD↔TEST)
F-SM-005 (P1): State machine visualization (for TUI/debugging)
F-SM-006 (P0): Property-based tests for state machine correctness

### 6.2 Manifests
F-MAN-001 (P0): Design Manifest schema v1 (see Section 1.4)
F-MAN-002 (P0): Manifest validation on freeze
F-MAN-003 (P0): Manifest immutability enforcement
F-MAN-004 (P0): Research Manifest schema
F-MAN-005 (P0): Plan Manifest schema (DAG structure)
F-MAN-006 (P0): Build Receipt schema
F-MAN-007 (P0): Test Certificate schema
F-MAN-008 (P0): Deploy Receipt schema
F-MAN-009 (P1): Manifest schema migrations
F-MAN-010 (P0): Property-based tests for manifest validation

### 6.3 Design Stage
F-DSGN-001 (P0): Intake submode (natural language → structured intent via LLM)
F-DSGN-002 (P1): Contracts submode (interfaces, schemas)
F-DSGN-003 (P0): Axioms submode (non-violable rules as hard constraints)
F-DSGN-004 (P0): Tests submode (acceptance criteria)
F-DSGN-005 (P0): Freeze submode (produce validated Design Manifest)
F-DSGN-006 (P1): Design amendment flow (create new manifest referencing predecessor)
F-DSGN-007 (P2): Non-coder "translator" intake agent (aggressive clarification)

### 6.4 Research Stage
F-RES-001 (P1): Scope submode (define questions, budgets, source policy)
F-RES-002 (P1): Search submode (query web via container)
F-RES-003 (P1): Fetch submode (retrieve pages with caps)
F-RES-004 (P1): Extract submode (bounded excerpts)
F-RES-005 (P1): Summarize submode (cited summaries via LLM)
F-RES-006 (P1): Freeze submode (produce Research Manifest)
F-RES-007 (P1): Content mirroring (store full content locally, not just hashes)
F-RES-008 (P2): Headless browser container for JS-rendered content (Playwright, budget-limited)
F-RES-009 (P1): Frozen research mode (default for reproducible builds)
F-RES-010 (P2): Research refresh command (intentional update of cached content)

Research v1 limitation: JS-rendered content not supported. Static HTML only.

### 6.5 Plan Stage
F-PLAN-001 (P0): Decompose submode (design → task graph via LLM)
F-PLAN-002 (P0): Schedule submode (task graph → DAG with ordering)
F-PLAN-003 (P0): Budget submode (assign token/time/network limits per task)
F-PLAN-004 (P1): Assign submode (select toolchains, container images)
F-PLAN-005 (P0): Gate submode (encode policy checks as DAG nodes)
F-PLAN-006 (P0): Enqueue submode (create jobs from DAG)
F-PLAN-007 (P0): Freeze submode (produce immutable Plan Manifest)
F-PLAN-008 (P1): DAG visualization
F-PLAN-009 (P1): Re-schedule without re-decompose (partial plan updates)

### 6.6 Build Stage
F-BUILD-001 (P0): Exec submode (run jobs in containers)
F-BUILD-002 (P0): Container policy enforcement (network deny, resource limits)
F-BUILD-003 (P0): Artifact capture (outputs → CAS)
F-BUILD-004 (P0): Log capture (streaming to DB)
F-BUILD-005 (P0): Receipt generation (Build Receipt with provenance)
F-BUILD-006 (P1): Checkpoint support (save intermediate state for resume)
F-BUILD-007 (P1): Incremental builds (skip unchanged tasks)

### 6.7 Test Stage
F-TEST-001 (P0): Test execution in containers
F-TEST-002 (P0): Test Certificate generation (pass/fail + evidence)
F-TEST-003 (P0): Failure classification (typed errors)
F-TEST-004 (P0): BUILD↔TEST loop support
F-TEST-005 (P1): Coverage tracking (when toolchain supports)
F-TEST-006 (P2): Flaky test detection

### 6.8 Deploy Stage
F-DEP-001 (P0): Dry-run preview
F-DEP-002 (P0): Host prompt rule enforcement
F-DEP-003 (P0): Export verified artifacts only
F-DEP-004 (P0): Deploy Receipt generation
F-DEP-005 (P1): GitHub Actions workflow generation
F-DEP-006 (P2): Container registry push (with signing)

### 6.9 LLM Integration
F-LLM-001 (P0): Provider abstraction trait
F-LLM-002 (P0): Anthropic provider implementation
F-LLM-003 (P0): Streaming response processing
F-LLM-004 (P0): Token counting with budget enforcement
F-LLM-005 (P0): Request/response logging (redacted)
F-LLM-006 (P0): Tokenizer vocabulary caching (64MB budget)
F-LLM-007 (P1): Retry with exponential backoff
F-LLM-008 (P1): Fallback policies (degraded operation on provider failure)
F-LLM-009 (P2): OpenAI provider implementation
F-LLM-010 (P2): Local provider (Ollama) implementation
F-LLM-011 (P1): Context assembly streaming (never materialize full context)

### 6.10 Storage & Encryption
F-STOR-001 (P0): SQLCipher integration
F-STOR-002 (P0): Key derivation (Argon2id)
F-STOR-003 (P0): Secure key memory (mlock)
F-STOR-004 (P0): CAS in SQLite (BLOB storage)
F-STOR-005 (P0): Event logging in SQLite
F-STOR-006 (P0): Schema migrations
F-STOR-007 (P1): Backup command (encrypted export)
F-STOR-008 (P1): Restore command
F-STOR-009 (P2): Key rotation

### 6.11 Container Runtime
F-CONT-001 (P0): Docker Engine integration
F-CONT-002 (P0): Image digest pinning
F-CONT-003 (P0): Network policy (default-deny)
F-CONT-004 (P0): Resource limits (memory, CPU, pids)
F-CONT-005 (P0): Seccomp profile (default)
F-CONT-006 (P1): AppArmor/SELinux profiles
F-CONT-007 (P1): Volume policy (reject host mounts by default)
F-CONT-008 (P2): Podman support
F-CONT-009 (P2): Image signature verification
F-CONT-010 (P2): SBOM generation

### 6.12 Scheduling & Coordination
F-SCHED-001 (P0): Job queue with bounded size
F-SCHED-002 (P0): Token-bucket rate limiting
F-SCHED-003 (P0): Backpressure propagation
F-SCHED-004 (P1): Lockfile-based multi-instance coordination
F-SCHED-005 (P2): Coordination daemon
F-SCHED-006 (P2): Shared cache coordination
F-SCHED-007 (P2): Port allocation pool
F-SCHED-008 (P2): Disk quota enforcement

### 6.13 Error Handling
F-ERR-001 (P0): Typed error hierarchy
F-ERR-002 (P0): Human-readable error templates
F-ERR-003 (P1): Error code registry
F-ERR-004 (P1): Suggested actions per error type
F-ERR-005 (P2): LLM-assisted error explanation (opt-in)
F-ERR-006 (P1): Error pattern tracking

### 6.14 Inspect Mode
F-INSP-001 (P1): Current state query
F-INSP-002 (P1): Active manifests query
F-INSP-003 (P1): Resource usage query (memory, tokens, containers)
F-INSP-004 (P1): Log viewer (streaming)
F-INSP-005 (P1): DAG viewer
F-INSP-006 (P1): Decision history query

### 6.15 UX: CLI, TUI, Guided Workflow
F-UX-001 (P0): CLI-first interface (scriptable commands for all stages)
F-UX-002 (P1): TUI (terminal UI) with mode/stage visualization
F-UX-003 (P1): First-run guided workflow (wizard with progressive disclosure)
F-UX-004 (P2): Non-coder "translator" intake agent
F-UX-005 (P2): Localization-ready message IDs

### 6.16 Homely Layer: Creature Crew & Wellbeing Nudges
F-HOME-001 (P1): Homely messaging layer (one friendly line + one factual detail line)
F-HOME-002 (post-v1): Creature crew with roles (owl communicator, etc.)
F-HOME-003 (P1): Wellbeing nudges ("care pings") - configurable, non-intrusive
F-HOME-004 (P2): "Warmth without interruption" policy

Note: F-HOME-002 (creature crew) is explicitly deferred to post-v1.
A half-implemented mascot system is worse than no mascot system.

### 6.17 Maintenance: GC, Cleanup, Janitor
F-MNT-001 (P1): Garbage collection for CAS (reference counting)
F-MNT-002 (P1): Zombie container reaper ("janitor")
F-MNT-003 (P2): Disk quota policy and enforcement
F-MNT-004 (P1): Event log rotation/archival
F-MNT-005 (P1): Database vacuum (when safe)

### 6.18 Secrets Management
F-SEC-001 (P1): Secrets vault (encrypted storage)
F-SEC-002 (P1): Ephemeral injection (into containers)
F-SEC-003 (P1): Redaction pipeline (logs, receipts, exports)
F-SEC-004 (P1): Sentinel value detection
F-SEC-005 (P2): External secrets integration (e.g., 1Password CLI)

### 6.19 Provenance & Supply Chain
F-PROV-001 (P0): Build receipts with full provenance
F-PROV-002 (P1): SLSA provenance generation
F-PROV-003 (P2): Sigstore integration (signing)
F-PROV-004 (P2): SBOM generation
F-PROV-005 (P2): Attestation verification

### 6.20 Internal Versioning
F-VCS-001 (P1): Content-addressed object storage (via CAS)
F-VCS-002 (P1): Immutable tree snapshots
F-VCS-003 (P1): Refs and tags
F-VCS-004 (P1): Retention policies
F-VCS-005 (P2): Git export bridge
F-VCS-006 (P2): Git import (for existing repos)

---

## 7) Roadmap (Milestones with Exit Criteria)

Milestones are ordered to ship a usable core early, then expand UX and advanced features.

### M0 — Repo Genesis (P0)
Deliverables:
- README.md, DEVPATH.md, LICENSE (GPLv3)
- Cargo workspace scaffold
- CI: fmt/clippy/test
- Property-based testing infrastructure (proptest)
- Design Manifest schema v1 defined

Exit:
- Clean `cargo test` locally and in CI
- Property-based tests for Design Manifest validation pass

### M1 — Daemon Skeleton + IPC + Coordination Design (P0)
Deliverables:
- `steved` starts, listens on unix socket
- `steve status` and `steve ping`
- IPC framing + serialization (binary recommended)
- Multi-instance coordination protocol designed (Section 3.8)
- Host prompt subsystem implemented (prompt-record-execute pattern)

Exit:
- Multiple clients can query concurrently
- Bounded IPC queues exist
- Coordination protocol documented with IPC surface defined
- Host prompt pattern usable (even if no DEPLOY yet)

### M2 — Canonical DB + SQLCipher + Writer Queue + IDs (P0)
Deliverables:
- SQLite schema v0 (Section 5.2)
- SQLCipher integration with key derivation
- WAL mode enabled
- Single-writer DB thread + request queue
- Core IDs (ProjectID/RunID/etc.)
- CAS table for blob storage

Exit:
- DB writes are durable and encrypted
- Key management works (passphrase prompt or env var)
- No unbounded memory in DB paths
- Property-based tests for schema validation

### M3 — State Machine Enforcement (P0)
Deliverables:
- Explicit state machine coded as enum + transition function
- Transition logging to audit table
- Allowed loops enforced
- Property-based tests for transition correctness

Exit:
- Invalid transitions rejected deterministically
- Transitions queryable and inspectable
- proptest covers arbitrary transition sequences

### M4 — LLM Integration Layer v0 (P0)
Deliverables:
- Provider abstraction trait
- Anthropic provider implementation
- Context assembler (streaming)
- Token budget enforcer
- Request/response logger (with redaction)
- Tokenizer cache with 64MB budget

Exit:
- Can make LLM calls from steved
- Token usage tracked and logged
- Context assembled without full materialization
- Budget enforcement prevents overspend

### M5 — CAS + Event Logging + Backpressure (P0)
Deliverables:
- CAS operations in SQLite (store, retrieve)
- Event logging in events.db
- Backpressure on event writes
- Correlation IDs for tracing

Exit:
- Control plane remains stable under event flood
- CAS objects retrievable by hash
- Events queryable by run_id and correlation_id

### M6 — DESIGN Stage + Intake (P0)
Deliverables:
- Intake submode (LLM-driven intent capture)
- Acceptance criteria capture
- Constraints and axioms capture
- Design Manifest freeze and validation

Exit:
- Can capture user intent via CLI
- Design Manifest produced and validated
- Manifest stored in CAS and referenced in DB

### M7 — PLAN Freeze + DAG Artifact (P0)
Deliverables:
- Decompose submode (design → task graph via LLM)
- Schedule submode (task graph → DAG)
- Plan Manifest schema + validation
- `steve plan freeze` creates frozen plan

Exit:
- Frozen plan is hash-addressed and reproducible
- DAG is inspectable (CLI output)
- Plan references Design Manifest by hash

### M8 — Container Harness (BUILD) (P0)
Deliverables:
- Container runner wrapper
- Policy validation (network default-deny, resource limits)
- BUILD executes plan steps
- Artifact capture to CAS
- Log capture to events.db

Exit:
- Build artifacts stored in CAS
- Receipts record image digest + outputs + log pointers
- Network policy enforced (can test with network-requiring step failing)

### M9 — TEST Certificates + BUILD↔TEST Loop (P0)
Deliverables:
- TEST stage runs tests and emits certificate
- Failure classification (typed errors)
- BUILD↔TEST loop operational

Exit:
- Pass/fail gating works
- Certificate is recorded and replayable
- Loop can iterate until tests pass

### M10 — DEPLOY with Host Prompt Rule (P0)
Deliverables:
- DEPLOY stage with dry-run
- Host prompts required and recorded (using M1 subsystem)
- Export verified artifacts only
- Deploy Receipt generation

Exit:
- Deploy receipts exist with user decision recorded
- Override path is explicit and auditable
- Unverified artifacts cannot be exported

### M11 — Error Taxonomy + Translation + Inspect Mode (P1)
Deliverables:
- Typed error hierarchy (Section 4.8)
- Human-readable templates
- Inspect commands (state, manifests, resources, logs)

Exit:
- Failures are understandable without raw exit codes alone
- Inspect mode provides visibility into system state

### M12 — Research v0 (P1)
Deliverables:
- Scope, Search, Fetch, Extract, Summarize submodes
- Content mirroring (full content stored locally)
- Cited summaries
- Frozen research default
- Research Manifest

Exit:
- Research artifacts pinned and referenced by plan
- No live refetch during reproducible runs unless explicit
- DESIGN↔RESEARCH and PLAN↔RESEARCH loops work

Documentation note: v1 Research supports static HTML only. JS-rendered content requires M17+.

### M13 — Secrets v0 + Redaction (P1)
Deliverables:
- Secrets vault (encrypted storage within SQLCipher)
- Ephemeral injection into containers
- Redaction pipeline for logs/receipts/exports
- Sentinel value detection

Exit:
- Secrets never appear in logs/exports (test with sentinel values)

### M14 — TUI + Guided Workflow (P1)
Deliverables:
- TUI with DAG viewer and logs
- Guided first-run wizard
- Progressive disclosure (hides manifests/DAG until needed)

Exit:
- Beginners can complete a project flow without learning DAG vocabulary
- TUI provides meaningful visibility during builds

### M15 — Multi-Instance Broker + Shared Caches (P2)
Deliverables:
- Global coordination daemon (upgrades lockfile protocol)
- Shared cache coordination (images, research)
- Port allocation pool
- Disk quota enforcement

Exit:
- 10 instances can run concurrently without thrashing
- Shared resources accessed safely

### M16 — Homely Layer v1 (P1)
Deliverables:
- Homely messaging layer
- Wellbeing nudges (care pings)
- "Warmth without interruption" policy

Exit:
- Homely UX enhances without interrupting critical loops
- Nudges are configurable and non-intrusive

Note: Creature crew (F-HOME-002) deferred to post-v1.

### M17 — Supply Chain Enhancements (P2)
Deliverables:
- SLSA provenance generation
- Image signature verification
- SBOM generation

Exit:
- Provenance artifacts integrate with external tooling (cosign, etc.)

### M18 — Headless Browser Research (P2)
Deliverables:
- Playwright container for JS-rendered content
- Budget limits and timeouts
- Integration with Research stage

Exit:
- JS-rendered pages can be fetched and cached
- Resource usage bounded

---

## 8) Testing Strategy (STEVE tests itself)

### 8.1 Test Categories

**Unit tests** for pure logic:
- State machine transitions
- Manifest validation
- Token counting
- Error classification

**Property-based tests** (proptest) for:
- State machine correctness (arbitrary transition sequences)
- Manifest schema validation (arbitrary malformed inputs)
- Receipt determinism (same inputs → same outputs)
- DAG validity (no cycles, valid edges)

**Integration tests** for daemon+client:
- Full stage flows (DESIGN → BUILD)
- Multi-client concurrency
- Container execution
- Encryption roundtrip

**Stress tests** ("doctor" mode) for:
- Resource exhaustion resistance
- Backpressure correctness
- Log flood stability
- Concurrent instance behavior

**Security tests** for:
- Secrets leakage (redaction correctness)
- Container escape attempts (policy enforcement)
- Network policy enforcement

### 8.2 Test Invariants
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
- SQLCipher encryption (data unreadable without key)

### 8.3 Test Tooling
- `cargo test` for unit and property tests
- Custom integration test harness for daemon+client
- `steve doctor` command for stress tests
- CI runs all tests on every commit

---

## 9) Non-Goals (for early milestones)

- Full GUI/web UI (TUI is preferred)
- Multi-provider LLM support in v1 (abstraction exists; Anthropic only implemented)
- Complex branching/merging UX in internal VCS before core build/test/deploy works
- Heavy background indexing enabled by default (must remain opt-in and rate-limited)
- Creature crew mascots (deferred to post-v1)
- JS-rendered research content (deferred to M18)
- Windows native support (WSL is required)

---

## 10) Resolved Design Decisions

| Decision | Resolution | Rationale |
|----------|------------|-----------|
| Hash algorithm for CAS | BLAKE3 | Faster than SHA-256, cryptographically sound, designed for content-addressing |
| Git backend vs internal store | Internal store, Git as export | STEVE semantics differ from Git; don't inherit Git complexity |
| Events storage | SQLite (events.db) | SQLCipher provides encryption; atomic transactions; simple backup |
| Headless browser | Playwright in container | Industry standard, good automation support, containerizable |
| Connector/plugin model | Out-of-process helpers | Security isolation for untrusted code |
| Attestation format | SLSA Provenance v1.0 | Industry standard, broad tooling support |
| Encryption | SQLCipher | Transparent encryption, single key surface, well-audited |

---

## 11) Open Design Decisions (remaining)

- Exact Argon2id parameters for key derivation (memory, iterations) — benchmark on target hardware
- Event rotation policy (time-based vs size-based vs hybrid)
- Exact seccomp profile contents (start with Docker default, customize as needed)
- Plugin IPC protocol details (gRPC vs custom binary vs JSON-RPC)
- Wellbeing nudge content and cadence defaults

---

## 12) Logging and Observability

### 12.1 Log Format
All logs use JSON Lines format with required fields:
```json
{
  "ts": "2025-01-18T12:00:00.000Z",
  "level": "info",
  "target": "steved::build",
  "correlation_id": "run_01JFXYZ...",
  "msg": "Container started",
  "fields": { "container_id": "abc123", "image": "rust:1.75" }
}
```

### 12.2 Correlation IDs
Every operation receives a correlation ID that flows through:
- State transitions
- LLM requests
- Container executions
- Event logs

This enables tracing a single user action through all subsystems.

### 12.3 Metrics (P2)
Future milestone may add Prometheus-compatible metrics:
- Token usage by stage/run/project
- Build duration histograms
- Container resource usage
- Error counts by type

---

End of DEVPATH v0.2.
