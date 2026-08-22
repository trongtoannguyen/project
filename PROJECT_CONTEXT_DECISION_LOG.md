# Project Context & Decision Log — Messaging Platform

> **Status:** Active source of truth  
> **Last consolidated:** 2026-08-23  
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
| #4 | 2026-08-16 | Active | Start as a modular monolith with clear domain boundaries and integration-ready contracts. Do not build a generic external API in parallel with MVP. | The end-user application reveals what should become generic. A later API must be an access layer over application/domain contracts—not a database-model exposure. | Revisit when a concrete external consumer needs integration. |

### Interpretation notes

- #1’s mention of group messaging is the product direction; #2 deliberately stages it after MVP v1. These are compatible, not competing decisions.
- #4 is an architectural constraint, not a promise to deliver API keys, multi-tenancy, SDKs, webhooks, or OAuth in MVP.

---

## 3. Active scope and acceptance boundary

### MVP v1 outcome

An invited user can register and log in on the web client, start or access a direct conversation, exchange persisted text messages with another authenticated user in real time when both are online, retrieve message history after reload or offline time, and record a basic read state.

### Must-have acceptance criteria

| Capability | MVP v1 is accepted when… | Explicit boundary |
| --- | --- | --- |
| Authentication | A user can register and log in with email/password, receive and use JWT-based authentication, and cannot access another user's protected resources. | Password-reset, social login, MFA, and external OAuth are excluded. |
| Direct conversation | An authenticated user can create or access a 1-to-1 conversation and list their own conversations. | A conversation accepts exactly two distinct participants in v1. Group creation and group membership management are excluded. |
| Text message | A participant can send a non-empty text message only to a direct conversation they belong to. | Attachments, edit/delete, reactions, replies, and rich content are excluded. |
| Persistence and history | Accepted messages survive reload/reconnect; an authorized participant can retrieve their direct-conversation history. | Search, retention policies, export, and advanced pagination UX are excluded; API-level pagination policy remains a design question. |
| Real-time delivery | If sender and recipient are connected, the recipient receives the newly accepted message through WebSocket without manually reloading. | Cross-node socket routing, push notifications, and a delivered-status guarantee are excluded. Reconnect behaviour must recover history rather than silently losing accepted messages. |
| Basic read receipt | A recipient can mark a message/conversation read, and the sender can see an unambiguous basic read state. | `delivered` is not a separate MVP state. Per-device and per-message receipt detail are not yet fixed. |
| Operational usability | Invited users can use the normal flow without the founder manually repairing routine failures. Failures are surfaced clearly and reconnect/reload does not erase accepted messages. | This is a small private test release, not an uptime/SLA commitment. |

### Should-have follow-up within the MVP phase (not a v1 acceptance gate)

- Group conversations and N-recipient real-time fan-out (the next milestone).
- Basic online/offline presence.
- A distinct `delivered` state in addition to `read`.

Presence and the distinct `delivered` state retain their original “should have” priority: they are planned immediately after the v1 must-have flow, without creating a separate milestone. Group chat remains the exception: it is a separate milestone under decision #2.

### Explicitly out of MVP

- Typing indicators; message edit/delete; attachments; push notifications; message search.
- Voice/video calls; AI features; stories/status; advanced multi-device sync.
- External API keys, multi-tenancy, OAuth, SDKs, and webhooks.
- Microservices, Kafka, sharding, multi-region, Kubernetes, and distributed WebSocket infrastructure.

---

## 4. Architectural constraints and principles

### Constraints that apply now

- **Modular monolith first:** Java, Spring Boot, PostgreSQL, Redis, WebSocket, REST API, JWT, Gradle, JUnit, Testcontainers, Docker, and GitHub Actions are the current intended starting stack. A substitution needs a recorded decision and rationale.
- **Domain before transport:** Controllers/WebSocket handlers and persistence infrastructure must not define the domain model. External-facing contracts must not directly expose database models.
- **Integration-first, not integration-now:** Use clear application boundaries and stable contracts so a future public API is additive. Do not generalize before there is a real consumer/use case.
- **N-participant-ready model:** Model conversation participation separately so group chat can be enabled later without a data-model rewrite; enforce the v1 two-participant rule in application/domain validation.
- **Reliability as a learning requirement:** Design explicitly for timeout, retry, reconnect, duplicate requests/messages, concurrent actions, and authorization failure. The exact guarantees are open until specified below.
- **Simple, measurable scaling:** Introduce Redis, event infrastructure, service extraction, or distributed WebSockets only for a named requirement/bottleneck with trade-offs recorded.

### Non-goals for initial architecture

No Kafka merely to be “event-driven”; begin with in-process domain events when an event is useful. No distributed design merely to simulate scale. No public database access as an integration method.

---

## 5. Current constraints, assumptions, and dependencies

### Constraints

- Solo founder/developer; work in small, finishable increments with limited work in progress.
- Backend learning and reliability take priority over frontend styling.
- Small private test group means real behaviour matters, while enterprise scale and market growth do not yet drive requirements.

### Working assumptions (not decisions; validate or replace)

- Email/password is sufficient for private MVP identity and account recovery can wait.
- One web client session per user is adequate for MVP validation; simultaneous-device semantics are not yet required.
- A single deployable instance is sufficient for MVP. Horizontal scale is not an MVP acceptance condition.
- Message content is plain text and sensitive enough to require normal authenticated transport and access control, but end-to-end encryption is not currently required.
- Friends can be invited/identified through a simple workflow; a contacts/social graph is not implied.

### Dependencies

- A deployable environment reachable by invited testers.
- A chosen frontend stack and a minimal web client implementation.
- A defined way to provision/invite test users.
- Test environments for PostgreSQL and any selected runtime dependencies.

---

## 6. Decisions required before domain modeling

These questions materially affect aggregates, invariants, identifiers, authorization, or message/receipt semantics. Resolve them before treating the domain model as stable.

1. **Identity and lifecycle:** Is email globally unique and immutable for MVP? Are users allowed to self-register freely, or is registration invite-only for the private test?
2. **Direct-conversation uniqueness:** Must a pair of users have exactly one active direct conversation, or may they create multiple threads? Can a user message themselves?
3. **Message ordering:** What ordering must users observe within a conversation—server acceptance order, client timestamp, or another rule? Is a gap-free sequence required?
4. **Duplicate/idempotency behaviour:** Will the client supply a message idempotency key? On retry/reconnect, should the server return the original accepted message rather than create another?
5. **Read-receipt unit:** Is “read” stored per message, or as a participant’s last-read position per conversation? What precisely appears to the sender in MVP?
6. **Offline/reconnection contract:** Does “offline history” mean polling/REST history after reconnect is sufficient, or must WebSocket automatically backfill missed messages? What is the user-visible recovery expectation?
7. **Authorization and membership lifecycle:** Can either participant leave, archive, block, or delete a direct conversation in MVP? If none applies, is membership immutable once created?

Questions that can wait until after initial domain modeling: exact frontend framework, presence protocol, delivered-state definition, and group membership roles.

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

Resolve the seven domain-shaping questions in section 6, then model `User`, `Conversation`, `ConversationParticipant`, `Message`, and the chosen read-receipt representation. Define invariants before database tables or REST/WebSocket payloads.
