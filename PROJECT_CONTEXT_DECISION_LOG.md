# Project Context & Decision Log — Messaging Platform

> **Status:** Active source of truth  
> **Last consolidated:** 2026-08-23 (added #12 — participant lifecycle / privacy-freedom principle)  
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

The Section 6 gate is resolved (#12). Proceed to model `User`, `Conversation`, `ConversationParticipant`, and `Message`. Define invariants before database tables or REST/WebSocket payloads. Decisions #7–#10 should be reflected directly in the model and contracts; #12 should shape `Message` and `ConversationParticipant` so a future block relationship and per-participant delete-visibility flag don't require a breaking migration, without implementing them in v1. Before implementation, define a minimal service map, service-owned data, synchronous/asynchronous contract boundaries, and failure modes for the end-to-end MVP slice.
