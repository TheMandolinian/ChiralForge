# ChiralForge

ChiralForge is a **workstream forge** for software projects: it turns a project’s blueprint into **claimable “episodes”** (bounded tasks) with **contracts**, **proof bundles**, and **compatibility checks**. Work can be completed by humans or AI agents, but **main only advances through a Merkle-gated lane** when deterministic requirements are satisfied. This is a repo planner and will help when mulitple devs are interacting.

ChiralForge is designed to be **DRL-first** (powered by HHDRL / HashHelix DRL) and **forge-friendly** (GitHub can remain the code host). The primary goal is to enable **safe, scalable parallelism**: many contributors working simultaneously without turning main into chaos.

---

## Table of contents
- [What problem this solves](#what-problem-this-solves)
- [Core concepts](#core-concepts)
- [The 5 chiral lanes](#the-5-chiral-lanes)
- [Merkle-gated main](#merkle-gated-main)
- [DRL-first design](#drl-first-design)
- [MCP + agent integration](#mcp--agent-integration)
- [Repository layout (planned)](#repository-layout-planned)
- [Episode Contract v1](#episode-contract-v1)
- [Proof Bundle v1](#proof-bundle-v1)
- [Compatibility v1](#compatibility-v1)
- [Lifecycle: from plan to merge](#lifecycle-from-plan-to-merge)
- [Determinism rules](#determinism-rules)
- [Security model (future)](#security-model-future)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)
- [Glossary](#glossary)

---

## What problem this solves

Modern software is complex enough that “just open issues and accept PRs” breaks down under scale:
- Issues are often ambiguous, underspecified, and overlap in scope.
- PR review becomes the bottleneck and “trust” becomes the acceptance system.
- Parallel work produces merge conflicts, integration debt, and inconsistent outcomes.
- AI can produce code quickly, but without **contracts** and **proofs**, it increases risk.

ChiralForge shifts the primitive from “PRs and trust” to **Episodes with Contracts + Proof**:
- Tasks are small, bounded, and machine-checkable.
- Completion is evidence-driven and replay-friendly.
- Main advances only through a deterministic gate, not opinions.

---

## Core concepts

### Project Constitution
The “season bible.” Stable rules for a project:
- definition of done
- invariants (must-never rules)
- module boundaries and ownership
- required gates (lint/typecheck/tests/etc.)
- dependency policy / security policy

A constitution is hashable and versioned.

### Episode
An “episode” is a **claimable task** that is intentionally bounded:
- explicit allowed scope (paths/modules/surfaces)
- explicit required gates
- explicit expected outputs and doc updates
- explicit “no-go” constraints

Episodes are designed to be completed independently and merged safely.

### Episode Contract
The canonical, hashable specification for an episode. It is the single source of truth for what “done” means.

### Proof Bundle
A deterministic evidence bundle proving gates were run and outputs match:
- gate results
- log hashes
- artifact hashes
- environment fingerprint
- patch hash
- base ref tested against (e.g., main SHA)

### Compatibility Certificate
A deterministic check that the change composes with policy and main:
- scope conformance
- interface policy conformance (optional in early versions)
- conflict detection and rule evaluation

### Merkle Gate
The only path that advances “main” (conceptually: the project’s committed state).
The gate consumes contract + proof + compatibility and emits:
- a merge record (linking to code host merge commit ref)
- an updated DRL/Merkle state root

---

## The 5 chiral lanes

ChiralForge is organized around **five chiral lanes** (not “five engines”).
Each lane produces a **hashable artifact** and a corresponding **DRL fact**.

1) **Canon Lane** — Project Constitution
- Publishes the project’s stable constraints and required gates.
- Produces: `canon_hash`

2) **Episode Lane** — Work Packaging
- Publishes claimable episode contracts.
- Produces: `contract_hash`

3) **Proof Lane** — Evidence
- Runs/records deterministic gates and outputs.
- Produces: `proof_hash`

4) **Compatibility Lane** — Composability
- Verifies scope + policy compatibility against the intended base.
- Produces: `compat_hash`

5) **Gate Lane** — Merkle Advancement
- Advances main only if Canon/Episode/Proof/Compat requirements are satisfied.
- Produces: `next_root` (and references a code-host merge ref)

These lanes allow ChiralForge to stay simple:
**Merge eligibility is a pure function of recorded facts.**

---

## Merkle-gated main

ChiralForge treats “main” as a **gated state transition**.

Instead of:
- “a PR was approved, merge it”

ChiralForge aims for:
- “the contract exists”
- “proof bundle exists and passes”
- “compat cert exists and passes”
- therefore “the gate may advance main”

**Important:** In the overlay model, GitHub remains the repository host and merge mechanism.
ChiralForge records the merge commit SHA (or equivalent) and commits to it in HHDRL via a Merkle-rooted event chain.

This provides:
- auditability (“why did this merge?”)
- non-repudiation (“which proofs were used?”)
- replayable context (“what was main at the time?”)

---

## DRL-first design

ChiralForge is “DRL-first” in the sense that the authoritative workflow history is stored as **HHDRL facts**.

### What goes into the DRL
- project constitution publications
- episode contract publications
- proof bundle records (hashes + pointers)
- compatibility certificates
- main advancement events

### What does NOT go into the DRL (by default)
- full logs
- large build artifacts
- bulky binaries

Instead, ChiralForge records **content hashes + pointers** so the ledger remains lean and the evidence remains verifiable.

---

## MCP + agent integration

ChiralForge is designed to be **agent-native** without becoming “agent-dependent.”

MCP (Model Context Protocol) can act as the tool bus:
- agents can browse episodes, claim them, run gates, and submit proofs
- but the **gate** decides based on deterministic evidence

A minimal MCP tool surface (conceptual):
- `episodes.browse(filters)`
- `episodes.get_contract(episode_id)`
- `episodes.claim(episode_id)`
- `proof.run(episode_id, patch_ref)`
- `compat.check(episode_id, submission_id)`
- `gate.enqueue_merge(episode_id, submission_id)`
- `gate.status(queue_id)`

MCP calls can be logged as DRL facts (or hashed inputs) to preserve replayability.

---

## Repository layout (planned)

This repo will likely evolve into a Rust workspace with separated concerns:

- `crates/chiralforge-core/`
  - domain types: Episode, Contract, Proof, Compatibility, Gate
  - DRL event schemas + deterministic state transition rules
- `crates/chiralforge-cli/`
  - local authoring + verification commands
  - create episodes, run gates, generate proof bundles
- `crates/chiralforge-mcp/`
  - MCP server exposing lane operations as tools
- `crates/chiralforge-adapters/`
  - GitHub/GitLab adapters, CI adapters, artifact storage pointers
  - intentionally kept outside the deterministic core
- `spec/`
  - Project Constitution template
  - Episode Contract template
  - Proof Bundle format
  - Compatibility rules format
- `docs/`
  - design notes, threat model, roadmap, examples

---

## Episode Contract v1

An Episode Contract is a **hashable document** that defines “done.”

Minimum fields (v1):
- Episode identity: `episode_id`, `project_id`, `canon_hash`
- Intent: one-paragraph summary of outcome
- Scope bounds:
  - allowed path prefixes and/or module IDs
  - explicit forbidden paths
- Definition of Done:
  - required gates list (lint/typecheck/tests/etc.)
  - expected artifacts (if any)
  - required doc updates (if any)
- Constraints:
  - “must not” list (no new deps, no API breaks, etc.)
- Dependencies:
  - prerequisite episodes (optional)
- Submission requirements:
  - patch format requirements
  - proof bundle required

Why scope matters:
> Scope bounds are the main tool that keeps parallelism from collapsing into merge conflict.

---

## Proof Bundle v1

A Proof Bundle is the evidence record that gates were run.

Minimum fields (v1):
- `episode_id`, `submission_id`
- `base_ref` (the main SHA or equivalent used during verification)
- `patch_hash`
- `env_fingerprint` (toolchain lockfiles + versions hashed)
- gate results:
  - gate id/name
  - pass/fail
  - `log_hash`
  - `artifact_hashes[]` (optional)
- `proof_hash` (hash of the entire proof bundle object)

Principle:
> If the Proof Lane cannot verify it deterministically, it didn’t happen.

---

## Compatibility v1

Compatibility starts simple and grows.

v1 compatibility checks typically include:
- **scope_ok**: patch touches only allowed paths/surfaces
- **base_ok**: base_ref still valid (or rebase required)
- **gates_ok**: required gates match Canon + Contract
- **conflict_detected**: overlap detection (path-based)

Later versions can include:
- interface policy checks (API surface constraints)
- schema migration rules
- dependency policy enforcement
- semantic merge requirements

---

## Lifecycle: from plan to merge

1) **Canon Lane**
- publish/update Project Constitution
- DRL records `canon_published(project_id, canon_hash, version)`

2) **Episode Lane**
- author and publish Episode Contracts against a canon version
- DRL records `episode_published(episode_id, contract_hash, canon_hash, scope, gates, deps)`

3) **Work + Proof Lane**
- claimant implements change in a branch
- proof runner executes required gates, records evidence
- DRL records `proof_recorded(episode_id, submission_id, proof_hash, patch_hash, base_ref, env_fingerprint)`

4) **Compatibility Lane**
- compatibility checker validates scope/policy against base_ref
- DRL records `compat_certified(episode_id, submission_id, compat_hash, ruleset_hash, base_ref)`

5) **Gate Lane**
- gate verifies prerequisites (canon + contract + proof + compat)
- merge occurs on the code host (overlay model)
- DRL records `main_advanced(project_id, prev_root, next_root, merge_ref, episode_id, proof_hash, compat_hash)`

---

## Determinism rules

ChiralForge core logic should be deterministic:
- no wall-clock dependence in state transitions
- no randomness inside the gate logic
- no network I/O inside deterministic core functions
- all non-deterministic inputs must enter explicitly as recorded facts (hash + pointer)

Adapters (GitHub/CI/artifact fetch) may involve network I/O, but their outputs must be treated as **inputs** and recorded (hash/pointer) so decisions are replayable.

---

## Security model (future)

ChiralForge is designed for a world where contributors may be untrusted.
A future security model may include:
- trust tiers for who can claim what
- sandboxed execution runners for proof generation
- artifact provenance requirements
- anti-spam constraints (rate limits, claim expirations)
- cryptographic attestations for proof bundles
- policy that prevents secrets exfiltration in CI

A guiding principle:
> Rewards (if implemented) should be a pure function of proof + merge facts, not social reputation.

---

## Roadmap

### v0: Specs + core data model
- Constitution template
- Episode Contract template
- Proof Bundle format
- Compatibility format
- DRL event schemas for the 5 lanes

### v1: Local workflow (CLI)
- author episodes locally
- run proof generation locally
- produce deterministic proof bundles

### v2: Overlay integrations
- GitHub adapter: open PRs, read checks, record merge refs
- CI adapter: collect artifact hashes and logs pointers
- store DRL facts for episodes/proofs/advancements

### v3: MCP + agent-native operations
- MCP server exposing lane tools
- agent-driven “episode completion” with the gate as final authority

### v4: Marketplace primitives (optional)
- reputation derived from proof + merge
- claim management, anti-fraud, payouts (only after core is stable)

---

## Contributing

ChiralForge is early-stage. Contributions should follow these principles:
- prefer deterministic core logic
- keep adapters separate from the core
- document invariants and hash inputs/outputs
- keep specs stable and versioned

When contributing code, include:
- tests for deterministic state transitions
- updates to spec docs when formats change
- clear rationale in commit messages/PR descriptions

---

## License

TBD by repo owner. (A common approach is “open core”: specs + verifier open source; hosted services optional later.)

---

## Glossary

- **Episode**: a bounded, claimable task.
- **Contract**: the hashable spec for an episode.
- **Proof Bundle**: evidence record that required gates were run and outputs match.
- **Compatibility Certificate**: deterministic composability check outcome.
- **Canon / Constitution**: the stable project ruleset and invariants.
- **Gate Lane**: the only lane permitted to advance main.
- **Merkle Root**: a commitment to a set of facts/states via hashing.
- **DRL (HHDRL)**: HashHelix Deterministic Recurrence Ledger; the provenance + replay backbone.

---
