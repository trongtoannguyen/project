# Project Context & Decision Log — Messaging Platform

> **Status:** Active source of truth  
> **Last consolidated:** 2026-08-25 (added #13 — initial service map: Identity / Messaging / Realtime services, stateless JWT; detailed contracts moved to `service-boundaries/*.md`)  
> **Authority:** This document records product and architecture decisions specific to this project. `AGENTS.md` defines the operating principles and process. If they appear to conflict, this document governs project-specific decisions; the operating principles still govern how new decisions are made.
>
> **Change rule:** Do not edit or remove a historical decision. Record a new decision with a new ID and mark the earlier one `Superseded by #N` when applicable. Clarifications that do not alter a decision may be added with their date and source.

---

## 1. Project charter

### Product

A self-built, real-time messaging platform for direct (1-to-1) and, after the first milestone, group conversation. It is both a serious learning project for system design/system thinking and a future reusable messaging capability for the founder's other products.

### Intended MVP users

The founder and a small group of invited friends/acquaintances. Their real usage is intended to reveal genuine behaviour and failure modes; this project is not currently seeking mass-market product-market fit or growth.

### Problem and value

The founder needs a messaging foundation they control, rather than depending on Firebase, Twilio, Stream, or a comparable provider. The immediate value is learning real-time systems through justified design decisions; the long-term value is a stable core with clean boundaries that can later be exposed to other projects without a rewrite.

### Success orientation

Prioritize, in this order:

1. Learning value demonstrated by explicit, evidence-based decisions.
2. A working, dependable end-to-end messaging flow for invited users.
3. Clean future integration boundaries.

Do not prioritize feature parity, visual polish, user growth, or generic external APIs in the first MVP.

---

## 2. Decision register (chronological, immutable history)

| ID | Date | Status | Decision | Rationale / consequence | Revisit trigger |
| --- | --- | --- | --- | --- | --- |
| #1 | 2026-08-16 | Active | Build a self-owned real-time messaging platform for learning first, with later reuse in the founder's products. Initial intended capability includes 1-to-1 and group messaging. | Optimize for learning value and clean architecture rather than product-market fit. Real invited-user usage is part of validation. | Supersede only if project purpose changes. |
| #2 | 2026-08-16 | Active | MVP v1 exposes **only 1-to-1 conversations**. Group chat is the immediately subsequent milestone. | Learn connection lifecycle, persistence, delivery, and read flow before group fan-out. Model participation for N users from the start, but enforce two participants in MVP. | Revisit after stable real-user validation of MVP v1 real-time delivery. |
| #3 | 2026-08-16 | Active | Build a web client first; mobile follows later. The founder builds both backend and frontend. | Web enables faster end-to-end validation. MVP frontend is a functional test client, not a UI-polish project. | Revisit after MVP feedback or a mobile requirement. |
| #4 | 2026-08-16 | Superseded by #11 | Start as a modular monolith with clear domain boundaries and integration-ready contracts. Do not build a generic external API in parallel with MVP. | The end-user application reveals what should become generic. A later API must be an access layer over application/domain contracts—not a database-model exposure. | Superseded on 2026-08-23 by an initial-microservices direction. |
| #5 | 2026-08-23 | Active | MVP identity uses a normalized, globally unique, immutable email. Registration is invite-only through a founder-managed email allowlist; no email invitation delivery workflow is required. | Fits a small private test while avoiding public-registration abuse and email-delivery scope. Email changes and password recovery are deferred. | Revisit when public onboarding or account recovery is required. |
| #6 | 2026-08-23 | Active | A pair of distinct users has exactly one direct conversation; self-messaging is not allowed. | Prevents duplicate threads and simplifies conversation list, history, and receipt semantics. | Revisit only if multiple named threads per pair becomes a user need. |
| #7 | 2026-08-23 | Active | Messages have a server-assigned, unique, strictly increasing `sequenceNumber` within a conversation. Client timestamps do not determine order; gap-free numbering is not a public guarantee. | Produces a deterministic order under concurrent sends and supports history sync and read positions without imposing unnecessary sequence guarantees. | Revisit if a different ordering or replication requirement emerges. |
| #8 | 2026-08-23 | Active | The client creates and retains a UUID `clientMessageId` for each user send across retries/reconnects. The server enforces uniqueness on `(conversationId, senderId, clientMessageId)`; an equivalent retry returns the originally accepted message, while the same key with different content conflicts. | Prevents duplicate messages when a network response is lost and makes retry behaviour explicit. Server `messageId` remains the canonical identifier. | Revisit when additional client types or delivery protocols require a broader idempotency contract. |
| #9 | 2026-08-23 | Active | Store each participant's monotonic `lastReadSequence` per conversation rather than a receipt record per message. Read state is derived: messages at or below the other participant's read position are read. | Compact for MVP and naturally extends to group conversations; no write per message is needed. | Revisit if per-message/per-device receipt auditability becomes a requirement. |
| #10 | 2026-08-23 | Active | WebSocket delivers live messages only. After reload or reconnect, the client synchronizes persisted missed messages through REST using its last known per-conversation sequence; it then resumes live WebSocket delivery. | Separates durable sync from transient socket notification, makes recovery testable, and avoids socket-session backfill complexity. | Revisit for multi-device sync, offline-first clients, or a justified richer delivery protocol. |
| #11 | 2026-08-23 | Active | Start the project with a microservices architecture, replacing the initial modular-monolith direction in #4. Preserve clear domain boundaries, stable contracts, and integration-first design. | Founder-directed architecture change. It enables early practice with service boundaries and distributed-system concerns, while adding operational, testing, data-consistency, and delivery complexity to MVP. Exact service boundaries and infrastructure choices remain unapproved until defined from the domain and MVP flows. | Revisit after the first end-to-end MVP slice demonstrates whether the added distribution cost is serving the learning and product goals. |
| #13 | 2026-08-25 | Active | Initial service map for microservices direction (#11): **3 services** — (1) **Identity Service** owns the `User` aggregate; (2) **Messaging Service** owns `Conversation`+`ConversationParticipant` and `Message` **together in one service/one database** (not split further); (3) **Realtime Service** owns no domain aggregate, only WebSocket connection lifecycle and live delivery. JWT authentication is **stateless** — Identity Service signs tokens; Messaging and Realtime services verify locally via shared key, with no per-request call back to Identity Service. Full boundary/contract detail lives in separate per-service files (see below), not in this log. | Cross-aggregate invariants (#6, #7, sender-participant validity) are heaviest between `Conversation` and `Message` — keeping them in one service avoids distributed-transaction/saga complexity with no current bottleneck justifying it (founder-directed: avoid over-engineering at this stage). Stateless JWT avoids a cascading runtime dependency on Identity Service from every other service; trade-off accepted is no instant token revocation in v1. Detailed contracts, service-owned data, and failure modes are kept out of this log to avoid over-detailing it — see `service-boundaries/identity-service-boundary.md`, `service-boundaries/messaging-service-boundary.md`, `service-boundaries/realtime-service-boundary.md`. | Revisit Messaging Service split if group chat or write volume creates a measured bottleneck. Revisit stateless JWT if instant revocation/session-management becomes a real requirement. Revisit the deterministic-UUIDv5 `conversationId` scheme (used for #6, 1-1 only) when group chat is designed — it does not auto-extend if group conversations may share an identical participant set (see `messaging-service-boundary.md` §8). |
| #12 | 2026-08-23 | Active | Resolves the Section 6 open question. Direct conversations are **immutable** in MVP v1 — no server-side leave, archive, or delete of the conversation itself. Message delete and Block remain **out of MVP v1**, deferred to the milestone immediately after v1 (does not change v1 acceptance scope). For that follow-up milestone, the product adopts a **Telegram-style, privacy/freedom-first model**: (a) **Block** is a real domain relationship (`User A blocks User B`) that prevents new messages from the blocked party; existing conversation/history is unaffected. (b) **Message delete** supports both "delete for me" (hidden only for the deleting participant) and "delete for everyone" (hidden for both participants) as user-chosen, per-message options; in both cases the server **always retains the underlying message data** — deletion is a per-participant visibility flag at the application layer, never physical/domain-level deletion. | Founder-directed: this product treats privacy, user data control, and freedom over one's own conversation as a core product value, not just a UX nicety. Keeping v1 scope to "immutable conversation" avoids re-opening the out-of-MVP message-edit/delete boundary while still recording the long-term shape so later domain modeling (delete-visibility per participant, block relationship) isn't designed blind. | Revisit if the privacy/freedom principle turns out to require server-side purge (e.g. legal/compliance need), or if "delete for everyone" needs additional guarantees (e.g. edit window, notification of deletion). |

### Interpretation notes

- #1’s mention of group messaging is the product direction; #2 deliberately stages it after MVP v1. These are compatible, not competing decisions.
- #11 supersedes #4’s modular-monolith starting point. Its boundary and contract principles remain active; it does not promise API keys, multi-tenancy, SDKs, webhooks, or OAuth in MVP.
- #12 resolves the Section 6 gate: v1 domain modeling can proceed with an immutable-conversation assumption. Block and per-participant message-delete visibility are not modeled into v1 tables/contracts, but the domain model for `Message` and `ConversationParticipant` should anticipate them (e.g. avoid a shape that would require a breaking migration to add a visibility flag or a block relationship later).

---

## 3. Active scope and acceptance boundary

### MVP v1 outcome

An invited user can register and log in on the web client, start or access a direct conversation, exchange persisted text messages with another authenticated user in real time when both are online, retrieve message history after reload or offline time, and record a basic read state.

### Must-have acceptance criteria

| Capability | MVP v1 is accepted when… | Explicit boundary |
| --- | --- | --- |
| Authentication | A user on the founder-managed allowlist can register and log in with email/password, receive and use JWT-based authentication, and cannot access another user's protected resources. | Email is normalized, globally unique, and immutable in MVP. Password-reset, social login, MFA, external OAuth, and a full invitation-email workflow are excluded. |
| Direct conversation | An authenticated user can create or access their one direct conversation with another distinct user and list their own conversations. | A conversation accepts exactly two participants in v1; a user cannot message themselves or create another thread for the same pair. Group creation and membership management are excluded. |
| Text message | A participant can send a non-empty text message only to a direct conversation they belong to; retried sends do not create duplicates. | Each send carries a retained client idempotency key. Attachments, edit/delete, reactions, replies, and rich content are excluded. |
| Persistence and history | Accepted messages survive reload/reconnect; an authorized participant can retrieve messages after their last known conversation sequence. | Messages are ordered by server-assigned `sequenceNumber`; search, retention policies, export, and advanced pagination UX are excluded. |
| Real-time delivery | If sender and recipient are connected, the recipient receives the newly accepted message through WebSocket without manually reloading. After reconnect/reload, the client recovers missed persisted messages through REST sync, then resumes live socket delivery. | Cross-node socket routing, push notifications, socket-driven backfill, and a delivered-status guarantee are excluded. |
| Basic read receipt | A recipient can advance their conversation `lastReadSequence`; the sender sees which messages are read from that position. | `delivered` is not a separate MVP state. Per-device and per-message receipt records are excluded. |
| Operational usability | Invited users can use the normal flow without the founder manually repairing routine failures. Failures are surfaced clearly and reconnect/reload does not erase accepted messages. | This is a small private test release, not an uptime/SLA commitment. |

### Should-have follow-up within the MVP phase (not a v1 acceptance gate)

- Group conversations and N-recipient real-time fan-out (the next milestone).
- Basic online/offline presence.
- A distinct `delivered` state in addition to `read`.

Presence and the distinct `delivered` state retain their original “should have” priority: they are planned immediately after the v1 must-have flow, without creating a separate milestone. Group chat remains the exception: it is a separate milestone under decision #2.

### Explicitly out of MVP

- Typing indicators; message edit/delete (see #12 — planned as the milestone immediately after v1, not designed away); attachments; push notifications; message search.
- Voice/video calls; AI features; stories/status; advanced multi-device sync.
- External API keys, multi-tenancy, OAuth, SDKs, and webhooks.
- Kafka, sharding, multi-region, Kubernetes, distributed WebSocket infrastructure, and service splits beyond the approved initial service map.

---

## 4. Architectural constraints and principles

### Constraints that apply now

- **Microservices from inception:** The system starts as independently deployable services with explicit contracts. Service boundaries must derive from the domain and MVP flows, rather than technical layers or speculative future scale. Java, Spring Boot, PostgreSQL, Redis, WebSocket, REST API, JWT, Gradle, JUnit, Testcontainers, Docker, and GitHub Actions remain intended starting technologies; each service/infrastructure substitution needs a recorded rationale.
- **Domain before transport:** Controllers/WebSocket handlers and persistence infrastructure must not define the domain model. External-facing contracts must not directly expose database models.
- **Integration-first, not integration-now:** Use clear application boundaries and stable contracts so a future public API is additive. Do not generalize before there is a real consumer/use case.
- **N-participant-ready model:** Model conversation participation separately so group chat can be enabled later without a data-model rewrite; enforce the v1 two-participant rule in application/domain validation.
- **Reliability as a learning requirement:** Design explicitly for timeout, retry, reconnect, duplicate requests/messages, concurrent actions, and authorization failure. The exact guarantees are open until specified below.
- **Deliberate distributed complexity:** Microservices are an explicit initial decision (#11), not an accidental result. Introduce Redis, event infrastructure, distributed WebSockets, or additional service splits only for a named requirement/bottleneck with trade-offs recorded.

### Non-goals for initial architecture

No Kafka merely to be “event-driven”; begin with the simplest service-to-service integration that meets a named requirement. No public database access as an integration method. Each service owns its data and exposes a contract rather than sharing its database.

---

## 5. Current constraints, assumptions, and dependencies

### Constraints

- Solo founder/developer; work in small, finishable increments with limited work in progress.
- Backend learning and reliability take priority over frontend styling.
- Small private test group means real behaviour matters, while enterprise scale and market growth do not yet drive requirements.

### Working assumptions (not decisions; validate or replace)

- Founder-managed email allowlisting is sufficient for private MVP onboarding; account recovery and email change can wait.
- One web client session per user is adequate for MVP validation; simultaneous-device semantics are not yet required.
- MVP may run with one instance per service; horizontal scale is not an MVP acceptance condition.
- Message content is plain text and sensitive enough to require normal authenticated transport and access control, but end-to-end encryption is not currently required.
- Friends can be invited/identified through a simple workflow; a contacts/social graph is not implied.

### Dependencies

- A deployable environment reachable by invited testers.
- A chosen frontend stack and a minimal web client implementation.
- A defined way to provision/invite test users.
- Test environments for PostgreSQL and any selected runtime dependencies.

---

## 6. Decisions required before domain modeling

1. **Authorization and membership lifecycle — RESOLVED by #12 (2026-08-23).** Direct conversations are immutable in v1 (no server-side leave/archive/delete). Block and per-message delete-visibility are deferred to the next milestone, with the shape recorded in #12 so the v1 domain model can anticipate them without a breaking migration.

Questions that can wait until after initial domain modeling: exact frontend framework, presence protocol, delivered-state definition, group membership roles, and public/invite-email onboarding.

---

## 7. Decision-record template for future sessions

```text
#N — Short decision title
Date:
Status: Proposed | Active | Superseded by #N | Rejected
Owner: Founder
Decision:
Problem / context:
Alternatives considered:
Why this choice now:
Trade-offs and consequences:
Scope affected:
Architecture/domain impact:
Validation or evidence required:
Revisit trigger:
```

For externally observable events or integration contracts, additionally record: event/contract version, producer, payload/schema, ordering, delivery semantics, retry behaviour, and idempotency strategy.

---

## 8. Next recommended lifecycle step

Domain modeling (Section 9) is complete. The initial service map and per-service boundaries are resolved by #13 — see `service-boundaries/identity-service-boundary.md`, `service-boundaries/messaging-service-boundary.md`, `service-boundaries/realtime-service-boundary.md` for service-owned data, API/internal contracts, the `sequenceNumber` assignment mechanism, and end-to-end failure modes for the MVP slice.

Proceed next to: database schema design (derived from Section 9 + the service-boundary files, not independently), then REST/WebSocket payload finalization, then Implementation.

---

## 9. Domain model — User, Conversation, ConversationParticipant, Message

> **Consolidated:** 2026-08-23. Scope: entities, value objects, aggregates, invariants only. Database schema and API/WebSocket contracts are deliberately out of scope for this section — they belong to the next lifecycle step (Section 8) and must be derived from this model, not defined ahead of it.

### 9.1 Entities

| Entity | Identity | Aggregate role | Notes |
| --- | --- | --- | --- |
| `User` | `userId` | Root of its own aggregate | Email (#5) is a property, not the identity, despite being immutable. |
| `Conversation` | `conversationId` | Root of the `Conversation` aggregate | Immutable participant set after creation in v1 (#12). |
| `ConversationParticipant` | Composite `(conversationId, userId)` | Child entity within the `Conversation` aggregate | Created together with the `Conversation` in v1 — no later join. Future home for `lastReadSequence` (#9) and, post-v1, block/delete-visibility state (#12). |
| `Message` | `messageId` (server-assigned, canonical per #8) | Root of its own aggregate — **decision confirmed 2026-08-23** | Carries `conversationId` and `senderId` as references, not object graph edges. |

### 9.2 Value objects

| Value object | Owning entity | Purpose |
| --- | --- | --- |
| `Email` | `User` | Normalized, globally unique, immutable (#5). |
| `SequenceNumber` | `Message` | Server-assigned; strictly increasing within a conversation (#7). Modeled as a VO rather than a raw integer so "only the server assigns it, it only increases" is enforced at the type level. |
| `ClientMessageId` | `Message` | Client-generated UUID used for send idempotency (#8). |
| `MessageContent` | `Message` | Non-empty text in v1; encapsulates the "non-empty" validation in one place. |
| `LastReadSequence` | `ConversationParticipant` | Wraps `SequenceNumber` with a "monotonic, never decreases" invariant (#9). |

**Explicitly deferred, not modeled in v1:** `ParticipantRole` (no role concept while conversations are strictly 2-party), any block relationship, and any per-participant message-visibility flag. Per the 2026-08-23 decision, these are **not** added as placeholder fields now — v1's domain model carries only what v1 requires. #12 is satisfied by keeping the current shape from actively blocking these additions later (e.g. not hard-coding a two-party-only visibility assumption into `Message` itself), not by pre-adding empty fields. This is a design constraint to carry into the later persistence/schema decision, not a Section 9 model element.

### 9.3 Aggregates

**Aggregate 1 — `User`**
Standalone; no child entities in v1.

**Aggregate 2 — `Conversation` (root) + `ConversationParticipant` (child)**
Grouped because the invariant "exactly two distinct participants, no duplicate pair" (#6) can only be enforced consistently if `Conversation` and its `ConversationParticipant`s are created/validated within one transactional boundary. `ConversationParticipant` has no reason to exist independently of its `Conversation`.

**Aggregate 3 — `Message` (root, independent of `Conversation`) — confirmed 2026-08-23**
`Message` is its own aggregate root rather than a child of `Conversation`. Rationale: a conversation can hold many thousands of messages; nesting `Message` under `Conversation` would force loading or paginating the whole history to validate any conversation-level invariant, violating the "keep aggregates small" principle, and doesn't fit a write-heavy, one-transaction-per-message pattern. `Message` holds `conversationId` as a plain reference.

Trade-off accepted: `sequenceNumber`'s strict-increase and uniqueness guarantee (#7) is no longer self-enforced by a single aggregate instance, since it spans many `Message` instances within one `conversationId`. This becomes a cross-aggregate invariant that needs a mechanism at the application/domain-service or persistence layer (e.g. a per-conversation DB sequence/constraint, or a single-writer-per-conversation pattern). Design of that mechanism is explicitly deferred to Architecture/Technical Planning (see 9.5).

### 9.4 Invariants enforceable within a single aggregate instance

**`Conversation` + `ConversationParticipant`**

- Exactly two participants, both distinct `User`s (#6).
- Participant set does not change after creation in v1 — no leave/archive/delete (#12).
- A participant's `LastReadSequence` only increases, never decreases (#9).

**`Message`**

- `MessageContent` must be non-empty.
- `sequenceNumber` is assigned only by the server (never client-supplied) and increases relative to prior state at creation time.
- `(conversationId, senderId, clientMessageId)` is unique; identical key + identical content returns the original accepted message (idempotent retry); identical key + different content is a conflict (#8).
- Message content is immutable after creation in v1 (edit/delete out of v1 scope per #12).

**`User`**

- Email is unique system-wide and immutable after creation (#5).

### 9.5 Invariants spanning multiple aggregate instances (not enforced by the model alone)

These are recorded here so they are not lost, but **mechanism design is deferred to the Architecture/Technical Planning lifecycle step**, per the 2026-08-23 decision:

- A given pair `(userA, userB)` has at most one `Conversation` (#6) — spans potentially many `Conversation` instances; needs a domain/application service or a uniqueness constraint at the persistence layer.
- `sequenceNumber` values across all `Message`s sharing one `conversationId` must be strictly increasing and non-duplicate — spans many `Message` instances; see trade-off in 9.3.
- The `senderId` of a `Message` must be a valid current participant of its `conversationId` at send time — spans the `Message` aggregate and the `Conversation` aggregate; `Message` does not hold a direct reference to `ConversationParticipant`, so this needs to be checked via an application/domain service against a repository, not via in-memory object navigation.

### 9.6 Explicit decisions made during this modeling pass (2026-08-23)

1. `Message` is confirmed as an independent aggregate root (not nested in `Conversation`).
2. No placeholder fields (e.g. `visibilityState`, block relationship) are added to the v1 model for #12; the model is kept to v1's actual requirements, with the constraint that the eventual persistence/schema design must not foreclose adding them without a breaking migration.
3. Design of the enforcement mechanism for cross-aggregate invariants (9.5) is deferred to the Architecture/Technical Planning lifecycle step, not designed here.
