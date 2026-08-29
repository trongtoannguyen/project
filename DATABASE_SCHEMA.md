# Database Schema — Messaging Platform

> Nguồn quyết định: derive từ `PROJECT_CONTEXT.md` Section 9 (domain model) + `service-boundaries/*.md` (contract/cơ chế enforce invariant). File này chỉ chứa DDL cụ thể — lý do/trade-off của từng quyết định nằm ở 2 nguồn trên, không lặp lại ở đây để tránh trùng lặp khi update.
>
> **Quy ước cập nhật:** mỗi lần schema đổi, thêm 1 mục vào "Lịch sử thay đổi" cuối file (ngày, bảng/cột bị ảnh hưởng, lý do ngắn gọn, link tới decision # hoặc boundary file liên quan). Không sửa DDL cũ trong lịch sử — chỉ thêm migration mới, giống nguyên tắc "không sửa lịch sử" của `PROJECT_CONTEXT.md`.
>
> Mỗi service sở hữu database riêng — không có FK vật lý cross-service (theo #13). Reference sang service khác (`user_id`, `sender_id`) là UUID thuần, hợp lệ hóa qua JWT + application layer, không qua FK constraint.

---

## Identity Service

Nguồn: `service-boundaries/identity-service-boundary.md` §2.

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY,
    email           VARCHAR(320) NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_users_email_normalized UNIQUE ((LOWER(email)))
);
```

**Ghi chú vận hành:**

- `id`: sinh UUID ở application layer trước khi insert (không dùng DB-specific default, giữ portable).
- Application layer luôn `email.trim().toLowerCase()` trước khi insert/query; functional index `LOWER(email)` là lưới an toàn tầng DB.
- Không có `updated_at` — không field nào được sửa sau khi tạo ở v1 (#5).

---

## Messaging Service

Nguồn: `service-boundaries/messaging-service-boundary.md` §2–3.

```sql
-- conversations
-- id sinh deterministic (UUIDv5) ở application layer cho 1-1 (v1).
-- Xem messaging-service-boundary.md §3 cho cơ chế đầy đủ.
CREATE TABLE conversations (
    id          UUID PRIMARY KEY,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- conversation_participants
-- Entity con trong aggregate Conversation. Tạo cùng lúc với
-- conversations, không join sau (v1, #12).
CREATE TABLE conversation_participants (
    conversation_id                UUID NOT NULL REFERENCES conversations(id),
    user_id                        UUID NOT NULL,
    last_read_sequence             BIGINT NOT NULL DEFAULT 0,
    recipient_verification_status  VARCHAR(10) NOT NULL DEFAULT 'VERIFIED',

    PRIMARY KEY (conversation_id, user_id),
    CONSTRAINT chk_verification_status
        CHECK (recipient_verification_status IN ('VERIFIED', 'PENDING', 'FAILED'))
);

CREATE INDEX idx_conversation_participants_user
    ON conversation_participants (user_id);

-- messages
-- Aggregate root độc lập. conversation_id/sender_id là reference thuần.
CREATE TABLE messages (
    id                  UUID PRIMARY KEY,
    conversation_id     UUID NOT NULL REFERENCES conversations(id),
    sender_id           UUID NOT NULL,
    sequence_number     BIGINT NOT NULL,
    client_message_id   UUID NOT NULL,
    content             TEXT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_messages_content_non_empty CHECK (length(trim(content)) > 0),
    CONSTRAINT uq_messages_conversation_sequence UNIQUE (conversation_id, sequence_number),
    CONSTRAINT uq_messages_idempotency UNIQUE (conversation_id, sender_id, client_message_id)
);

CREATE INDEX idx_messages_conversation_sequence
    ON messages (conversation_id, sequence_number);
```

**Ghi chú vận hành:**

- `conversations.id`: sinh bằng UUIDv5(namespace, sorted(userA_id, userB_id)) — chỉ áp dụng 1-1 (v1). Namespace UUID cố định là chi tiết implementation, cấu hình 1 lần.
- `sequence_number`: gán trong transaction dùng `SELECT ... FOR UPDATE` trên row `conversations` tương ứng (Cơ chế A) — UNIQUE constraint ở đây là lưới an toàn cấu trúc, không phải cơ chế chính.
- `chk_messages_content_non_empty`: enforce invariant `MessageContent` non-empty (Section 9.4) ngay ở tầng DB, bổ sung cho validation application layer.
- Không có FK vật lý cho `user_id`/`sender_id` → bảng `users` (khác database, khác service).
- `recipient_verification_status`: mặc định `VERIFIED` (áp dụng cho participant tự tạo request, đã qua JWT). Participant còn lại nhận `PENDING`/`FAILED` theo kết quả gọi `GET /users/{userId}` lúc tạo conversation — xem `messaging-service-boundary.md` §4.0, §7.

---

## Realtime Service

Không có schema — không sở hữu persistent data. Xem `service-boundaries/realtime-service-boundary.md` §2 (connection registry chỉ tồn tại trong bộ nhớ).

---

## Lịch sử thay đổi

| Ngày | Service | Bảng/cột | Thay đổi | Nguồn |
| --- | --- | --- | --- | --- |
| 2026-08-25 | Identity | `users` | Tạo bảng ban đầu | #13, identity-service-boundary.md |
| 2026-08-25 | Messaging | `conversations`, `conversation_participants`, `messages` | Tạo 3 bảng ban đầu | #13, messaging-service-boundary.md |
| 2026-08-27 | Messaging | `conversation_participants.recipient_verification_status` | Thêm cột — hỗ trợ fail-open + lazy retry khi Identity Service down lúc tạo conversation | messaging-service-boundary.md §4.0, §7 |
