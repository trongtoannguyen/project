# Realtime Service — Boundary & Contract

> Nguồn quyết định: Decision #13 (PROJECT_CONTEXT.md). File này là chi tiết kỹ thuật triển khai từ #13, không phải quyết định độc lập — mọi thay đổi lớn về boundary/contract ở đây nên phản ánh ngược lại bằng 1 decision record mới nếu làm thay đổi trade-off đã chốt.

## 1. Trách nhiệm (bounded context)

**Không sở hữu domain aggregate nào.** Đây là transport/delivery mechanism thuần túy — giữ WebSocket connection sống, biết "user nào đang online ở đâu", và đẩy dữ liệu do Messaging Service cung cấp tới đúng client đang kết nối.

Nguồn sự thật (source of truth) của mọi dữ liệu vẫn là Messaging Service. Realtime Service không bao giờ tự quyết định nội dung/thứ tự message — chỉ relay.

## 2. Dữ liệu sở hữu

**Không có persistent domain data / không có database riêng.** Chỉ giữ state tạm thời trong bộ nhớ:

- Connection registry: `Map<userId, Connection>`.
- **Quyết định multi-tab/device (đã chốt):** 1 connection/user — connection mới ghi đè connection cũ. Đúng với working assumption "one session per user adequate for MVP" (PROJECT_CONTEXT.md Section 5). Nâng cấp lên `Map<userId, Set<Connection>>` là thay đổi cục bộ, không phải redesign, khi có yêu cầu multi-device thật.

Nếu sau này cần nhiều instance Realtime Service (multi-node), connection registry cần chuyển sang shared state (ví dụ Redis) — **chưa cần ở MVP** (1 instance, #4 constraint).

## 3. WebSocket handshake & authentication

- **Quyết định (đã chốt):** JWT gửi qua query param lúc mở connection — `wss://.../ws?token=<JWT>`.
  - Lý do chọn query param thay vì header: browser WebSocket API không hỗ trợ set custom header khi mở connection — đây là giới hạn kỹ thuật của web client (#3), không phải lựa chọn tùy ý.
  - Bắt buộc dùng WSS (TLS) để token không lộ trên đường truyền; cần đảm bảo access log của proxy/load balancer không ghi lại query string chứa token.
- Verify JWT (stateless, cùng shared key với Messaging Service — xem `identity-service-boundary.md` mục 4) **1 lần lúc handshake**. Nếu invalid → reject connection ngay, không cho mở.
- Không verify lại mỗi message sau đó — kết nối đã xác thực giữ nguyên trạng thái tới khi đóng.
- JWT hết hạn giữa chừng: connection vẫn tồn tại tới khi client tự đóng/reconnect. Giới hạn đã biết, chấp nhận ở v1.

## 4. Inbound contract (Messaging Service → Realtime Service)

Không phải API public — nội bộ giữa 2 service.

### 4.1 Message mới

Payload: `messageId`, `conversationId`, `senderId`, `recipientId`, `sequenceNumber`, `content`, `createdAt`.
Xử lý: tra `recipientId` trong connection registry → nếu có connection, push xuống qua WebSocket; nếu không (offline), no-op — không lưu "pending push" nào (REST sync ở Messaging Service là lưới an toàn).

### 4.2 Read-state thay đổi

Payload: `conversationId`, `readerId`, `lastReadSequence`.
Xử lý: tra participant còn lại (không phải `readerId`) trong connection registry → push nếu online, no-op nếu không.

### Nguyên tắc chung cho cả 2 loại event

- Nhận qua HTTP nội bộ, không cần message broker (#4 non-goals — chưa cần Kafka/RabbitMQ ở quy mô 1 instance/service).
- Nếu Realtime Service không nhận được (down) — không có hệ quả mất dữ liệu, vì Messaging Service không chờ/không retry; client tự phát hiện qua REST sync khi reload/reconnect.

## 5. Connection lifecycle

- **Heartbeat/ping-pong**: bắt buộc, để phát hiện "zombie connection" (client mất mạng đột ngột không đóng WebSocket đúng cách) và dọn khỏi registry sau timeout.
- Client tự chịu trách nhiệm reconnect khi phát hiện WebSocket đóng (client-side responsibility, hành vi WebSocket chuẩn).
- Realtime Service crash/restart: toàn bộ registry mất — chấp nhận được vì không phải domain data; client tự phát hiện và reconnect.

## 6. Dependency & failure mode

- Phụ thuộc Messaging Service **chỉ theo chiều nhận tín hiệu** (inbound), không có chiều ngược lại (Realtime không bao giờ gọi Messaging Service để lấy dữ liệu — mọi thứ cần thiết đã nằm trong payload event).
- Không phụ thuộc Identity Service runtime (stateless JWT verify, giống Messaging Service).
- Realtime Service down: không ảnh hưởng tính đúng đắn của dữ liệu (Messaging Service vẫn hoạt động độc lập); chỉ mất khả năng "live push" tạm thời.

## 7. Explicitly out of scope (v1)

- Cross-node socket routing / multi-instance broadcast (Section 3 loại trừ rõ).
- Presence (online/offline status hiển thị cho người khác) — chỉ có "connection tồn tại hay không" phục vụ mục đích routing nội bộ, không phải feature Presence công khai.
- Push notification khi offline (ngoài scope MVP).

## 8. Điều chưa quyết / để lại sau

- Cơ chế đóng connection chủ động khi JWT hết hạn giữa chừng (hiện để tồn tại tới khi client tự đóng).
- Shared connection registry (Redis) khi cần multi-instance — chỉ thiết kế khi có bottleneck/yêu cầu thật.
