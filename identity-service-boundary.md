# Identity Service — Boundary & Contract

> Nguồn quyết định: Decision #13 (PROJECT_CONTEXT.md). File này là chi tiết kỹ thuật triển khai từ #13, không phải quyết định độc lập — mọi thay đổi lớn về boundary/contract ở đây nên phản ánh ngược lại bằng 1 decision record mới nếu làm thay đổi trade-off đã chốt.

## 1. Trách nhiệm (bounded context)

Sở hữu domain `User` — identity, authentication, và không gì khác. Không biết gì về `Conversation`, `Message`, hay bất kỳ khái niệm messaging nào.

## 2. Dữ liệu sở hữu (service-owned data)

> DDL đầy đủ: xem `DATABASE_SCHEMA.md` § Identity Service.

Tương ứng aggregate `User` (xem PROJECT_CONTEXT.md Section 9.1):

| Field | Nguồn gốc | Ghi chú |
| --- | --- | --- |
| `userId` | Identity | PK |
| `email` | VO `Email` | Normalized, unique toàn hệ thống, immutable (#5) |
| `passwordHash` | — | Không lưu plaintext |
| `createdAt` | — | |

Không có bảng nào khác. Không JOIN, không lưu reference tới `conversationId`/`messageId`.

## 3. API bên ngoài

- `POST /auth/register` — chỉ chấp nhận email nằm trong allowlist (#5), không có invitation-email workflow.
- `POST /auth/login` — trả JWT access token đã ký (xem mục 4).
- Không có password-reset, MFA, OAuth trong v1 (Section 3, MVP acceptance boundary).
- `GET /users/{userId}` — trả thông tin cơ bản (`userId`, `email`, `createdAt`). Dùng bởi Messaging Service để verify `recipientUserId` tồn tại lúc tạo conversation (xem mục 5.1). Yêu cầu JWT hợp lệ (service-to-service call vẫn mang token, hoặc dùng service credential riêng — chi tiết implementation).

## 4. JWT issuance & verification contract

- **Quyết định:** stateless verify (Cách 1, đã chốt trong #13) — Identity Service ký JWT bằng shared secret/private key; Messaging Service và Realtime Service tự verify bằng public key/shared secret, **không gọi lại Identity Service mỗi request**.
- Claim tối thiểu trong JWT: `userId`, `exp`. Không cần nhúng email hay dữ liệu khác nếu service khác không cần.
- Hệ quả đã chấp nhận: **không có revocation tức thời** ở v1 (logout ngay lập tức, khóa tài khoản tức thời không khả thi trực tiếp — chỉ hết hạn tự nhiên theo `exp`). Đây là trade-off đã ghi nhận, revisit nếu có yêu cầu revocation thật sự (xem #13 revisit trigger).
- Key distribution: cấu hình lúc deploy (secret/public key giống nhau ở mọi service verify), không đồng bộ runtime.

## 5. Dependency & failure mode

- Identity Service **không phụ thuộc runtime vào bất kỳ service nào khác**.
- Nếu Identity Service down: user hiện tại (đã có token hợp lệ) **không bị ảnh hưởng** ở Messaging/Realtime Service — đây chính là lý do chọn stateless verify. Chỉ đăng nhập mới bị chặn.
- **Dependency inbound mới (5.1):** Messaging Service gọi `GET /users/{userId}` lúc tạo conversation mới (xem `messaging-service-boundary.md` §4 — `POST /conversations`). Đây là dependency **hẹp phạm vi, không phải hot path** — chỉ xảy ra khi tạo conversation lần đầu giữa 1 cặp user, không xảy ra ở gửi/nhận tin nhắn. Nếu Identity Service down lúc này, Messaging Service fail-open (vẫn tạo conversation, đánh dấu `PENDING`, retry lười ở lần tương tác kế tiếp) — không chặn người dùng, không tạo cascading failure trên hot path.

## 6. Điều chưa quyết / để lại sau

- Cơ chế revocation (nếu cần sau này) — không thiết kế trong v1, ghi nhận là hybrid approach khả dĩ (short-lived token + refresh, hoặc blacklist cache riêng) khi có yêu cầu thật.
- Key rotation strategy — chưa cần ở quy mô 1 secret/1 lần cấu hình của MVP.
