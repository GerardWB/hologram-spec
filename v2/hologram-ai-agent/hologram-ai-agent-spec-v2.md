# Verifiable Service Agent — Architecture Spec v2

> **Status:** DRAFT
> **Repo:** `hologram-ai-agent`
> **Supersedes:** Issue #84 (the v1 draft spec). This document replaces it in full.
> **Scope:** A clean, framework-agnostic architecture for the agent. No implementation library
> (NestJS, LangChain, pgvector vendor, etc.) is assumed normative — those are implementation
> choices. This spec defines *what* the system is and *why*, not *which libraries* build it.

---

## Table of Contents

1. [Decisions & Rationale](#1-decisions--rationale)
2. [Terminology](#2-terminology)
3. [Goals & Non-Goals](#3-goals--non-goals)
4. [Architecture Overview](#4-architecture-overview)
5. [Identity, Connection & Trust](#5-identity-connection--trust)
6. [Channel Layer](#6-channel-layer)
7. [Session Lifecycle](#7-session-lifecycle)
8. [Memory Model](#8-memory-model)
9. [Privacy & GDPR Architecture (PBAC)](#9-privacy--gdpr-architecture-pbac)
10. [RBAC & Cross-Session Access](#10-rbac--cross-session-access)
11. [Interaction Features](#11-interaction-features)
12. [Integrity & Audit](#12-integrity--audit)
13. [Platform Plumbing](#13-platform-plumbing)
14. [Data Model](#14-data-model)
15. [Deferred / Open Questions](#15-deferred--open-questions)
16. [Phased Rollout](#16-phased-rollout)

---

## 1. Decisions & Rationale

This table captures every architectural fork resolved during design. It is the canonical record;
the sections below elaborate. Verify this reflects intent before reading the detail.

| # | Decision | Resolution | Rationale |
|---|----------|------------|-----------|
| D1 | Channel architecture | **Channels as first-class peers (Model B).** A channel-agnostic core speaks only Principal / Session / canonical MessageEntry / capability events. DIDComm, AG-UI, A2A are equal adapters. | Clean, framework-agnostic core with no transport dependency; accommodates three distinct identity paths natively. |
| D2 | Role of DIDComm | DIDComm is the **trust & capability anchor** — the only channel that can serve trust-bound operations (auth, credential exchange, auth challenges). It is **not** the privileged transport (no tunneling through it; it may also carry conversation). Per-message channel selection is the agent's routing decision (§6.2), bounded by capability profiles. | AG-UI/A2A have no wallet/sensors; DIDComm does — but it's a fine conversation transport too (e.g. Hologram-browser-only). |
| D3 | Conversation vs capability channel | **Separable.** A Principal has one capability channel (DIDComm) and one-or-more conversation sessions. Capability requests route to the capability channel regardless of where the conversation is. | One wallet, many screens (human); one A2A peer with a DIDComm endpoint for credential exchange (service). |
| D4 | Capability-channel cardinality | **A2A: 1:1** (one A2A session ↔ one DIDComm channel). **AG-UI: shared** (one DIDComm capability channel per Principal, shared across that Principal's AG-UI sessions). | The capability channel is bound to the trust-bearing *device*, not the session. For A2A the device is the peer; for humans it's the singular wallet. |
| D5 | Bootstrap / binding pattern | Two patterns converging on one end-state: **DIDComm-first** (authenticate, then open conversation channel) and **channel-first** (open provisional session, then bind via DIDComm — required for cross-device AG-UI, e.g. QR scan). | DIDComm is always the trust-establishing event; sequencing/topology is the only variable. |
| D6 | Binding integrity | Per-session **binding nonce** echoed over DIDComm; an unbound provisional session has **no Principal, no roles, zero data/tool access**. | Prevents session hijack via leaked QR; an unbound session is an attack surface that must touch nothing privacy-scoped. |
| D7 | Session termination triggers | (1) DIDComm channel closes → its AG-UI sessions close. (2) A new DIDComm channel supersedes an existing one for the same Principal → old channel + its sessions close. (3) The credential underlying a session fails periodic re-verification (revoked / expired / issuer no longer trustable) → that session terminates; for a Principal whose identity credential fails, **all** its sessions close. (4) Idle-timeout → quiet sessions reaped. | (1)(2) are channel-lifecycle; (3) is credential/identity-lifecycle; (4) is hygiene. Opening a new tab *reuses* the existing channel (lightweight confirm) — that is **not** a new channel and does not supersede. |
| D8 | Credential-liveness enforcement | **TTL-bounded revalidation:** on session activity, if cached validity is older than `N` seconds, re-verify (revocation GET + re-walk issuer chain); failure → terminate + require re-presentation. Plus an independent **idle-session timeout**. Uniform across humans (badge + issuer chain) and services (ECS-Service + backing identity). | A revocation check is a cheap remote GET; TTL self-throttles bursts (≤1 check per `N`s per session) and bounds max-staleness-while-active. Idle timeout reaps quiet sessions so none lingers stale. Credential *reissue* is the issuer's concern over its own DIDComm channel — invisible to the agent. |
| D9 | A2A identity | **Symmetric peer-to-peer.** No client/server distinction. Each peer resolves the other as a Principal: `actorIdentity` from **ECS-Service** + `backingIdentity` from the **ECS-Org / ECS-Persona** presented alongside it (connectionType `verifiable-service`). | A2A is peer-to-peer by nature; identity comes from Verifiable Trust, not DIDComm-presented claims. Two services of the same Org are distinct actors sharing one backing identity. |
| D10 | Memory tiers & authority | **Two authoritative stores:** long-term store (every MessageEntry) + working memory (LLM-curated notes). **Everything else derived & rebuildable:** active window, semantic index, window summaries. | Single sources of truth make erasure tractable; derived data rebuilds clean or is discarded. |
| D11 | Working memory vs summarization | **Working memory** = LLM-curated, durable, authoritative, principal-scoped, **principal cannot edit**. **Summarization** = mechanical, transient compression of the active window only, **never persisted as an independent record**. | They stop overlapping: one is curated+durable, one is mechanical+throwaway. Keeps the erasure lineage clean. |
| D12 | Privacy classification | **Two orthogonal axes + one tag.** Visibility: `session-private` / `principal-private` / `organization-shared` / `partner-shared` / `public`. Lifecycle state: `active` / `archived` / `legal-hold` / `crypto-erased` / `deleted`. Purpose: a tag recording the processing purpose (purpose-limitation) — used for **audit + elevation** (D15), not as a continuous retrieval predicate. `organization-shared` = same **backing identity** as the agent's own (home); `partner-shared` = same backing identity as a **recognized partner** — see D31–D32. | A flat enum can't express real combinations (org-shared + legal-hold; principal-private + crypto-erased). Three independent dimensions can. |
| D13 | Access control model | **RBAC tool-gating is the default** (role → permitted tools = usual tasks). **Purpose enters only as an access *exception*** — to reach a normally-forbidden (out-of-role) tool, the principal **declares a purpose + justification** and a **human approves** (purpose-based **elevation**, §10 / §11.3). Memory retrieval stays **authorize-before-retrieve** on **visibility + lifecycle state** (PEP/PDP), never filtering after the LLM has seen data. | Matches the enterprise model: employees work via roles; special-purpose access is a human-approved, audited exception, not a continuous per-call predicate. The GDPR cornerstone (no unauthorized personal data in the candidate set) is kept via visibility + state. |
| D14 | Role vs purpose | **Distinct.** Role = *who you are / what you may do by default* (per-principal, from credential → the RBAC tool set covering usual tasks). Purpose = *the declared reason for an access exception* (per-request, to elevate to an out-of-role tool), **supplied by the human, never the LLM**. | Roles cover everyday work; purpose is the justification that unlocks an exception and is reviewed by a human. |
| D15 | Purpose declaration & elevation | To use a normally-forbidden tool, the principal **declares a purpose** (from a **governance-defined list** in the agent-pack) **+ free-text justification**; a **human approver** accepts/refuses; on approval the grant is **purpose-scoped and time-boxed**, then logged. **Never LLM-declared and never self-granted** — the principal declares, a human authority approves. | Mirrors how PBAC is run in practice (request access for a pre-approved purpose → approve → time-box → rescind). Keeps purpose unmanipulable: declaration ≠ authorization. |
| D16 | PEP/PDP placement | **App-layer PDP** for v2: one centralized query-builder all retrieval must route through. DB row-level security (RLS) noted as **v3 hardening**. | Pragmatic for v2; RLS makes bypass physically impossible but constrains schema/connection model. |
| D17 | Embeddings vs erasure | **Option A:** on crypto-erasure, the entry's embedding is **hard-deleted** (not crypto-erased). The semantic index sits **outside** the immutability/anchor chain (derived, rebuildable). No per-entry "embeddability" flag. | You can't "destroy a key" for a partially-invertible vector; encrypting vectors breaks ANN search. Hard-delete is the only approach compatible with real vector search. |
| D18 | Sensitive-data redaction | **Ingestion-time redaction/tokenization** of obviously-sensitive patterns is the correct layer for PAN-style data — flagged **open**, not in v2. | The right control is input-side, not a downstream embedding flag. Keeps the privacy model to its clean axes. |
| D19 | Consent / lawful basis | **Option A for v2:** operator-asserted lawful basis (supports consent, contract, legitimate-interest). No active consent management. A `lawfulBasis` field + a reserved predicate seam are included so **Option B (first-class consent) is a v3 extension** without rearchitecting. | Adequate for enterprise/internal deployments now; consent management is its own subsystem. |
| D20 | Crypto-erasure mechanism | Per-entry key; digest anchored over **ciphertext**; on accepted erasure, **destroy the key**; keep the anchor; mark `crypto-erased`. Non-personal metadata (visibility, purpose, timestamps) survives for audit. | Preserves tamper-evident auditability without touching the ledger while making plaintext unrecoverable. |
| D21 | Erasure lineage | An erasure cascades along a **lineage graph** to all derived artifacts: embeddings (hard-delete), working-memory notes citing the source (flag for LLM revision), window summaries (auto-regenerate clean), tool-call args/results, credential-derived claims, approval records. | Erasing only the raw message leaves the system still "remembering" via derivatives. |
| D22 | Languages | **Removed from core/memory.** Agent has **one configured language** (its own operation). **Presentation language** is an AG-UI frontend concern. Memory stores the conversation's actual language with a tag. | Per-language summaries/embeddings are unnecessary machinery; flow/menu localization belongs to the channel adapter. |
| D23 | Image generation | **Event-driven and non-blocking.** Respond quickly, generate in the background; **input is not blocked.** The generated artifact is persisted as a `MessageEntry` ordered by its **initiation timestamp**, so transcript ordering is preserved by the timestamp — not by holding input. A `generating` state may be exposed to the LLM so it can answer "still working on it" if asked. | Timestamped, ordered MessageEntries make the "lose the time-point" worry moot without blocking; blocking would fight the async, multi-channel design. |
| D24 | Media references | Store **bucket/key** reference + metadata; generate **presigned URLs on demand**. Never persist binaries or base64. AES key persisted alongside metadata (not in LLM context) for admin/audit decryption. | Handles URL expiration cleanly; keeps memory text-first. |
| D25 | Search backend | **pgvector embeddings from the start** (not ILIKE). **Hybrid:** short LLM-generated descriptions as compact ambient context + vector search for depth. | Semantic recall is core; ILIKE can't do it; building it later is rework. |
| D26 | Context window | **Token-based**, not message-count-based. | Rich messages have variable size; token budget is the real constraint. |
| D27 | Trust-layer failure modes | Verana anchoring: **async best-effort with retry queue, never blocks conversation.** Revocation-check unavailable: **fail-closed for gated actions, fail-open for ungated.** | Availability of conversation must not depend on ledger availability; security-sensitive actions must fail safe. |
| D28 | Schema versioning | **Version everything:** memory schema, agent-pack schema, system-prompt add-on. Migration notes required. | Old packs/data must survive upgrades. |
| D29 | Capability fallback | When a capability needs DIDComm but the Principal has no bindable capability channel: **configurable per-action, safe default `fail`** (alt: `degrade` — skip + log unmet requirement). | Security-sensitive default; flexibility where the operator accepts it. |
| D30 | Tenancy | **Single-deployment**, with **Organization / Persona as configuration**. Identity model is **multi-organization-aware** (federation-aware): it resolves and distinguishes the backing identity of every Principal, home or partner — but the agent hosts no partner's data; it only recognizes partner identities. Not multi-tenancy. | Matches deployment model; org/partner scopes are config + verified identity, not multi-tenant infrastructure. |
| D31 | Principal = two-axis identity | A Principal is **`(actorIdentity, backingIdentity)`**. `actorIdentity` = the specific actor (a service's ECS-Service, or a human's ECS-Badge subject). `backingIdentity` = the Org **or** Persona behind the actor. Two Principals can differ in actor while sharing backing identity (A1, A2 of Org A). Org and Persona are **peer kinds** — a Persona never belongs to an Org; both are independent backing identities, equivalent for scoping (Reading X). | Captures "different services/people, same organization." Uniform across humans and services. |
| D32 | Identity resolution paths | **Service** (`verifiable-service`): presents **ECS-Service + (ECS-Org or ECS-Persona)** → backing identity presented directly. **Human** (`verifiable-user-agent`): presents **ECS-Badge** → backing identity obtained by **trust-resolving the badge issuer's DID** to its ECS-Org/Persona. Symmetric: a badge is to a human what (ECS-Service + Org/Persona) is to a service. | One uniform `(actorIdentity, backingIdentity)` outcome via two paths. Badge trust rests on the issuer's verified backing identity, not bare DID resolution. |
| D33 | Badge = identity only, not authorization | The ECS-Badge carries **identity** (actor subject) + **informative attributes** — e.g. `userName`, `photo` (identity) and optionally `title`, `department`, etc. (organizational position). These are **verified facts the issuer asserts, never role assertions.** The badge carries no agent-local roles or purposes: it is **entirely the verifying party's (B1's) decision** whether to assign roles from these facts, via `roleRules` (§5.6). Claims earn their place only where a fact must cross a trust boundary. The `photo` attribute may serve as the **face-match reference** for auth challenges (§11.4). | Authorization is the relying party's decision over verified facts, not the connecting party's assertion (the D15 principle). "CEO at Org A" is a fact; "admin at B1" is B1's mapping choice. |
| D34 | Role source: agent-pack-native (v2) | Roles are **defined and assigned entirely within the agent-pack** (Option 1): the agent maps verified credential facts (badge claims — `title`, `department`, `backingIdentity` relation, …) → agent-local roles via `roleRules` (§5.6). **No external runtime dependency.** **External IAM integration (Option 2 — runtime IAM as the role-assignment authority via OIDC/SAML/SCIM) is deferred to v3.** | Self-contained v2 with no dependency on IAM availability. The credential is the source of identity facts; the agent-pack rules are the source of role assignment. IAM integration is its own design, better done separately. |
| D35 | Role assignment: rules, not rosters | Roles are assigned by **B1-side rules over verified identity facts** (`backingIdentity` relation home/partner + selected claims), never by enumerating identities. Default: **home** backing identity → richer mapping; **recognized partner** → safe baseline (`partner`) unless a rule elevates on claims; **unrecognized but valid** → authenticated, **no** backing identity, `public`-scoped only. Per-actor overrides allowed where B1 must single someone out (layered default + override). | Static per-identity config doesn't scale; rules over claims do. Never trust party-asserted privilege by default. |

---

## 2. Terminology

| Term | Definition |
|------|-----------|
| **Principal** | An authenticated entity interacting with the agent — a human, a bot, or another agent. Identity is established by the verifiable credential(s) presented at authentication, **not** by the DID itself, and is modeled as the pair **`(actorIdentity, backingIdentity)`** (see below). A Principal is the durable identity that conversation sessions and the capability channel attach to. |
| **Actor identity** | The specific actor behind a Principal: a service's **ECS-Service** identity, or a human's **ECS-Badge** subject. The first axis of Principal identity. |
| **Backing identity** | The Organization **or** Persona standing behind the actor — the second axis of Principal identity. Org and Persona are peer kinds (a Persona never belongs to an Org); equivalent for scoping purposes. Two Principals can share a backing identity while differing in actor (services A1, A2 of Org A; employees E1, E2 of Org A). |
| **ECS-Badge** | A credential held by a human, **issued by a Verifiable Service** that itself presents an ECS-Org or ECS-Persona credential. Carries the human's actor identity + identity attributes (userName, photo, optionally department/group). The human's backing identity is obtained by **trust-resolving the badge issuer's DID**. The badge is the human analog of a service's (ECS-Service + Org/Persona). |
| **Trust-resolve** | (Verana terminology.) Resolving a DID **and** verifying the entity's credential trust chain — not a bare DID-document lookup. Badge-derived backing identity requires trust-resolving the issuer. |
| **Connection type** | How a Principal connects. `verifiable-service` — a public resolvable DID representing a service/agent/bot; presents ECS-Service + (ECS-Org or ECS-Persona). `verifiable-user-agent` — a user agent (mobile wallet), typically `did:peer:`; presents an ECS-Badge over DIDComm. |
| **Relying party** | The role the agent plays with respect to identity: it does **not** define or own an identity directory. It establishes identity from presented credentials and maps the **verified facts** they carry to agent-local roles via configured rules (§5.6). (In v3 it may additionally rely on an external IAM at runtime — D34/O8.) |
| **Channel** | A transport adapter into the channel-agnostic core. Three exist: **DIDComm**, **AG-UI**, **A2A**. Each translates transport-specific messages ↔ canonical `MessageEntry` and renders core capability events in its own idiom (or declares it cannot). |
| **Capability channel** | The channel used for trust-bound operations (authentication, credential exchange, auth challenges). Always **DIDComm**. One active per Principal. |
| **Conversation session** | A thread of interaction on a channel (`sessionId`). A Principal may hold several concurrently (e.g. multiple AG-UI screens). Each has its own conversation history. |
| **Role** | An RBAC permission set held by a Principal (`user`, `admin`, `operator`, `auditor`, …). Per-principal. **Assigned by the agent (relying party) via rules over verified identity facts**, not self-asserted. Answers *what may this principal do/see*. Fully configurable; unlimited custom roles. |
| **Purpose** | The declared reason a principal needs an **access exception** (`finance-support`, `fraud-audit`, `it-support`, …), chosen from the agent-pack's governance-defined list. **Human-declared, human-approved** (D15); never LLM-chosen. Recorded for audit; enforces purpose-limitation at elevation (§10.1). |
| **Visibility** | Axis A of privacy classification — who may access an entry: `session-private` / `principal-private` / `organization-shared` / `partner-shared` / `public`. The org/partner tiers compare **backing identity**. |
| **Lifecycle state** | Axis B of privacy classification — legal/retention status: `active` / `archived` / `legal-hold` / `crypto-erased` / `deleted`. |
| **Agent-pack** | The configuration bundle defining agent behavior: system prompt, language, enabled tools (role-gated, with optional elevation policy), role-assignment rules, recognized backing identities (home/partner), governance-defined purposes, approval & elevation policies, auth-challenge requirements, retention, and feature gates. Authored by the agent administrator. |
| **Working memory** | LLM-curated, durable, principal-scoped notes written via tool call. Authoritative. The principal cannot read or edit it. |
| **MessageEntry** | The canonical, transport-independent unit of memory. Rich and typed (see §8, §14). |
| **Digest** | A SHA-256 hash used for integrity. Applied to external referenced objects and to batches of history entries anchored to the Verana ledger. |
| **Crypto-erasure** | Making plaintext unrecoverable by destroying its per-entry key while retaining ciphertext + digest anchor, so auditability survives erasure. |
| **PEP / PDP** | Policy Enforcement Point / Policy Decision Point — the PBAC split between *deciding* a scope and *applying* it. |

---

## 3. Goals & Non-Goals

### Goals

1. **Channel-agnostic core** — the agent's logic deals only in Principals, Sessions, MessageEntries, and capability events; transports are interchangeable adapters.
2. **Unified trust model** — one coherent way to establish identity and route trust-bound capabilities across DIDComm, AG-UI, and A2A.
3. **Rich, continuous memory** — the LLM can reference prior tool calls, generated/received media, sent messages, reactions, and decisions across turns and (when authorized) across sessions.
4. **Text-first, never binary** — prompts over pixels, transcriptions over audio, references over blobs.
5. **Provable integrity** — historized data is tamper-evident via Verana-anchored digests covering messages and their external references.
6. **GDPR by construction** — purpose-limited, minimized, access-controlled-before-retrieval, erasable (including derived data), auditable.
7. **Configurable governance** — roles, purposes, approval policies, auth challenges, and retention are all agent-pack configuration, not code.
8. **Clean over cautious** — prefer clear, minimal structure. This is a from-scratch spec; nothing in the current implementation is assumed to carry forward.

### Non-Goals (v2)

- First-class consent management (operator-asserted lawful basis only; consent is v3 — D19).
- Ingestion-time PII redaction subsystem (flagged open — D18).
- DB row-level security enforcement (app-layer PDP for v2; RLS is v3 — D16).
- Multi-tenancy (single **deployment**; the identity model is federation-*aware* — it recognizes partner backing identities — but hosts no partner's data and runs no partner's tenant — D30).
- Mobile-native AG-UI client SDK (the AG-UI channel is specified; a native client is downstream).
- Defining the Verifiable Trust verification procedure itself (out of scope; referenced as a dependency).

---

## 4. Architecture Overview

```
        ┌────────────┐   ┌────────────┐   ┌────────────┐
        │  DIDComm   │   │   AG-UI    │   │    A2A     │     Channel adapters
        │  adapter   │   │  adapter   │   │  adapter   │     (transport ↔ canonical)
        └─────┬──────┘   └─────┬──────┘   └─────┬──────┘
              │                │                │
              └────────────────┼────────────────┘
                               ▼
        ┌──────────────────────────────────────────────┐
        │              Channel-Agnostic Core            │
        │  • Principal & session resolution             │
        │  • Capability router (→ capability channel)   │
        │  • Conversation orchestration (LLM turn)      │
        │  • Tool invocation + purpose binding          │
        └───────────────┬──────────────────┬───────────┘
                        │                  │
              ┌─────────▼────────┐  ┌──────▼───────────┐
              │  Memory subsystem │  │  Policy subsystem │
              │  (§8)             │  │  PBAC PEP/PDP (§9)│
              └─────────┬────────┘  └──────┬───────────┘
                        │                  │
        ┌───────────────▼──────────────────▼───────────┐
        │            Persistence & Integrity            │
        │  Long-term store · Working memory · Semantic  │
        │  index · Media refs · Digest anchors (Verana) │
        └───────────────────────────────────────────────┘
```

**Reading the diagram:** transports never reach persistence directly. Every inbound message becomes
a canonical `MessageEntry`; every retrieval passes the PBAC PEP/PDP; every capability event is routed
to the channel that can serve it. The core has no knowledge of SSE, DIDComm message types, or A2A
tasks — only of Principals, Sessions, MessageEntries, and capability events.

---

## 5. Identity, Connection & Trust

### 5.1 The Principal is the durable identity

Sessions and channels are transient; the Principal is the join key for everything durable —
working memory, preferences, credentials, cross-session access. Two sessions are "the same person"
because they resolve to the same Principal, never because they share a `connectionId`.

### 5.2 Principal identity: two axes (D31)

A Principal is **not flat**. It is the pair:

```
Principal identity =
  actorIdentity     — the specific actor   (a service, or a specific human)
  backingIdentity   — the Org OR Persona standing behind that actor
```

Two Principals can share a `backingIdentity` while differing in `actorIdentity`. This is the load-
bearing fact for cross-org scoping:

```
Org A
 ├── Service A1 ──┐                          From B1's perspective:
 ├── Service A2 ──┤                            A1, A2, E1, E2 are FOUR distinct
 ├── Human  E1 ───┼──→ connect to ──→ B1         Principals (distinct actorIdentity)
 └── Human  E2 ───┘   (service, Org B)         but SHARE one backingIdentity = Org A
```

**Org and Persona are peer kinds (D31).** A Persona never belongs to an Org; both are independent
backing identities and are **equivalent for scoping** (Reading X). The distinction is informational
(who is *responsible* for the actor — an organization vs a persona) and may matter in policy, but the
visibility comparison is a single equality on `backingIdentity` regardless of kind.

### 5.3 Identity-resolution paths — symmetric (D32)

Resolution is connection-type-specific but produces the same uniform `(actorIdentity, backingIdentity)`
pair. The two paths are **symmetric**: a badge is to a human what (ECS-Service + Org/Persona) is to a
service.

| Connection type | Presents | `actorIdentity` | `backingIdentity` |
|-----------------|----------|-----------------|--------------------|
| `verifiable-service` | **ECS-Service** + (**ECS-Org** or **ECS-Persona**) | ECS-Service | the Org/Persona **presented directly** |
| `verifiable-user-agent` (human) | **ECS-Badge** | the badge subject | the Org/Persona obtained by **trust-resolving the badge issuer's DID** |

**Badge trust rule (D32):** a human's backing identity is only as trustworthy as the badge *issuer*.
Resolution must **trust-resolve** the issuer DID — resolve it *and* verify the issuer's own
ECS-Org/Persona credential chain — not merely read the DID document. Otherwise anyone could
self-issue a badge claiming any issuer. Trust rests on the issuer's verified backing identity.

> Initial connection verification follows the
> [Verifiable Trust spec](https://verana-labs.github.io/verifiable-trust-spec/) — consumed as a
> dependency, not re-specified here.

### 5.4 The ECS-Badge carries identity, not authorization (D33)

The badge provides **identity** (actor subject) and **informative attributes**:

- **Identity:** `userName`, `photo`.
- **Organizational position (optional):** `title` (e.g. CEO, Janitor), `department` (e.g. Finance), etc.

These are **verified facts the issuer asserts** — *never* role assertions. The badge carries no
agent-local roles or purposes. Whether `title`, `department`, or any other fact translates into a B1
role is **entirely B1's decision**, made by `roleRules` over these facts (§5.6). "CEO at Org A" is a
fact the issuer vouches for; "admin at B1" is a mapping B1 may or may not choose to make.

- **Claims earn their place only when they must cross a trust boundary.** `title`/`department` belong
  in a badge precisely when an *external* relying party's access rules may depend on intra-org
  structure (a portable, verifiable "I'm in Finance at Org A"). For a home-only deployment that never
  keys access on such facts, they are dead weight and may be omitted.
- **The `photo` attribute may serve as the face-match reference** for auth challenges (§11.4) — the
  same credential that establishes identity provides the reference for later re-verifying it.

### 5.5 The agent is a relying party; roles are agent-pack-native in v2 (D34)

The agent is a **relying party**: it does not own an identity directory. It establishes *identity*
from presented credentials and derives *roles* from the verified facts those credentials carry. In v2
this is **fully self-contained in the agent-pack** (Option 1) — no external system is contacted at
runtime:

```
credential presented (badge / service creds)
   ↓ agent trust-resolves issuer, reads verified facts (title, department, backingIdentity, …)
   ↓ agent maps facts → agent-local roles via roleRules in the agent-pack (§5.6)
```

Credentials behave like a point-in-time **snapshot** of the issuer's view (the same "claims-as-snapshot"
model OIDC ID tokens / SAML assertions use). **Freshness** is maintained not by short lifetimes or any
live re-fetch, but by **issuer-initiated revoke/reissue over the issuer↔holder DIDComm channel**: when
the issuer's view changes, it revokes the old credential and/or pushes a fresh one to the holder. The
agent never tracks reissuance — it only validates the credential presented to it (§5.7). Services are
analogous: their ECS-Service + Org/Persona credentials are the snapshot.

> **External IAM integration is v3 (D34, O8).** Where an organization wants its existing IAM
> (Okta/Entra/Keycloak/…) to be the runtime authority for role assignment — via OIDC/SAML/SCIM rather
> than agent-pack rules over credential facts — that is a deliberate v3 capability with a different
> availability profile (a live dependency on the IAM). v2 deliberately avoids that runtime coupling.

### 5.6 Role assignment: rules over verified facts, not rosters (D35)

Authorization is the **relying party's decision over verified identity facts** — never the connecting
party's self-assertion (the D15 principle, applied to roles). To stay flexible, B1 assigns roles by
**rules**, not by enumerating identities:

```yaml
# agent-pack (illustrative)
recognizedBackingIdentities:
  - identity: did:…:orgB        # the agent's own org
    relation: home
  - identity: did:…:orgA        # a recognized partner
    relation: partner

roleRules:                       # evaluated over verified facts; first match wins
  - when: { relation: home }                       → rolesFrom: badgeFacts   # trust own org's facts
  - when: { relation: partner }                    → role: partner           # safe baseline
  - when: { relation: partner, claim.tier: gold }  → role: partner-plus      # claim-based elevation
  # no match (unrecognized but valid credential)   → authenticated, no backing identity, public-only
perActorOverrides:               # optional: single out a specific known actor
  - actor: did:…:someExternalCollaborator          → role: operator
```

Defaults (D35): **home** backing identity → its facts may map directly (B1 trusts its own org's role
assignments); **recognized partner** → safe baseline `partner` unless a rule elevates on a verified
claim; **unrecognized but valid** → authenticated but **no backing identity**, `public`-scoped only,
never org/partner-shared. Per-actor overrides layer on top for specific known individuals. Services and
humans use the **same** rule mechanism keyed on `backingIdentity`.

> **Recognition for visibility is independent of role privilege.** A recognized partner is known for
> `partner-shared` *visibility* scoping (§9) yet may hold only a baseline role. The two axes
> (visibility/purpose vs role) stay orthogonal.

### 5.7 Credential-liveness lifecycle (D7, D8)

Every active session rests on a credential; the agent enforces its continued validity by a
**TTL-bounded revalidation**, uniform across humans and services:

```
on session activity (action or turn):
  if (now − lastChecked) > N seconds:
      re-verify the underlying credential:
        • human   → ECS-Badge revocation + re-walk issuer trust chain
        • service → ECS-Service + ECS-Org/Persona revocation/expiry + chain
      lastChecked = now
      if no longer trustable (revoked / expired / issuer untrustable):
          terminate session → entity must re-present a valid credential
  else:
      proceed (cached validity still fresh)
```

- **One mechanism, both effects.** Activity *triggers* checks (no waste on idle sessions); the TTL `N`
  *bounds* maximum validity-staleness for an active session and **self-throttles** bursts (≤ 1 check
  per `N` seconds per session). A revocation check is a cheap remote GET, so `N` can be small.
- **The check re-walks the issuer chain**, not just the leaf: if Org A itself loses trust, every
  A-badged human's session fails the check. Same for services and their backing identity.
- **Idle-session timeout (D7 #4):** a separate, independent timeout reaps quiet sessions so none
  lingers open-and-stale; the user simply re-authenticates on return. Covers the one gap
  activity-triggered checking leaves (a never-returning idle session).
- **Reissuance is the issuer's concern, invisible to the agent.** The agent validates only the
  credential presented to it; whatever the Org does to revoke/replace it travels the Org↔holder
  DIDComm channel and reaches the agent only when the holder next presents a (fresh) credential.
- `N` (max-staleness-while-active) and `idleTimeout` are configurable; `N` directly sets the
  worst-case post-revocation exposure window for an active session.

### 5.8 Tenancy & federation-awareness (D30)

**Single deployment.** **Organization** and **Persona** are configuration values, not infrastructure
boundaries. The identity model is nonetheless **multi-organization-aware**: it resolves and
distinguishes the `backingIdentity` of every Principal — home or partner — so that `organization-shared`
and `partner-shared` (§9) can be enforced. This is *federation-awareness*, **not** multi-tenancy: the
agent recognizes partner identities but hosts no partner's data and runs no partner's tenant.

### 5.9 DIDComm as the trust & capability anchor (D2)

DIDComm is the **most capable** channel: it can carry ordinary conversation like any other channel
(e.g. the Hologram-browser-only case, where DIDComm is the *sole* channel), **and** it is the only
channel that can serve trust-bound operations. So trust establishment and trust-bound capabilities are
**always** routed to DIDComm — not because other channels are barred from conversation, but because
only DIDComm has a wallet, credential store, and device sensors. The three capabilities no other
channel can serve:

1. **Initial authentication & bootstrap** — establishes the Principal.
2. **Credential exchange** — VC request/presentation (A2A has no concept of this; AG-UI has no wallet).
3. **Auth challenges** — biometric, face-match-with-liveness, NFC document read (require a sensored device).

Message routing is otherwise the agent's decision via the capability router (§6.2), bounded only by
each channel's capability profile (§6.5): DIDComm is *eligible* for everything, *required* only for the
three above, and *not mandatory* for ordinary conversation when another conversation channel is active.

---

## 6. Channel Layer

### 6.1 Channels as first-class peers (D1)

No channel is privileged for *transport*. Each adapter has exactly two responsibilities:

- **Inbound:** transport-specific message → canonical `MessageEntry`.
- **Outbound:** canonical agent action / capability event → transport-specific rendering, **or** an explicit declaration that the channel cannot render it.

### 6.2 Capability routing (D2, D3)

The conversation channel and the capability channel are **separable**. When the core emits a
capability event, the **capability router** asks "which bound channel can serve this?" and routes
accordingly.

| Capability | Preferred channel | If conversation is on AG-UI / A2A |
|-----------|-------------------|-----------------------------------|
| Text / menus / conversation | the active conversation channel | served in place |
| Credential exchange | **DIDComm only** | routed to the Principal's bound DIDComm channel |
| Auth challenge (biometric / NFC / face-match) | **DIDComm only** | routed to the Principal's bound DIDComm channel |
| Approval (non-sensor) | any channel | served in place, or notified cross-channel |
| Reaction / feedback signal | channel-native if supported | omitted if the channel has no equivalent |

Auth challenges are **Principal-scoped but carry `originSessionId`** (D3): a challenge triggered by
the laptop AG-UI session goes to the mobile wallet, and the result routes back to the laptop session —
not broadcast to the Principal's other sessions.

### 6.3 Capability-channel cardinality (D4)

- **A2A → 1:1.** One A2A conversation, one DIDComm channel, torn down together. No sharing (machine peers gain nothing from multiplexing).
- **AG-UI → shared.** One DIDComm capability channel per Principal, shared across all that Principal's AG-UI sessions. Matches the one-wallet-many-screens reality.

Unifying principle: *the capability channel is bound to the trust-bearing device, not the session.*
For A2A the device is the peer; for humans it is the singular wallet — same principle, different
cardinality.

### 6.4 Bootstrap & binding flows (D5, D6)

All flows converge on one end-state: **a Principal with a bound DIDComm capability channel + a
conversation channel.** Two patterns:

**DIDComm-first ("bootstrap")** — authenticate over DIDComm, then open the conversation channel
carrying proof of the established Principal. Natural for A2A and same-device AG-UI.

```
DIDComm authenticate (VT / VC)  →  Principal resolved  →  open A2A / AG-UI bound to Principal
```

**Channel-first ("bind")** — open a *provisional* conversation session, then trigger a DIDComm
authentication that retroactively binds it. Required for cross-device AG-UI (QR scan).

```
open AG-UI (provisional, unbound)  →  render binding affordance (QR + nonce)
   →  wallet scans, authenticates over DIDComm, echoes nonce
   →  core matches nonce → binds session to Principal
```

**Invariants:**

- **Unbound provisional session** (D6): no Principal, no roles, **zero access to memory, tools, or cross-session data.** It may only advance binding (render QR, poll status). It is an attack surface and must touch nothing privacy-scoped.
- **Binding nonce** (D6): the provisional session generates a one-time nonce embedded in the QR; the wallet echoes it over DIDComm; the core matches them. Prevents a leaked/photographed QR from binding an attacker's session.
- **AG-UI participates in the binding handshake** — it is not purely a conversation channel; it must render the auth-binding affordance and await the result.
- **Reusing an existing capability channel:** opening a *second* AG-UI session while the Principal's DIDComm channel is already live still generates a per-session binding nonce, but the binding can be satisfied by a **lightweight confirmation** on the existing channel — not a full re-authentication. New conversation session = new nonce; binding served by the live channel.

### 6.5 Channel capability profiles

Each adapter declares which capabilities it can render. The core uses these profiles plus the
capability router to decide routing and fallback (D29). Example sketch:

| Capability | DIDComm | AG-UI | A2A |
|-----------|:-------:|:-----:|:---:|
| Text | ✓ | ✓ | ✓ |
| Rich menu | ✓ | ✓ | ✓ (structured) |
| Credential exchange | ✓ | ✗ | ✗ |
| Auth challenge | ✓ | ✗ | ✗ |
| Reaction signal | ✓ | ✓ | ✗ |
| Media (ref) | ✓ | ✓ | ✓ |

When a required capability has no servable channel for the Principal, fallback is **configurable
per-action with default `fail`** (D29).

---

## 7. Session Lifecycle

Four termination triggers, cleanly separated by what they act on (D7, D8):

| Trigger | Acts on | Effect |
|---------|---------|--------|
| **DIDComm channel closes** | channel | Its dependent AG-UI sessions close. |
| **New DIDComm channel supersedes** existing one for same Principal | channel | Old channel + its AG-UI sessions close. (The lost-phone path: recover → new channel → old sessions gone.) |
| **Underlying credential fails re-verification** (revoked / expired / issuer untrustable) | credential / identity | That session terminates; if the failing credential is the Principal's identity credential, **all** its sessions close. Entity must re-present a valid credential. |
| **Idle timeout** | hygiene | A session quiet beyond `idleTimeout` is reaped; the user simply re-authenticates on return. |

**Critical distinction (D7):** "a new DIDComm connection" means a new *capability-channel
establishment*, **not** a new conversation session binding to the existing channel. Opening a tablet
tab reuses the live channel (lightweight confirm) and leaves the laptop untouched. Recovering on a
new phone establishes a genuinely new channel and supersedes the old one.

**Credential-liveness enforcement — TTL-bounded revalidation (D8):** a single mechanism subsumes
"periodic" and "per-action" checking. Detailed in §5.7; in summary:

- On session **activity**, if cached validity is older than `N` seconds, re-verify the underlying
  credential (revocation GET + **re-walk the issuer trust chain**) and terminate on failure. Activity
  triggers the check (no waste on idle sessions); the TTL bounds staleness and **self-throttles**
  bursts to ≤ 1 check per `N` seconds per session.
- An independent **idle-session timeout** reaps quiet sessions, closing the one gap activity-triggered
  checking leaves (a never-returning idle session holding a since-revoked credential).
- Uniform across humans (badge + issuer chain) and services (ECS-Service + backing identity).
- **Reissuance is invisible to the agent** — it validates only the credential presented; replacement
  travels the issuer↔holder DIDComm channel and reaches the agent at the holder's next presentation.
- `N` (max-staleness-while-active, the worst-case post-revocation exposure window) and `idleTimeout`
  are configurable.

---

## 8. Memory Model

### 8.1 The canonical MessageEntry

Replaces the legacy flat `{ role, content: string }`. A rich, typed, transport-independent unit
(full shape in §14). It carries: role, the text representation, typed `attachments`, `sourceType`
(`text` / `voice` / `media` / `tool-call` / `approval` / `refusal` / `auth-challenge` / `reaction` /
`state-update`), privacy classification (§9), language tag (D22), and integrity digests (§12).

### 8.2 Text-first attachment rules (D24)

Never persist binaries or base64. Persist the text representation + reference + integrity material:

| Media type | Persisted | Never persisted |
|-----------|-----------|-----------------|
| Generated image | prompt, provider, bucket/key, refId, dimensions, AES key, digest | image binary / base64 |
| Principal-sent image | bucket/key, mimeType, brief description (D25), AES key, digest | image binary |
| Voice note | STT transcription, `sourceType:voice`, AES key, digest | audio binary |
| Video | bucket/key, mimeType, duration, AES key, digest | video binary |
| File | bucket/key, filename, mimeType, size, AES key, digest | file contents |
| Credential presentation | credentialId, type, issuer DID, summary, digest | full presentation JSON |

- **AES key** is persisted alongside metadata (for admin/audit decryption per §10) — **never** placed in the LLM context window.
- **Media references** are stored as bucket/key; **presigned URLs are generated on demand** (D24), solving expiration.
- **Image descriptions** are generated once and stored (D25) — not recomputed each time the image is referenced.

### 8.3 Memory tiers (D10, D11)

```
AUTHORITATIVE
├─ Long-term store .......... every MessageEntry, ever. The single source of truth.
└─ Working memory ........... LLM-curated durable notes. Principal-scoped. Principal cannot edit.

DERIVED (rebuildable; never holds data not traceable to an authoritative source)
├─ Active context window .... token-based slice of recent entries; rebuilt each turn.
├─ Semantic index ........... pgvector embeddings; pure retrieval accelerator.
└─ Window summaries ......... transient compression of old turns *within the window only*;
                              never persisted as an independent record; regenerated on demand.
```

- **Working memory vs summarization (D11):** working memory is *active, curated, durable* (the LLM decides "this matters" and writes it). Summarization is *passive, mechanical, throwaway* (the system compresses an overflowing window). A fact that matters lands in working memory by LLM decision; a fact that doesn't simply ages out of the window and survives only in the raw long-term store. They never both hold the same fact authoritatively.
- **Principal cannot edit working memory** — a hard security rule. Working memory is the LLM's curated model of the principal, not a principal-writable field (which would be a manipulation surface).

### 8.4 Window & retrieval (D25, D26)

- **Token-based window** (D26): the active context is bounded by token budget, not message count, because rich entries vary in size.
- **pgvector from the start** (D25): semantic retrieval is core; exact/`ILIKE` search cannot satisfy it.
- **Hybrid retrieval** (D25): short stored descriptions act as compact ambient context; vector search provides depth. Long text is chunked for embedding/retrieval.
- **All retrieval is privacy-scoped** (§9): there is no global semantic search. Every query — including vector search — runs *within an already-authorized subset* scoped by **visibility + lifecycle** (D13).

---

## 9. Privacy & GDPR Architecture (PBAC)

### 9.1 Prior art & model basis

This architecture is not bespoke invention. It applies **Purpose-Based Access Control (PBAC)** — an
established model that adds a contextual *purpose* layer on top of RBAC — in the form most common in
practice: **purpose-declared access elevation** (request access for a pre-approved purpose → a human
approves → the grant is time-boxed → it is rescinded). The design borrows three constructs from the
PBAC / XACML literature:

- **PBAC** — access decisions consider not just *who* (role) but *why* (the declared purpose of the request). Recognized as the evolution beyond RBAC for privacy-regulated data (e.g. GDPR Art. 5(1)(b) purpose limitation).
- **PEP / PDP split** — a Policy Decision Point computes the authorized scope; a Policy Enforcement Point applies it. We adopt this split for memory retrieval (§9.5) and tool gating (§10).
- **Purpose hierarchies & purpose-matching** — purposes form a hierarchy; a matching algorithm decides whether a request's declared purpose satisfies a permitted purpose. This grounds the governance-defined purpose list and the optional role → permitted-purposes check applied at elevation (D15).

> **Scope note.** v2 realizes purpose as a *human-declared, human-approved elevation* (D15) — the way
> PBAC is run in industry (e.g. Immuta projects: pick a pre-approved purpose, get approval, time-box,
> rescind). Applying purpose as a *continuous predicate* that scopes every agent-memory retrieval is a
> stronger, more novel form that v2 does **not** mandate: memory is scoped by **visibility +
> lifecycle** (§9.2), with purpose recorded for audit. Continuous per-purpose memory minimization is
> deferred (§15, O9).

### 9.2 Two axes + a tag (D12)

Every memory entry carries three independent classifications:

- **Visibility (Axis A — who may access):** `session-private` · `principal-private` · `organization-shared` · `partner-shared` · `public`
- **Lifecycle state (Axis B — legal/retention status):** `active` · `archived` · `legal-hold` · `crypto-erased` · `deleted`
- **Purpose (tag — why collected):** e.g. `it-support`, `finance-support`, `fraud-audit`, `communication-preference`

The cross-product is the point. Combinations a flat enum can't express but that genuinely occur:
*organization-shared + legal-hold* (team-visible but undeletable under litigation),
*principal-private + crypto-erased* (erased preference whose metadata survives for audit).

**The org/partner visibility tiers compare `backingIdentity` (D12, D31):**

- `organization-shared` → visible to Principals whose `backingIdentity` **equals the agent's own**
  (home) backing identity — across all their actors (the agent's own services *and* its badged
  humans).
- `partner-shared` → visible to Principals whose `backingIdentity` equals a **recognized partner**
  backing identity — across all that partner's actors.

Both are **one comparison** — equality on `backingIdentity` — differing only by whether the matched
identity is home or a recognized partner (§5.6). They share a single enforcement path.

**Worked example (the A1/A2/E1/E2 case).** Agent B1 is a service backed by Org B. Org A operates
services A1, A2 and employs humans E1, E2 (each holding an A-issued ECS-Badge). All four connect to
B1 — A1/A2 over A2A, E1/E2 over DIDComm:

```
Principal   actorIdentity   backingIdentity   connection
A1          ECS-Service A1   Org A            verifiable-service
A2          ECS-Service A2   Org A            verifiable-service
E1          badge subject    Org A            verifiable-user-agent
E2          badge subject    Org A            verifiable-user-agent
```

An entry B1 marks `partner-shared` for Org A is visible to **all four** — they share
`backingIdentity = Org A` — regardless of whether they connected as services or humans. It is **not**
visible to a Principal backed by Org C. A `session-private` entry from E1's session stays with E1;
a `principal-private` entry follows E1 across E1's own sessions only. This is the channel-agnostic,
identity-based result: visibility keys on *who you are backed by*, never on *how you connected*.

### 9.3 Role vs purpose (D14)

| | Role | Purpose |
|--|------|---------|
| Question | *what you may do by default* | *why you need an exception now* |
| Bound to | the **Principal** (stable for session) | a **specific elevation request** |
| Source | credential claims → RBAC tool set | **declared by the human** + approved |
| Effect | unlocks the everyday tool set | unlocks one out-of-role tool, time-boxed |
| Example | `operator` → its usual tools | `fraud-audit` → temporary `read_customer_record` |

Roles cover usual work; **purpose is the declared reason that unlocks an exception** (§10 / §11.3) and
is reviewed by a human. Memory retrieval itself is scoped by **visibility + lifecycle state**:
`visibility-allowed(role) AND state = 'active' …` — purpose is **not** a continuous predicate term
(it is recorded for audit, and an active elevation grant widens the reachable tools, §11.3).

### 9.4 Purpose declaration & elevation (D15)

Purpose is **not** inferred by the system or the LLM — it is **declared by the human** when they need
an out-of-role tool, and **approved by a human**, so it can never become a social-engineering surface:

```
elevation request =
  1. principal hits a normally-forbidden (out-of-role) tool
  2. principal DECLARES a purpose (from the governance-defined list in the agent-pack)
     + a free-text JUSTIFICATION                       ← the human supplies both
  3. (optional) the declared purpose must be within the role → permitted-purposes set
  4. a human APPROVER accepts / refuses
  5. on accept → a purpose-scoped, time-boxed GRANT; the tool becomes reachable
     for the grant window; every use is logged with {purpose, justification, approver}
```

The governance-defined purpose list and the optional role → permitted-purposes set live in the
agent-pack (same config surface as RBAC), so purpose authorization does not require minting purposes
into credentials. The LLM may *detect* an unmet need and *draft* the request, but the principal must
submit it and a human must approve it — **declaration is not authorization** (§11.3).

### 9.5 Access-before-retrieval (D13, D16)

The GDPR cornerstone: **authorize the scope first, then search within it.** Unauthorized personal
data must never enter the candidate set — not be filtered out after the LLM (or even the ranker) has
seen it.

```
FORBIDDEN (filter-after):  search all → rank → drop unauthorized → show LLM
REQUIRED  (authorize-then-search):
   1. PDP computes scope predicate from (principal, role, visibility, backingIdentity, state)
   2. PEP applies it as a mandatory query filter
   3. vector/exact search runs ONLY within that predicate
   4. results are authorized by construction → shown to LLM
```

The org/partner terms of the predicate compare **backing identity** (§9.2), e.g.:

```
visibility = 'public'
OR (visibility = 'session-private'      AND entry.sessionId  = ctx.sessionId)
OR (visibility = 'principal-private'    AND entry.principalId = ctx.principalId)
OR (visibility = 'organization-shared'  AND entry.backingIdentity = ctx.homeBackingIdentity)
OR (visibility = 'partner-shared'       AND entry.backingIdentity = ctx.principalBackingIdentity
                                        AND ctx.principalBackingIdentity IN recognizedPartners)
```
…always AND-ed with `state = 'active'`. (Purpose is **not** a continuous predicate term — §9.4 / D15; an active elevation grant instead widens the reachable *tools*, §11.3.)

- **PEP/PDP placement (D16):** app-layer for v2 — the scope predicate is built in **one centralized
  query-builder that all retrieval must route through**. There is no code path that searches globally
  then drops rows. **RLS at the DB boundary is noted as v3 hardening.**
- **Data-model consequence:** every dimension the predicate filters on (`visibility`, `state`,
  `backingIdentity`, `principalId`, `sessionId`) must be an **indexed column**, decidable in
  the query — not computed at read time, not buried in a JSON blob. (`purpose` is stored as an audit
  annotation, not a retrieval-predicate term — §9.4.)

### 9.6 Data minimization

Retrieval returns the **minimum needed** for the current task: scoped by tenant/org/principal/role/
visibility/state; summaries preferred over raw where raw isn't necessary; sensitive data
exposed to the LLM only on a need-to-know basis. Embeddings are treated as personal data subject to
the full lifecycle (§9.7).

### 9.7 Erasure: crypto-erasure + lineage (D17, D20, D21)

**Crypto-erasure (D20)** for authoritative entries: each sensitive entry is encrypted with its own
key; the digest is anchored over the **ciphertext**; on an accepted erasure request the **key is
destroyed**, the anchor is kept, and the entry is marked `crypto-erased`. Plaintext becomes
unrecoverable while the Verana chain stays valid and untouched. Non-personal metadata (visibility,
purpose, timestamps) survives for audit ("an entry of this purpose existed and was erased on date X").

**Erasure is blocked by `legal-hold`** — the lifecycle axis overrides an erasure request where legal
retention applies (the two axes operating independently, per §9.2).

**Lineage cascade (D21):** erasure follows a dependency graph to all derived artifacts so the system
doesn't "remember" via derivatives:

| Derived artifact | Action on source erasure |
|------------------|--------------------------|
| Embedding (semantic index) | **hard-delete** (D17 — can't key-destroy a partially-invertible vector; index is outside the anchor chain) |
| Working-memory note citing the source | flag for LLM revision / removal |
| Window summary | auto-regenerates clean (transient, never authoritative) |
| Tool-call args/results containing the data | crypto-erased with the entry |
| Credential-derived claims | re-evaluated / removed |
| Approval records containing personal data | crypto-erased, audit metadata retained |

**Embeddings are explicitly outside the immutability chain (D17):** they are derived and rebuildable,
so they are hard-deleted on erasure rather than crypto-erased. The semantic index is never anchored.

### 9.8 Legal hold & retention (D27 anchoring; issue defaults)

Distinct data states: `active` · `archived` · `legal-hold` · `crypto-erased` · `deleted` + immutable
digest anchors. Defaults (configurable per deployment):

| Tier | Default retention | Notes |
|------|-------------------|-------|
| Long-term store (session histories) | 1 year | After expiry, **soft-deleted (archived)**, not hard-deleted — original must remain so the digest chain stays verifiable. Excluded from active queries & LLM context; still re-hashable by an auditor. |
| Media objects | 1 year (aligned) | Objects referenced by non-expired (incl. archived) entries must not be deleted. Truly orphaned objects may be GC'd earlier. |
| Credential presentations | indefinite | Tied to Principal identity, not session lifecycle. |
| Digest anchors | indefinite | Immutable on-ledger; local anchor records kept forever for verification. |

### 9.9 Memory-access audit log

Every authorized retrieval writes an access-log record, enabling "why did the agent know X?":

```
who (principal + role) · purpose · scope predicate · matched entry ids ·
from which sessions · shown_to_llm? · influenced_tool_call? · timestamp
```

### 9.10 Consent / lawful basis (D19)

**v2:** lawful basis is **operator-asserted** (supports consent, contract, legitimate-interest). No
active per-principal consent capture/management. A `lawfulBasis` field on the purpose definition and a
reserved predicate seam are included so that **v3 can add first-class consent (Option B)** — granular,
versioned, revocable, provable, with revocation cascading like a soft erasure — by populating a field
and extending one predicate term, **without rearchitecting.**

---

## 10. RBAC & Cross-Session Access

- **The agent is a relying party (D34):** roles are **not** defined in the agent's own directory and
  **not** self-asserted by the connecting party. In v2 they are **assigned by the agent from verified
  credential facts** via the rule mechanism of §5.6 — `roleRules` keyed on `backingIdentity` relation +
  selected claims (title, department, …). Role names, capabilities, and access policies are fully
  agent-pack-configurable; unlimited custom roles. (Runtime external-IAM assignment is v3 — D34/O8.)
- **Assignment defaults (D35):** **home** backing identity → may map its credential facts directly;
  **recognized partner** → safe baseline `partner` unless a rule elevates on a verified claim;
  **unrecognized but valid** → authenticated, no backing identity, `public`-scoped only. Optional
  per-actor overrides for specific known individuals. Services and humans share the **same** mechanism.
- **Recognition ≠ privilege:** being a recognized partner (for `partner-shared` *visibility*) is
  independent of *role* privilege — a recognized partner may hold only a baseline role. Visibility/
  purpose and role stay orthogonal.
- **Role × visibility access matrix** governs cross-session reads. Example:

| Role | `private` | `shared` | `global` | Scope |
|------|-----------|----------|----------|-------|
| `user` | own session only | read | read | own sessions |
| `partner` | no | read (own backing identity) | no | own backing-identity sessions |
| `operator` | no | read | read/write | all sessions |
| `admin` | read | read/write | read/write | all sessions |
| `auditor` | read (RO) | read (RO) | read (RO) | all sessions, all data, no writes |

- **Auditor / audit privilege:** unrestricted **read** to the persistent store (the audit log) for compliance, debugging, oversight — *independent of* what the LLM may access. Visibility scoping restricts the **LLM**; audit roles see everything. This includes decrypting media via the persisted AES keys (§8.2).
- **Cross-session awareness is always privacy-gated (§9.5):** authorization (visibility + lifecycle) precedes retrieval; the LLM never receives cross-session data it wasn't pre-authorized for.
- **Cross-session identity linking:** sessions link to one Principal via its **`(actorIdentity, backingIdentity)`** pair (§5.2) — `actorIdentity` from ECS-Service / ECS-Badge subject, `backingIdentity` presented (service) or trust-resolved from the badge issuer (human). `principal-private` scope keys on the full pair (the same actor across reconnections/screens); `organization-shared` / `partner-shared` key on `backingIdentity` alone (any actor of the same backing identity).

### 10.1 Tool-access tiers & purpose-based elevation (D13–D15)

The central enterprise pattern: employees use a **central agent wired to the company's systems as
tools**, and a role lets them perform their **usual tasks**. Access beyond that is a human-approved
exception. Every `(role, tool)` resolves to one of four tiers (extending the existing
`ALLOW` / `DENY` / `APPROVAL` decision with **`REQUESTABLE`**):

| Tier | When | Behavior |
|------|------|----------|
| **ALLOW** | tool is in the role's set (usual task) | runs directly; the LLM sees the tool |
| **APPROVAL** (sensitivity) | in-role but high-stakes | runs only after per-use human approval (§11.3) |
| **REQUESTABLE** (elevation) | **out-of-role** but listed as escalatable | **not** exposed to the LLM until requested; reachable only via a **purpose-declared, human-approved, time-boxed grant** |
| **DENY** (hard) | out-of-role and not escalatable | never available, not even by request |

**Elevation flow.** The principal needs something their role can't do → the agent matches it to a
`REQUESTABLE` tool and offers an elevation request → the principal **declares a purpose (from the
governance-defined list) + a free-text justification** → a human approver accepts/refuses → on accept,
a **purpose-scoped, time-boxed grant** makes the tool reachable for the window; every use is logged
(§11.3, §9.9). The LLM may draft the request but **never declares the purpose and never self-grants**.

```yaml
# agent-pack: governance-defined purposes + per-tool elevation policy
purposes:                       # the closed list a principal may declare (D15)
  - id: fraud-audit
    label: "Fraud investigation"
  - id: legal-review
    label: "Legal / litigation review"

mcp:
  servers:
    - name: crm
      accessMode: admin-controlled
      toolAccess:
        default: none
        roles:
          employee: [search_customer]
          finance:  [search_customer, read_invoice]
        approval:               # sensitivity gate (in-role, high-stakes)
          - tools: [issue_refund]
            approvers: [finance-manager]
            timeoutMinutes: 60
        elevation:              # NEW: out-of-role exception access (REQUESTABLE)
          - tools: [read_customer_record]
            purposes: [fraud-audit, legal-review]   # declarable purposes for this tool
            approvers: [security-officer]
            grantTtlMinutes: 480                     # time-boxed, purpose-scoped grant
```

A tool in no role and in no `elevation` policy stays a hard `DENY`. **Connection model matters:** with
a shared service-account connection (`admin-controlled`) the agent's RBAC + elevation **is** the
security boundary; with per-principal credentials the wrapped system also enforces its own access.

---

## 11. Interaction Features

### 11.1 Reaction intelligence

The LLM controls reaction handling **per outgoing message** via a `reactionPolicy` attribute stored
on the entry. Modes: `notify` / `historize`, or a conditional rule routing by emoji (e.g. notify on
❤️, historize on 👍). This is an LLM per-message decision, not a system-level config. Reactions arrive
as a dedicated channel-native signal (DIDComm reaction message type; AG-UI state event; omitted on
A2A — §6.5) and are recorded with `sourceType: reaction`.

### 11.2 Message state updates

`created` / `submitted` / `received` / `viewed` / `deleted` are wired to a real handler (replacing
the legacy no-op) and may be recorded as `sourceType: state-update` where useful to the LLM (e.g.
"the principal viewed the generated image").

### 11.3 Approval workflow (human-in-the-loop)

Two uses of one mechanism (§10.1):

- **Sensitivity approval** — an *in-role* but high-stakes tool requires per-use sign-off before it runs.
- **Elevation approval** — an *out-of-role* (`REQUESTABLE`) tool is unlocked by a **declared purpose**: the requester picks a purpose from the **governance-defined list** and adds a **free-text justification**; on approval the grant is **purpose-scoped and time-boxed** (`grantTtlMinutes`), so the tool stays reachable for the window rather than for a single call.

- **Who may approve:** any Principal whose role grants approval rights for the tool/action (`approvers`), optionally restricted by connection type (e.g. only `verifiable-service`).
- **Policies:** `count` (single / N distinct approvers), `percentage` (e.g. 51% majority, 100% unanimous). Optional `vetoOnRefusal`. **Self-approval:** if the requester already holds an approver role for the tool, it runs without a prompt.
- **The LLM never declares the purpose and never self-approves:** it may *detect* the unmet need and *draft* the request, but the human submits it and a human approves it (declaration ≠ authorization, D15).
- **Notification:** eligible approvers are notified (contextual menu item + badge; future: push / DIDComm to offline principals), routed per the capability router. Lifecycle: `PENDING → APPROVED / REJECTED / CANCELLED / EXPIRED` (first approver wins).
- **Persistence:** every request/approval/refusal is a first-class entry (`sourceType: approval` / `refusal`) carrying **`{purpose, justification, approver identity, role, timestamp, grant expiry}`**. The full chain is in the audit log (§9.9).

### 11.4 Auth challenges (D29)

- **Types:** device authentication (biometric/PIN), face-match with liveness, NFC document read.
- **System-enforced, not LLM-decided:** the agent-pack declares per-action challenge requirements; the system enforces them. The LLM is only *informed of the outcome* and adapts (abort on fail, proceed on success).
- **Routing:** always to the Principal's DIDComm capability channel, carrying `originSessionId` so the result returns to the initiating conversation session (§6.2).
- **Fallback (D29):** if no capability channel can serve the challenge, behavior is **configurable per-action, default `fail`** (alt `degrade` — skip and log the unmet requirement). Example:

```yaml
action: transfer_funds
  authChallenge:
    type: biometric
    onChannelUnavailable: fail   # default; alt: degrade
```

- **Persistence:** outcome recorded as `sourceType: auth-challenge` (success/failure, type, timestamp) in the audit trail. (The face-matching service already integrated in the passport service is to be integrated here.)
- **Face-match reference (D33):** the **photo attribute carried in the ECS-Badge** may serve as the reference image the face-match challenge verifies liveness against — the same credential that establishes identity provides the reference for re-verifying it.

### 11.5 Credential presentations

- First-class persistent data tied to the Principal (not ephemeral conversation), available to the LLM via a dedicated tool (`get_principal_credentials`).
- **Revocation re-checked on every privileged use** (D8) — never relying on a stale verification. A credential valid yesterday may be revoked today; this per-use check is also what powers lazy revocation enforcement (§7).

### 11.6 Principal preferences

- LLM persists discovered preferences (language, formatting, communication style) via `save_principal_preference`. Durable across sessions, tied to Principal identity, auto-loaded into context each turn. (Distinct from working memory: preferences are a structured, principal-scoped store; working memory is free-form curated notes.)

---

## 12. Integrity & Audit

- **Reference digests (D24, issue goal 12):** every externally-referenced object (media in object store, credential in a separate store) carries a SHA-256 digest stored in the referencing entry. Chain: Verana-anchored batch digest → covers the message → contains the reference digest → covers the external object. Any modification breaks the chain.
- **Verana anchoring (D27):** the system periodically computes a digest over a batch of persisted entries and anchors it to the [Verana ledger](https://verana-labs.github.io/verifiable-trust-vpr-spec/#mod-di-msg-1-store-digest) via `store-digest`. Anchoring is **async, best-effort, retry-queued — it never blocks the conversation.** Batch cadence is configurable.
- **What is and isn't covered:** authoritative entries + their external references are covered. The semantic index (embeddings) is **explicitly outside** the chain (D17) — derived, rebuildable, hard-deleted on erasure.

---

## 13. Platform Plumbing

### 13.1 System-prompt add-on (issue goal 15)

The **contract between the platform and the LLM**, kept separate from the agent-pack author's persona
prompt. It teaches the LLM the platform-level capabilities it must know to use them at all (working
memory and its rules, that retrieval is purpose-scoped so "missing" recall may mean out-of-scope not
nonexistent, that auth challenges are system-triggered, `reactionPolicy`, preference/credential
tools). Properties:

- **Additive** — never overrides the author's prompt; only appends platform capabilities.
- **Feature-gated** — only describes capabilities actually enabled (no dead instructions, no wasted tokens, no hallucinated tool calls).
- **Versioned (D28)** — so old agent-packs don't break on platform upgrades.
- **Token-conscious** — target < 500 tokens.

### 13.2 Event-driven media generation (D23)

Image (and similar) generation is event-driven: respond quickly and generate in the background.
**Input is not blocked** during generation. The generated artifact is persisted as a `MessageEntry`
ordered by its **initiation timestamp** (§8.2), so transcript ordering is preserved by the timestamp
rather than by holding input. A `generating` state may be surfaced to the LLM so it can answer "still
working on it" if the principal asks. The artifact is persisted to memory (prompt, ref, digest — §8.2)
rather than bypassing it.

### 13.3 Ingestion pipeline

Replaces the legacy string-extraction path. Every inbound transport message → channel adapter →
canonical `MessageEntry` (typed, classified, digested) → memory + orchestration. No raw `'media'`
literals; no memory-bypassing outbound media.

### 13.4 Provider-agnostic LLM & embeddings

Works with text-only and multimodal models. The text-first memory model (§8.2) guarantees a useful
text representation regardless of model modality; multimodal models may additionally consume
referenced media but the system never *depends* on that.

**Adapter taxonomy.** Provider support is organized in three tiers, not one adapter per vendor:

- **Native adapters** — a small set owning a distinct API/runtime (e.g. OpenAI, Anthropic, local via Ollama).
- **OpenAI-compatible generic** — a single adapter (base URL + key) covering the long tail of OpenAI-wire-compatible endpoints and gateways (e.g. OpenRouter, self-hostable LiteLLM, Mistral, Gemini's compat endpoint, Groq, DeepSeek, Together, vLLM, LM Studio). Adding such a provider is **configuration, not code**.
- **Special-protocol adapters** — only where the wire protocol differs from OpenAI: e.g. Azure OpenAI (deployment + `api-key`), AWS Bedrock (SigV4), Google Vertex AI (GCP auth). These earn dedicated adapters because they bring the enterprise data-residency / governance that the privacy posture (§9) values.

**Gateways are first-class.** Aggregators/proxies (OpenRouter, self-hostable LiteLLM) plug in through the OpenAI-compatible tier and add cross-provider fallback, cost/latency routing, and — relevant to §9 — provider data-retention controls.

**Embeddings are equally provider-agnostic.** The semantic index (§8.4, D25) depends on an embeddings provider; it must be pluggable on the same principle — cloud (e.g. OpenAI, Cohere, Voyage) or self-hosted (e.g. `bge` / `nomic` via a local endpoint) — never hard-wired to one vendor.

### 13.5 Schema versioning (D28)

Memory schema, agent-pack schema, and system-prompt add-on are independently versioned, with
migration notes, so deployed packs and stored data survive upgrades.

---

## 14. Data Model

Indicative canonical shapes (field names illustrative; types normative in intent, not in language).

```
Principal
  id
  connectionType        : 'verifiable-user-agent' | 'verifiable-service'
  actorIdentity         : ECS-Service id (service) | ECS-Badge subject (human)
  backingIdentity       : { kind: 'org'|'persona', id }        # indexed (§9.5)
  backingRelation       : 'home' | 'partner' | 'unrecognized'  # resolved via §5.6
  roles                 : [Role]            # assigned by roleRules, not asserted
  identityAttributes    : { userName?, photoRef?, department?, … }  # from badge/creds
  preferences           : [PrincipalPreference]
  capabilityChannelId   : ref → DIDComm channel (nullable for channel-less peers)
  credentialRef         : the underlying credential + lastChecked (TTL liveness, §5.7)

Session
  id
  principalId           : ref → Principal (null while provisional/unbound)
  channelType           : 'didcomm' | 'ag-ui' | 'a2a'
  state                 : 'provisional' | 'active' | 'closed'
  bindingNonce          : one-time, for channel-first binding
  lastActivityAt        : drives idle-timeout (§7)
  credentialLastChecked : drives TTL-bounded revalidation (§5.7)

MessageEntry                         # the canonical memory unit
  id
  sessionId, principalId
  role                  : 'principal' | 'agent' | 'system'
  sourceType            : 'text'|'voice'|'media'|'tool-call'|'approval'
                          |'refusal'|'auth-challenge'|'reaction'|'state-update'
  text                  : text representation (always present)
  attachments           : [MessageAttachment]
  language              : tag (D22)
  # privacy classification (indexed — §9.5)
  visibility            : 'session-private'|'principal-private'
                          |'organization-shared'|'partner-shared'|'public'
  backingIdentity       : { kind, id }   # of the producing Principal; for org/partner scoping
  state                 : 'active'|'archived'|'legal-hold'|'crypto-erased'|'deleted'
  purpose               : tag
  lawfulBasis           : 'consent'|'contract'|'legitimate-interest'  # operator-asserted (D19)
  # integrity
  contentDigest         : SHA-256 over ciphertext (D20)
  encKeyRef             : per-entry key reference (destroyed on crypto-erasure)
  reactionPolicy        : 'notify'|'historize'|conditional   # outgoing only
  createdAt

MessageAttachment
  type                  : 'image'|'voice'|'video'|'file'|'credential'
  bucketKey             : object-store reference (presigned on demand — D24)
  mimeType, size, dims, duration, filename   (as applicable)
  textRepr              : prompt | transcription | description | summary
  aesKey                : for admin/audit decryption — never in LLM context (§8.2)
  refDigest             : SHA-256 of referenced object (§12)

WorkingMemoryNote                     # authoritative, LLM-curated (D11)
  principalId
  content               : free-form text (principal cannot read/edit)
  visibility, purpose, state           # same classification axes
  backingIdentity                      # for org/partner-shared scoping
  lineageRefs           : [MessageEntry ids]   # for erasure cascade (D21)

ApprovalRecord                        # sensitivity (in-role) or elevation (out-of-role) — §11.3
  kind : 'sensitivity' | 'elevation'
  serverName, toolName, actionType, policy, eligibleRoles
  purpose, justification               # elevation: both declared by the human (D15)
  decisions[ {approverId, role, decision, reason, ts} ]
  outcome, vetoed, grantExpiresAt      # elevation grant is purpose-scoped + time-boxed

AuthChallengeRecord
  type, originSessionId, outcome, ts
  faceMatchRef          : badge photo ref, where applicable (§11.4, D33)

CredentialPresentation
  principalId, credentialId, type, issuerDid, summary, refDigest
  # issuer trust-resolved (§5.3); liveness re-checked per TTL (§5.7)

DigestAnchor
  batchId, digest, veranaTxRef, anchoredAt        # never deleted

MemoryAccessLog                       # §9.9
  principalId, role, purpose, scopePredicate,
  matchedEntryIds[], fromSessionIds[], shownToLlm, influencedToolCall, ts

# ---- agent-pack configuration (relying-party policy) ----
RecognizedBackingIdentity             # §5.6
  identity (Org/Persona DID), relation : 'home' | 'partner'
RoleRule                              # §5.6 — evaluated over verified facts, first match wins
  when { relation?, claim.*? }  →  { role | rolesFrom: badgeFacts }
PerActorOverride                      # §5.6
  actor (DID)  →  role
PurposeDefinition                     # governance-defined declarable purposes (§9.4 / §10.1)
  id, label, parent (hierarchy), lawfulBasis
ElevationPolicy                       # per-tool out-of-role access (§10.1)
  tools[], purposes[], approvers[ roles ], grantTtlMinutes
RolePurposeCeiling                    # OPTIONAL role → [permitted purposes] check at elevation (§9.4)
ElevationGrant                        # runtime: an approved, time-boxed grant
  principalId, serverName, toolName, purpose, approverId, grantedAt, expiresAt
```

Note: a MessageEntry also carries `backingIdentity` (indexed) — the backing identity of the Principal
whose interaction produced it — which the §9.5 predicate compares for `organization-shared` /
`partner-shared`.

---

## 15. Deferred / Open Questions

| # | Item | Disposition |
|---|------|-------------|
| O1 | **First-class consent management** (granular, versioned, revocable, provable; revocation as soft-erasure) | **v3.** Seam reserved in v2 (D19): `lawfulBasis` field + extensible scope predicate. |
| O2 | **Ingestion-time PII redaction/tokenization** (PAN-style data) | **Open.** The correct layer is input-side, not the privacy-scope model (D18). Not in v2. |
| O3 | **DB row-level security (RLS)** for PEP enforcement | **v3 hardening** (D16). v2 enforces app-layer via the centralized query-builder. |
| O4 | **Purpose-matching algorithm** specifics (hierarchy compatibility rules) | Model grounded in PBAC purpose hierarchies (§9.1); precise algorithm is an implementation detail referencing the PBAC literature. **Applies to elevation-purpose matching (§9.4 / §10.1); memory retrieval no longer uses a purpose predicate (O9).** |
| O5 | **Token budget tuning** (window size, summary trigger, add-on cap) | Defaults provided as tunable, not hard commitments. |
| O6 | **Credential-liveness interval `N`** (max-staleness-while-active) | Configurable (§5.7 / D8). Bounds worst-case post-revocation exposure for an active session; tradeoff between exposure window and revocation-registry load. Operational tuning, not solved here. |
| O7 | **Embeddings & immutability** | Resolved (D17): outside the chain, hard-deleted on erasure. Noted here for visibility. |
| O8 | **External IAM integration** (runtime IAM as role-assignment authority via OIDC/SAML/SCIM — Option 2) | **v3** (D34). v2 uses agent-pack-native role rules (Option 1); IAM integration adds a live dependency with a different availability profile and is designed separately. |
| O9 | **Continuous per-purpose memory minimization** (purpose as an always-on predicate term scoping every memory retrieval, beyond visibility + lifecycle) | **Deferred.** v2 scopes memory by **visibility + lifecycle state** and realizes purpose as human-approved **elevation** (D13–D15); purpose is recorded for audit. Whether the agent's own memory also needs continuous purpose-scoping (stronger GDPR data-minimization) is revisited later. |

---

## 16. Phased Rollout

Sequenced by dependency — each phase builds on the prior.

1. **Phase 1 — Canonical core & memory model.** Channel-agnostic core, `MessageEntry`, typed attachments, long-term store, token-based window. (Foundation for everything.)
2. **Phase 2 — Privacy & PBAC.** Two-axis classification, purpose model, PEP/PDP query-builder, access-before-retrieval, access log. (Must precede cross-session features.)
3. **Phase 3 — Semantic memory.** pgvector index, hybrid retrieval, working memory, summarization. (Scoped by Phase 2.)
4. **Phase 4 — Channels.** DIDComm capability channel, AG-UI adapter + binding flows, A2A adapter, capability router. (Consumes the core's capability events.)
5. **Phase 5 — Interaction features.** Reactions, approvals, auth challenges, credential tool, preferences, message-state handling.
6. **Phase 6 — Integrity & retention.** Reference digests, Verana batch anchoring, retention/archival jobs, crypto-erasure + lineage cascade.

> Phases 1–3 are the critical path: they establish the data model and the privacy guarantees that
> everything else depends on. Channels (4) can begin in parallel once the canonical `MessageEntry`
> and capability-event vocabulary (Phase 1) are frozen.
