# Messaging Service — Boundary & Contract

> Nguồn quyết định: Decision #13 (PROJECT_CONTEXT.md). File này là chi tiết kỹ thuật triển khai từ #13, không phải quyết định độc lập — mọi thay đổi lớn về boundary/contract ở đây nên phản ánh ngược lại bằng 1 decision record mới nếu làm thay đổi trade-off đã chốt.

## 1. Trách nhiệm (bounded context)

Sở hữu 2 aggregate: `Conversation` + `ConversationParticipant`, và `Message` (xem PROJECT_CONTEXT.md Section 9.3). Cả hai cùng 1 service, cùng 1 database — quyết định có chủ đích (Phương án B, #13) để giữ invariant liên-aggregate (#6, #7, senderId-hợp-lệ) trong 1 transaction boundary duy nhất, tránh distributed transaction/saga ở giai đoạn chưa cần.

Không sở hữu dữ liệu `User` — chỉ giữ `userId` như reference thuần (foreign key logic, không FK vật lý cross-service).

## 2. Dữ liệu sở hữu (service-owned data)

| Bảng | Field chính | Ghi chú |
| --- | --- | --- |
| `Conversation` | `conversationId` (PK, UUID — deterministic cho 1-1, xem mục 3), `createdAt` | Immutable participant set sau khi tạo (#12) |
| `ConversationParticipant` | `(conversationId, userId)` composite, `lastReadSequence` | Tạo cùng lúc với `Conversation`, không join sau (v1) |
| `Message` | `messageId` (PK), `conversationId`, `senderId`, `sequenceNumber`, `clientMessageId`, `content`, `createdAt` | `conversationId`/`senderId` là reference thuần |

## 3. Invariant liên-aggregate và cơ chế enforce

Vì cùng DB, các invariant liên-aggregate (PROJECT_CONTEXT.md Section 9.5) được enforce bằng DB transaction/constraint nội bộ — không cần cross-service call:

| Invariant | Cơ chế |
| --- | --- |
| 1 cặp `(userA, userB)` chỉ có 1 `Conversation` (#6) | **Deterministic Conversation ID** — `conversationId = UUIDv5(namespace, sorted(userA_id, userB_id))`. Application layer sort 2 userId theo thứ tự cố định, nối chuỗi, sinh UUIDv5 (RFC 4122, name-based) làm `conversationId` trước khi insert. Vì cùng 1 cặp luôn cho ra cùng 1 UUID, "tạo mới" tự nhiên trở thành upsert trên chính PK `conversations.id` — PK constraint sẵn có tự chặn trùng, không cần cột phụ, không cần advisory lock riêng. **Chỉ áp dụng cho conversation 1-1 (v1).** Group chat (khi tới milestone đó) cần cơ chế sinh ID riêng — không kế thừa tự động, vì quy tắc nghiệp vụ "1 tổ hợp participant = 1 conversation duy nhất" chưa chắc đúng cho group (xem #13 revisit trigger). |
| `sequenceNumber` strictly increasing, không trùng, trong 1 `conversationId` (#7) | **Cơ chế A đã chốt:** `SELECT ... FOR UPDATE` trên row `Conversation` trong cùng transaction với insert `Message`. Lý do chọn: contention thấp (1-1 conversation, tối đa 2 writer), đơn giản, tự nhiên gap-free. Revisit khi group chat làm tăng contention thật sự. |
| `senderId` phải là participant hợp lệ của `conversationId` | JOIN nội bộ trong cùng transaction — không cần gọi Identity Service (JWT đã verify sẵn `userId`, chỉ cần check tồn tại trong `ConversationParticipant`) |

## 4. API bên ngoài (Client ↔ Messaging Service)

### 4.1 Gửi message — `POST /conversations/{conversationId}/messages`

Request: `{ "clientMessageId": "<uuid>", "content": "<text>" }`. `senderId` lấy từ JWT claim.

Thứ tự xử lý (1 transaction):

1. Verify participant hợp lệ.
2. Check `(conversationId, senderId, clientMessageId)`:
   - Tồn tại + cùng content → trả message gốc, **200 OK** (idempotent retry, #8).
   - Tồn tại + khác content → **409 Conflict**.
   - Chưa tồn tại → tiếp.
3. `SELECT FOR UPDATE` trên `Conversation`, gán `sequenceNumber` mới.
4. Insert `Message`, commit.
5. Sau commit: gọi Realtime Service async (Contract Realtime — mục 5), không chặn response.
6. Trả **200 OK**: `messageId`, `sequenceNumber`, `clientMessageId`, `content`, `createdAt`.

Error codes: `400` content rỗng, `403` không phải participant, `409` clientMessageId trùng khác content.

### 4.2 Sync lịch sử — `GET /conversations/{conversationId}/messages?sinceSequence=N`

- Verify participant hợp lệ trước khi trả data (authorization).
- Trả `sequenceNumber > sinceSequence`, sắp tăng dần.
- **Có giới hạn cố định mỗi lần gọi** (ví dụ 200 message/batch — con số cụ thể là chi tiết implementation). Client tự lặp gọi bằng `sequenceNumber` cuối cùng nhận được làm `sinceSequence` mới, dừng khi batch trả về ít hơn giới hạn.
- Edge case: nếu `sinceSequence` client gửi lớn hơn max sequence thực tế → dấu hiệu bất thường, log lại, cân nhắc trả lỗi rõ thay vì âm thầm trả rỗng.

### 4.3 Read-receipt — `PUT /conversations/{conversationId}/read`

Request: `{ "sequenceNumber": N }`. `userId` từ JWT.

1. Verify participant hợp lệ.
2. Validate `sequenceNumber` ≤ max sequence thực tế của conversation.
3. Update `ConversationParticipant.lastReadSequence` **chỉ khi giá trị mới > hiện tại** (monotonic, #9) — nếu không, no-op, không lỗi.
4. Commit.
5. Nếu có thay đổi thực sự (không phải no-op): gọi Realtime Service async, báo `conversationId`, `readerId`, `lastReadSequence` mới.
6. Trả 200 OK với `lastReadSequence` hiện tại.

## 5. Dependency & giao tiếp với Realtime Service (outbound)

- **Loại giao tiếp:** HTTP nội bộ, fire-and-forget, timeout ngắn. Không dùng message broker (Kafka/RabbitMQ) — chưa có yêu cầu multi-instance Realtime Service để cần pub/sub thật (#4 non-goals).
- **Nguyên tắc bắt buộc:** gọi Realtime Service **chỉ sau khi transaction DB đã commit thành công**, không bao giờ chặn response cho client vì lý do Realtime Service chậm/down.
- 2 sự kiện outbound:
  - Message mới tạo → payload: `messageId`, `conversationId`, `senderId`, `recipientId`, `sequenceNumber`, `content`, `createdAt`.
  - Read-state thay đổi → payload: `conversationId`, `readerId`, `lastReadSequence`.
- Nếu call fail/timeout: log warning, không retry, không rollback. REST sync (mục 4.2) là lưới an toàn cuối cùng.

## 6. Dependency vào Identity Service

- **Không có runtime call.** Xác thực JWT stateless (xem `identity-service-boundary.md` mục 4) — chỉ cần shared public key/secret cấu hình lúc deploy.
- Nếu Identity Service down, Messaging Service vẫn hoạt động bình thường cho user đã có token hợp lệ.

## 7. Failure modes tổng hợp (luồng gửi message)

| Tình huống | Xử lý |
| --- | --- |
| Client retry do timeout | Idempotency key (`clientMessageId`) đảm bảo không tạo trùng |
| 2 message cùng lúc, cùng conversation | Serialize bằng `SELECT FOR UPDATE` |
| Service crash trước commit | DB tự rollback, không cần xử lý thêm |
| Service crash sau commit, trước khi gọi Realtime | Message không mất — REST sync xử lý; client retry với cùng `clientMessageId` nếu response bị mất cũng an toàn (idempotent) |
| Realtime Service down khi gọi báo tin | Chấp nhận được — không retry, REST sync là lưới an toàn |
| `sinceSequence` bất thường khi sync | Log lại, cân nhắc trả lỗi rõ thay vì âm thầm trả rỗng |

## 8. Điều chưa quyết / để lại sau

- Con số cụ thể cho page size của sync lịch sử (implementation detail).
- Cơ chế xử lý khi group chat làm tăng contention trên `SELECT FOR UPDATE` (revisit khi có bottleneck thật, theo #2 nguyên tắc làm việc).
- Giá trị cụ thể của UUIDv5 namespace (1 UUID cố định của app, sinh 1 lần, dùng lại mọi lần hash — chi tiết implementation, không phải quyết định kiến trúc).
- Cơ chế sinh `conversationId` cho group chat — quyết định khi tới milestone đó, phụ thuộc quy tắc nghiệp vụ lúc đó: nếu "1 tổ hợp participant = 1 conversation duy nhất" vẫn đúng cho group, deterministic UUIDv5 mở rộng tự nhiên (hash cả participant set đã sort); nếu group cho phép nhiều conversation trùng participant set (giống Slack group DM), cần cơ chế khác (UUID random, hoặc hash + discriminator) — không kế thừa tự động từ cơ chế 1-1.
