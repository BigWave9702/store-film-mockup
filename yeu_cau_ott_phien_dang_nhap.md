# Yêu cầu cho OTT backend: xác thực và thu hồi phiên theo từng hệ thống

Tài liệu này mô tả những gì OTT backend cần làm để khớp với luồng phiên đăng nhập mới của CMS (quy tắc BR-11t, `thiet_ke_chi_tiet.md`). Cùng mô hình áp dụng cho Affiliate API; thay `OTT` bằng mã hệ thống tương ứng.

## 1. Nguyên tắc

1. **Mỗi hệ thống tự quản lý phiên và token của chính nó.** OTT có bản sao (replica) phiên / refresh token riêng, chỉ chứa phiên thuộc OTT. CMS giữ sổ phiên của mọi hệ thống nhưng chỉ lưu *định danh phiên* (không lưu token), và chỉ quản trị viên CMS xem được.
2. **Ghi bản sao của mình trước, rồi phát sự kiện Kafka.** Ghi phiên và dòng outbox trong cùng một transaction; tiến trình riêng đẩy outbox lên Kafka. Không phát sự kiện nếu chưa ghi được bản sao.
3. **Độc lập giữa các hệ thống.** Đăng nhập, làm mới, đăng xuất ở OTT không đụng tới phiên ở CMS hay Affiliate, và ngược lại.
4. **Ngoại lệ có chủ ý**: thay đổi của chính tài khoản (đổi / đặt lại mật khẩu, khóa, xóa) và thao tác thu hồi của quản trị viên CMS sẽ thu hồi phiên ở OTT qua sự kiện `authz.session.revoked`.
5. **Không gọi CMS mỗi yêu cầu.** CMS chỉ được gọi đúng một lần lúc đăng nhập (xác minh mật khẩu). Mọi yêu cầu sau đó OTT tự kiểm tra bằng token và bản sao của mình.
6. Thông tin kết nối (khóa dịch vụ, Kafka, tên topic, group) đặt trong cấu hình / biến môi trường, không ghi cứng.

## 2. Thông tin cần có từ CMS

| Mục | Giá trị |
| --- | --- |
| Mã hệ thống | `OTT` (đã có sẵn trong CMS) |
| Khóa dịch vụ | Quản trị viên CMS cấp bằng `POST /api/v1/cms/systems/{id}/service-key` (hiện một lần, dạng `sk.OTT.<secret>`); gửi trong header `X-System-Key` |
| Topic OTT **phát** | `session-events` (khóa thông điệp = `userId`) |
| Topic OTT **nhận** | `authz-events` (đã dùng cho quyền: `authz.user.access.changed`, `authz.definition.changed`; thêm `authz.session.revoked`) |
| Định dạng thời gian | ISO-8601 có offset, múi giờ GMT+7, ví dụ `2026-10-10T08:47:03.123+07:00`; phía nhận phải đọc đúng offset, không giả định `Z` |
| `userId` | id tài khoản trong CMS (OTT dùng chung tài khoản CMS đối với người dùng được cấp quyền ở OTT) |

## 3. Luồng đăng nhập

1. Client gọi endpoint đăng nhập của OTT với tên đăng nhập và mật khẩu.
2. OTT gọi CMS (một lần):

   ```
   POST {CMS}/internal/auth/verify
   X-System-Key: sk.OTT.<secret>
   Content-Type: application/json

   { "tenDangNhap": "alice", "matKhau": "********", "ip": "203.0.113.9" }
   ```

   `ip` là IP của người dùng cuối (để CMS giới hạn đăng nhập sai, dùng chung với CMS). Nếu bỏ qua, CMS dùng IP của OTT backend và có thể khóa nhầm mọi người dùng cùng lúc.

   Phản hồi `200`:

   ```json
   { "data": { "userId": 12, "tenDangNhap": "alice", "hoTen": "Alice",
               "access": { "heThong": "OTT", "userId": 12, "active": true,
                           "quyen": [ { "maQuyen": "ott.catalog.xem", "ten": "..." } ],
                           "menu": [ ] } } }
   ```

   | Mã | Ý nghĩa | OTT xử lý |
   | --- | --- | --- |
   | 401 `INVALID_CREDENTIALS` | Sai tên đăng nhập / mật khẩu, hoặc tài khoản bị khóa | Trả 401 cho client |
   | 403 `FORBIDDEN` | Tài khoản hợp lệ nhưng chưa được cấp quyền nào ở OTT | Trả 403 cho client |
   | 401 (khóa dịch vụ sai / hệ thống bị tạm ngưng) | Lỗi cấu hình OTT | Trả 503, cảnh báo vận hành, không thử lại vô hạn |
   | 429 `TOO_MANY_REQUESTS` | Vượt giới hạn đăng nhập sai | Trả 429 cho client |
   | 5xx / không kết nối được | CMS lỗi | Trả 503: **không** cho đăng nhập khi chưa xác minh được |

3. OTT tạo phiên **trong một transaction**:
   - `sessionId` ngẫu nhiên không đoán được (UUID v4), tối đa 64 ký tự. Đây là định danh dùng chung với CMS.
   - Bản ghi phiên trong bản sao của OTT (xem mục 6) kèm **hash** của refresh token (không lưu token thô).
   - Dòng outbox `session.started` (mục 5).
4. OTT cấp:
   - **Access token** ngắn hạn (khuyến nghị ≤ 15 phút), ký bằng khóa **của OTT**, claims tối thiểu: `iss` (OTT), `sub` (= `userId`), `sid` (= `sessionId`), `iat`, `exp`, `jti`.
   - **Refresh token** ngẫu nhiên (≥ 256 bit), dùng một lần và xoay vòng; chỉ lưu SHA-256.
5. Trả token cho client. Không dùng token do CMS cấp cho API của OTT, và không dùng JWKS của CMS để xác minh token của OTT.

## 4. Làm mới, đăng xuất, mỗi yêu cầu

**Làm mới (refresh)**
1. Tìm phiên theo hash refresh token (khóa dòng khi xử lý song song).
2. Nếu token đã dùng rồi (đã xoay) → coi là bị đánh cắp: kết thúc cả phiên, phát `session.ended` với `reason = REFRESH_REUSE_DETECTED`, trả 401.
3. Nếu phiên đã kết thúc hoặc hết hạn → 401.
4. Xoay token, cập nhật hạn mới, ghi outbox `session.refreshed` (cùng transaction), trả cặp token mới (giữ nguyên `sid`).

**Đăng xuất**
1. Đánh dấu phiên đã kết thúc (`ended`), thu hồi refresh token của phiên đó.
2. Ghi outbox `session.ended` với `reason = LOGOUT` (cùng transaction).
3. Chỉ phiên này bị ảnh hưởng; các phiên khác của cùng người dùng ở OTT và mọi phiên ở hệ thống khác giữ nguyên.

**Mỗi yêu cầu API**
1. Xác minh chữ ký, `iss`, `exp` của access token bằng khóa của OTT (offline).
2. Kiểm tra `sid` còn **đang hoạt động** trong bản sao của OTT (khuyến nghị đệm trong bộ nhớ / Redis, đệm hạn ngắn vài giây và xóa đệm ngay khi phiên bị kết thúc). Phiên đã `ended` thì từ chối dù token chưa hết hạn.
3. Kiểm tra quyền bằng bản sao quyền đã có (sự kiện `authz.user.access.changed`), không gọi CMS.

## 5. Sự kiện OTT phát lên `session-events`

Gửi ít nhất một lần, khóa Kafka = `userId` (giữ thứ tự theo người dùng), qua transactional outbox. Mọi thông điệp có các trường chung:

| Trường | Kiểu | Ghi chú |
| --- | --- | --- |
| `eventId` | string | Bắt buộc, duy nhất theo OTT (UUID); CMS dùng để chẩn đoán |
| `eventType` | string | `session.started`, `session.refreshed`, `session.ended` |
| `occurredAt` | string | Thời điểm phát sinh, ISO-8601 có offset |
| `origin` | string | `OTT` |
| `heThong` | string | `OTT` |
| `sessionId` | string | ≤ 64 ký tự |
| `userId` | number | Id tài khoản CMS (phải tồn tại, nếu không CMS bỏ qua thông điệp) |

```json
{ "eventId": "5b9c…", "eventType": "session.started", "occurredAt": "2026-10-10T08:47:03.123+07:00",
  "origin": "OTT", "heThong": "OTT", "sessionId": "e1b5…", "userId": 12,
  "startedAt": "2026-10-10T08:47:03.123+07:00", "expiresAt": "2026-10-17T08:47:03.123+07:00",
  "ip": "203.0.113.9", "userAgent": "OTT-Web/1.0" }
```

| `eventType` | Trường riêng | Khi nào |
| --- | --- | --- |
| `session.started` | `startedAt`, `expiresAt`, `ip`, `userAgent` | Sau khi đăng nhập thành công |
| `session.refreshed` | `expiresAt` (hạn mới) | Mỗi lần làm mới |
| `session.ended` | `endedAt`, `reason` | Đăng xuất, bị thu hồi, phát hiện dùng lại refresh token, hết hạn do OTT dọn |

`reason` là chuỗi chữ hoa gạch dưới (≤ 40 ký tự): `LOGOUT`, `REFRESH_REUSE_DETECTED`, `PASSWORD_CHANGED`, `PASSWORD_RESET`, `ACCOUNT_LOCKED`, `ACCOUNT_DELETED`, `REVOKED_BY_ADMIN`, `EXPIRED`. Giá trị lạ được CMS ghi là `UNKNOWN`.

CMS xử lý idempotent: gửi trùng hay sai thứ tự không làm sai; phiên đã `ended` không bao giờ mở lại. Thông điệp thiếu trường bắt buộc hoặc sai hệ thống / người dùng bị CMS bỏ qua (có ghi log), nên OTT cần kiểm thử hợp đồng này.

## 6. Bản sao phiên của OTT (gợi ý lược đồ)

Chỉ phiên của OTT. Tên cột tùy OTT, nội dung tối thiểu:

| Cột | Ghi chú |
| --- | --- |
| `session_id` (PK) | Trùng `sessionId` trong sự kiện |
| `user_id` | Id tài khoản CMS, có chỉ mục |
| `status` | `active` / `ended` |
| `end_reason`, `ended_at` | |
| `started_at`, `expires_at` | |
| `ip`, `user_agent` | |
| bảng token: `refresh_hash` (unique), `session_id`, `state` (`active`/`rotated`/`revoked`), `expires_at` | Chỉ lưu SHA-256 của refresh token |

Dọn bản ghi đã kết thúc / hết hạn sau một thời gian cấu hình được (đệm tối thiểu bằng hạn access token); quy tắc lưu trữ lâu dài do vận hành định sau.

## 7. Nhận lệnh thu hồi từ CMS (`authz-events`)

OTT đã (hoặc sẽ) tiêu thụ `authz-events` để giữ bản sao quyền. Thêm xử lý loại `authz.session.revoked`:

```json
{ "eventId": 1042, "eventType": "authz.session.revoked", "occurredAt": "2026-10-10T09:00:00.000+07:00",
  "userId": 12, "scope": "SESSION", "heThong": "OTT", "sessionId": "e1b5…",
  "reason": "REVOKED_BY_ADMIN", "revokedBeforeEpoch": 1791597600 }
```

| `scope` | `heThong` | OTT làm gì |
| --- | --- | --- |
| `SESSION` | `OTT` | Kết thúc đúng phiên `sessionId` và thu hồi refresh token của phiên |
| `ALL` | vắng mặt | Kết thúc **mọi** phiên đang hoạt động của `userId` ở OTT có thời điểm bắt đầu ≤ `revokedBeforeEpoch` (epoch giây), thu hồi mọi refresh token tương ứng |
| bất kỳ | khác `OTT` | Bỏ qua |

Sau khi kết thúc phiên (kể cả khi phiên đã kết thúc từ trước, để xác nhận), phát `session.ended` với `reason` lấy từ sự kiện. Khử trùng theo `eventId` (số tăng dần). Phiên mới tạo **sau** `revokedBeforeEpoch` không bị ảnh hưởng.

Ngoài ra khi nhận `authz.user.access.changed` có `active = false` cho `userId` của OTT, nên kết thúc mọi phiên của người dùng đó (CMS cũng phát `authz.session.revoked` trong các trường hợp khóa / xóa, nên đây chỉ là lớp bảo vệ thứ hai).

## 8. Yêu cầu phi chức năng

- **Độ trễ thu hồi**: cỡ giây (outbox CMS + Kafka). Thao tác nhạy cảm (đổi mật khẩu, thanh toán...) có thể đối chiếu bằng `POST /internal/authz/check` để có kết quả tức thì.
- **Kafka lỗi**: sự kiện nằm lại outbox và gửi bù theo thứ tự; đăng nhập vẫn thành công. Không được chặn đăng nhập vì Kafka.
- **CMS lỗi lúc đăng nhập**: từ chối đăng nhập (503), không tạo phiên.
- **Bảo mật**: không ghi mật khẩu / token thô vào log; chỉ gọi CMS qua kênh nội bộ có TLS; khóa dịch vụ lưu trong bí mật của hạ tầng; giới hạn tốc độ endpoint đăng nhập của OTT độc lập với CMS.
- **Idempotency**: xử lý `authz.session.revoked` lặp lại không gây lỗi; endpoint đăng xuất gọi lại trả thành công.
- **Đồng hồ**: dùng giờ máy chủ đồng bộ NTP; thời gian trong sự kiện theo GMT+7 kèm offset.

## 9. Tiêu chí nghiệm thu

1. Đăng nhập OTT thành công tạo đúng một dòng phiên ở OTT và CMS thấy phiên đó (màn "Phiên đăng nhập" của CMS, hệ thống `OTT`) trong vài giây.
2. Đăng xuất ở OTT: phiên OTT kết thúc, CMS ghi `ended / LOGOUT`; phiên cùng tài khoản ở CMS và các thiết bị OTT khác vẫn dùng được.
3. Đăng nhập CMS, đăng xuất CMS không ảnh hưởng phiên OTT.
4. Quản trị viên thu hồi một phiên OTT trên CMS: trong vài giây phiên đó không dùng được (kể cả access token chưa hết hạn), refresh bị từ chối, các phiên khác nguyên vẹn.
5. Đổi / đặt lại mật khẩu, khóa hoặc xóa tài khoản ở CMS: mọi phiên của tài khoản ở OTT bị kết thúc.
6. Dùng lại refresh token đã xoay: phiên bị kết thúc, CMS ghi `REFRESH_REUSE_DETECTED`.
7. Tắt Kafka rồi bật lại: sự kiện đăng nhập / đăng xuất phát bù đủ và đúng thứ tự, không trùng ảnh hưởng kết quả.
8. Sai mật khẩu quá giới hạn: bị CMS chặn (429); tài khoản chưa được cấp quyền ở OTT: nhận 403 và không có phiên.
