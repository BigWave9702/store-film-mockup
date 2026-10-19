# **THIẾT KẾ CHI TIẾT HỆ THỐNG**

Tài liệu này trình bày **kiến trúc chi tiết** của một nền tảng OTT theo yêu cầu, gồm các thành phần: **OTT Backend (API phục vụ người dùng)**, **CMS Platform**, **Affiliate/Publisher Platform**, **Media Orchestrator** (adapter cho Transcoder/DRM/Storage), sử dụng **Bitmovin** làm Transcode, **EZDRM** làm DRM, **Cloudflare R2** làm lưu trữ và **Cloudflare CDN**.

Mục tiêu là tách biệt rõ **boundary giữa các domain**, tránh khóa với nhà cung cấp (vendor lock-in) bằng tầng adapter và cấu hình linh hoạt, đồng thời lên lộ trình triển khai từng giai đoạn

Tài liệu này sẽ:

-   phân tích luồng nghiệp vụ chính (upload→ingest→encode→(DRM)→verify→publish→ playback)
-   thiết kế database và API cho từng subsystem,
-   chi tiết chiến lược vận hành (scaling, backup, bảo mật, cost vs ops)

# 1\. Tổng quan hệ thống

## 1.1. Phạm vi hệ thống

Hệ thống chia thành bốn phân hệ, mỗi phân hệ có cơ sở dữ liệu riêng và không truy vấn trực tiếp CSDL của phân hệ khác. Trao đổi giữa các phân hệ đi qua REST API, webhook và Event Bus (Kafka).

| **Phân hệ** | **Mã CSDL** | **Trách nhiệm dữ liệu** | **Công nghệ** |
| --- | --- | --- | --- |
| **OTT Backend (User API)** | ott\_db | Tài khoản, hồ sơ, thiết bị, gói cước, đăng ký, thanh toán, quyền xem, phiên phát, lịch sử xem | Java 21, Spring Boot, PostgreSQL, Redis |
| **CMS Platform** | cms\_db | Nội dung, metadata, phiên bản, phụ đề, ảnh bìa, quy trình duyệt, lịch phát hành, bản quyền, RBAC nội bộ | React/Angular admin, Spring Boot, PostgreSQL |
| **Affiliate Platform** | affiliate\_db | Nhà xuất bản, chiến dịch, link theo dõi, click, quy gán nguồn, chuyển đổi, quy tắc và giao dịch hoa hồng | Spring Boot, PostgreSQL |
| **Media Orchestrator** | media\_db | Tài nguyên media, tệp nguồn, công việc encode/DRM, công việc nhà cung cấp, callback, đối tượng lưu trữ, manifest đầu ra | Spring Boot worker, PostgreSQL, queue |

Giữa các domain:

-   **REST API / Webhook**: CMS ↔ Media API ↔ OTT API (chỉ thông tin khởi tạo/hoàn thành media, không stream video); Affiliate API ↔ OTT API (qua event Kafka, không truy vấn DB trực tiếp).
-   **Event Bus (Kafka)**: Lan truyền sự kiện (payment, subscription, conversion, playback) giữa OTT↔Affiliate↔Analytics.

Các topic Kafka chính: user-events, payment-events, affiliate-events, media-events, cms-events (CMS phát content.published / content.unpublished / content.updated và **language.changed** qua transactional outbox để OTT cập nhật; language.changed mang toàn bộ danh mục ngôn ngữ dùng để OTT giữ bảng ngon\_ngu\_tham\_chieu, kể cả dòng đã xóa mềm với daXoa = true, gộp khi chưa gửi), authz-events (CMS phát thay đổi phân quyền để OTT và Affiliate giữ bản sao quyền cục bộ, xem BR-11r), ott-config-events (OTT Backend quản trị phát thay đổi cấu hình gói cước, giá, tính năng, vùng giá để OTT User API làm mới bộ nhớ đệm, xem BR-28 đến BR-34).

## 1.2. Phân quyền theo tác nhân

Người dùng cuối đăng nhập vào OTT (ott\_db.nguoi\_dung). Nhân sự nội bộ và nhà cung cấp nội dung đăng nhập vào CMS (cms\_db.tai\_khoan\_he\_thong). Nhà xuất bản đăng nhập vào Affiliate (affiliate\_db.nha\_xuat\_ban). Ba domain này không dùng chung bảng tài khoản.

Phân quyền nội bộ CMS theo mô hình RBAC (Role-Based Access Control): một tài khoản có nhiều vai trò, mỗi vai trò gắn nhiều quyền hạn.

**Phân quyền tập trung nhiều hệ thống.** CMS là nơi định nghĩa và cấp quyền cho nhân sự nội bộ dùng cả ba hệ thống: CMS, OTT backend và Affiliate API (khu quản trị của hai hệ thống sau). Họ đăng nhập một lần ở CMS (cùng bảng tai\_khoan\_he\_thong) và nhận JWT ES256 do CMS cấp; vai trò có thể chứa quyền của nhiều hệ thống. Người dùng cuối của OTT và nhà xuất bản của Affiliate vẫn nằm ở bảng tài khoản riêng của hai hệ thống đó, không thuộc phạm vi này. Mỗi quyền hoặc menu thuộc đúng một hệ thống (bảng he\_thong). Xem BR-11q. Riêng OTT Backend quản trị có cổng đăng nhập riêng theo BR-11t: gọi CMS một lần để xác minh mật khẩu rồi tự tạo phiên và tự cấp access token / refresh token của mình (BR-29); không dùng JWT do CMS cấp cho API của OTT.

| **Tác nhân** | **Dữ liệu** | **Quyền chính** |
| --- | --- | --- |
| Người dùng cuối | ott\_db | Đăng ký gói, thanh toán, phát video theo quyền xem, quản lý thiết bị và hồ sơ |
| Nhà cung cấp nội dung | cms\_db | Tạo nội dung, tải video gốc lên R2, gửi QC, xóa nội dung của mình khi còn nháp, xem trạng thái duyệt của nội dung do mình cung cấp. Chỉ truy cập hai nhóm menu **Nội dung** và **Workflow**, dữ liệu giới hạn theo nhà cung cấp của tài khoản; không có quyền duyệt, xuất bản, gỡ xuất bản |
| Biên tập viên | cms\_db | Sửa metadata, phụ đề, ảnh bìa, gán danh mục |
| Kiểm duyệt viên | cms\_db | Duyệt hoặc trả lại nội dung ở bước QC, khởi tạo job transcode/DRM |
| Quản trị viên CMS | cms\_db | Toàn quyền nội dung, cấu hình quy trình, quản lý tài khoản và vai trò |
| Nhà xuất bản | affiliate\_db | Tạo chiến dịch, sinh link theo dõi, xem báo cáo click và hoa hồng của mình |
| Quản trị Affiliate | affiliate\_db | Duyệt nhà xuất bản, cấu hình quy tắc hoa hồng, đối soát chi trả |
| Tài khoản dịch vụ | Nội bộ | Media worker, Kafka consumer, tác vụ đối soát; xác thực bằng khóa dịch vụ, không có phiên đăng nhập người dùng |

Nguyên tắc đặc quyền tối thiểu áp dụng cho cả quyền truy cập R2.

-   CMS dùng khóa chỉ ghi được bucket original-private.
-   Media API dùng khóa đọc original-private và ghi streaming-public.
-   Không khóa nào có quyền trên cả hai chiều của bucket không liên quan

# 2\. Quy tắc nghiệp vụ (BR)

***Tài khoản và gói :***

-   BR-01: Gói
    -   Hệ thống có các gói mặc định *FREE, VIP, VIP\_PRO*.
    -   *goi\_cuoc* chỉ định nghĩa gói và tính năng, không lưu trực tiếp giá và thời hạn. Giá và thời hạn được quản lý tại *gia\_goi\_cuoc*.
-   BR-02: Giá gói cước
    -   Một gói có thể có nhiều mức giá theo vùng giá và thời hạn. Việt Nam sử dụng VN/VND; các thị trường khác sử dụng GLOBAL/USD. *gia\_goi\_cuoc* không lưu *don\_vi\_tien*; tiền tệ được xác định từ *vung\_gia*.
    -   Giá bán được xác định từ *gia\_goc* và *phan\_tram\_khuyen\_mai*. Không được có hai mức giá cùng *ma\_goi\_cuoc + ma\_vung\_gia + so\_thang* cùng hiệu lực.
-   BR-03: Đăng ký gói
    -   Người dùng có thể chọn bất kỳ thời hạn nào được cấu hình cho gói.
    -   Khi tạo đăng ký, hệ thống phải lưu snapshot giá tại thời điểm mua gồm: *ma\_gia\_goi\_cuoc, so\_thang, gia\_goc, phan\_tram\_khuyen\_mai, gia\_ban, don\_vi\_tien*. Thay đổi bảng giá sau này không làm thay đổi đăng ký đã mua.
-   BR-04: Thanh toán
    -   *thanh\_toan* lưu số tiền và tiền tệ thực tế của giao dịch.
    -   Số tiền thanh toán không được lấy lại từ *gia\_goi\_cuoc* sau khi giao dịch đã phát sinh.
-   BR-05: Đa ngôn ngữ gói cước
    -   Tên/mô tả gói được quản lý tại *goi\_cuoc\_da\_ngon\_ngu*.
    -   Tên/mô tả tính năng được quản lý tại *tinh\_nang\_da\_ngon\_ngu*.
    -   Không đưa nội dung đa ngôn ngữ vào gia\_goi\_cuoc.
    -   Ngôn ngữ được tham chiếu tới bảng dùng chung *ngon\_ngu*.
-   BR-06: Kế thừa quyền gói
    -   Gói có cấp độ quyền truy cập; gói cấp cao hơn được hưởng quyền của các gói cấp thấp hơn.
    -   Mặc định: *VIP\_PRO ⟶ VIP ⟶ FREE*.
    -   Việc xác định nội dung nào thuộc gói nào phải được thể hiện bằng chính sách quyền xem, không hard-code trong API.
-   BR-07: Quyền FREE.
    -   Với phim lẻ: FREE được xem theo chất lượng tối đa được cấu hình cho FREE.
    -   Với series: FREE chỉ được xem số tập đầu tiên theo cấu hình nghiệp vụ, ví dụ 1 hoặc 2 tập.
    -   Các giới hạn này phải được cấu hình, không hard-code cố định trong backend

***Quyền xem và phát video:***

-   BR-08: Người dùng phát được một nội dung khi có đăng ký *DANG\_HOAT\_DONG* phủ nội dung đó, hoặc có bản ghi *quyen\_xem* còn hạn (*den\_han* lớn hơn thời điểm hiện tại).
-   BR-09: Phát video kiểm tra giới hạn lãnh thổ. Nếu *ban\_quyen\_noi\_dung* giới hạn lãnh thổ, vùng của người dùng phải nằm trong danh sách cho phép.
-   BR-10: OTT API cấp phiên phát (*phien\_phat*) kèm play token HMAC/JWT có thời hạn. Cloudflare kiểm token trước khi phục vụ manifest. Token hết hạn thì phiên không phát tiếp.

***Nội dung và quy trình:***

-   BR-11: Nội dung đi qua các trạng thái *BAN\_NHAP*, *CHO\_QC*, *SAN\_SANG*, *DA\_XUAT\_BAN*, *GO\_XUAT\_BAN*. Chỉ nội dung *DA\_XUAT\_BAN* và trong khoảng lịch phát hành mới hiển thị ở OTT.
-   BR-11a: Nội dung đã từng xuất bản (*da\_tung\_xuat\_ban = true*) không được xóa (kể cả xóa mềm); chỉ được chuyển sang gỡ xuất bản (*GO\_XUAT\_BAN*, ẩn với người dùng OTT) và kích hoạt lại (*GO\_XUAT\_BAN → DA\_XUAT\_BAN*); không được quay về *BAN\_NHAP*. Nhà cung cấp được gửi nội dung của mình sang *CHO\_QC*, nhưng không được duyệt hoặc xuất bản.
-   BR-11b: Loại nội dung (phim lẻ, series, tập phim) và nhà cung cấp không đổi sau khi tạo. Tập phim bắt buộc thuộc một series và một mùa của series đó, nhà cung cấp của tập luôn theo series; không có hai tập cùng số thứ tự trong một mùa.
-   BR-11c: Nhà cung cấp chỉ xem được nội dung của chính mình và chỉ sửa khi nội dung còn *BAN_NHAP*; từ *CHO_QC* trở đi chỉ tài khoản nội bộ được sửa. Nhà cung cấp được xóa nội dung của mình khi còn *BAN_NHAP* (xem BR-11e).
-   BR-11d: Mô tả, tiêu đề và các trường văn bản của nội dung là văn bản thuần; API từ chối thẻ HTML và ký tự điều khiển, client chịu trách nhiệm escape khi hiển thị.
-   BR-11e: Xóa nội dung. Nội dung đã từng xuất bản không xóa được (BR-11a). Nội dung nháp chưa từng gửi duyệt (*BAN\_NHAP*, lịch sử quy trình chỉ có dòng tạo) được xóa hẳn khỏi cơ sở dữ liệu, nhà cung cấp cũng được xóa nội dung của mình ở trạng thái này. Nội dung đã qua bước duyệt nhưng chưa từng xuất bản được xóa mềm để giữ dấu vết. Mọi trường hợp xóa đều dọn toàn bộ tệp của nội dung trên Cloudflare R2 (tiền tố c{id}/) và yêu cầu Media API xóa tài nguyên media; việc dọn tệp được ghi cùng transaction (bảng viec\_don\_dep), thực hiện sau commit và tự thử lại khi R2 hoặc Media tạm lỗi.
-   BR-11f: Tệp (video gốc, phụ đề, ảnh bìa) chỉ được tải lên R2 bằng presigned URL do CMS cấp. Backend chỉ sinh khóa tệp, cấp URL có hạn và xác nhận bằng HEAD; không nhận nội dung tệp qua API (không có endpoint multipart).
-   BR-11h: Thay tệp. Khi nội dung đã được QC "Trả lại" (về *BAN\_NHAP*) và nhà cung cấp tải lại video gốc cho một phiên bản, hệ thống xóa video cũ trên R2, yêu cầu Media API xóa tài nguyên media cũ và đặt lại trạng thái media của phiên bản (xóa manifest, chất lượng, phiên bản phục vụ) để phải transcode lại. Thay hoặc xóa phụ đề / ảnh bìa cũng xóa tệp cũ trên R2. Không thay được video gốc khi nội dung đang ở *CHO\_QC* trở đi hoặc khi phiên bản đang được xử lý media. Lượt tải lên bị bỏ dở (đã cấp URL nhưng không bao giờ xác nhận) được dọn sau thời hạn URL cộng 24 giờ.
-   BR-11i: Thời hạn presigned URL. Video gốc: mặc định 6 giờ (cấu hình `R2_PRESIGN_TTL_VIDEO`); phụ đề và ảnh bìa: 15 phút (`R2_PRESIGN_TTL`). URL gắn với đúng một khóa tệp, kích thước và loại nội dung đã khai báo. Xác nhận (`complete`) vẫn được chấp nhận nếu tệp đã có trên R2 với đúng kích thước dù URL đã hết hạn lúc tải xong. Một lần PUT tối đa 5 GiB; tệp lớn hơn (hoặc cần tải lại theo phần) dùng tải lên nhiều phần (BR-11l).
-   BR-11j: Luồng trạng thái: *BAN\_NHAP → CHO\_QC* (gửi QC, chủ nội dung), *CHO\_QC → BAN\_NHAP* (trả lại, bắt buộc ghi chú, chỉ QC/ADMIN), *CHO\_QC → SAN\_SANG* (duyệt), *SAN\_SANG → DA\_XUAT\_BAN*, *DA\_XUAT\_BAN ↔ GO\_XUAT\_BAN*. Đã duyệt (*SAN\_SANG*) thì không trả lại về bản nháp. Điều kiện gửi QC: có thông tin nội dung, ít nhất một bản dịch, và (phim lẻ/tập phim) có phiên bản đã tải video nguồn; series không có media nên không cần. Duyệt, xuất bản và kích hoạt lại với phim lẻ/tập phim cần phiên bản mặc định đã transcode xong và có tệp đầu ra. Xuất bản và kích hoạt lại còn cần: bản dịch tiếng Việt, poster mặc định, chính sách quyền xem cấp FREE, ít nhất một lịch phát hành, bản quyền còn hiệu lực; tập phim chỉ xuất bản khi series của nó đang *DA\_XUAT\_BAN*.
-   BR-11k: Tài khoản nhà cung cấp và biên tập viên nội bộ chỉ thao tác (ghi, chuyển bước) trên nội dung do chính mình tạo; QC và quản trị có quyền `noi_dung.tat_ca` thao tác trên mọi nội dung. Lịch phát hành của cùng một nội dung không được chồng lấn; nội dung đang xuất bản phải giữ ít nhất một lịch phát hành và chính sách FREE. Mỗi lần xuất bản, gỡ, kích hoạt lại phát một sự kiện qua outbox; sửa nội dung đang xuất bản phát `content.updated` (gộp thành một sự kiện chờ gửi).
-   BR-11l: Tải lên nhiều phần (multipart) cho video gốc, tối đa 1 TiB. Backend vẫn không nhận nội dung tệp: `POST /uploads/multipart` tạo phiên trên R2 và sinh khóa tệp; kích thước phần do máy chủ chọn (mặc định 64 MB, `R2_MULTIPART_PART_SIZE`, tối thiểu 5 MiB, nâng lên để không quá 10.000 phần) nên client không chọn được. `POST /uploads/{id}/parts` cấp URL PUT cho từng phần (tối đa 100 phần mỗi lần, hạn 15 phút, độ dài từng phần được ký vào URL); `GET /uploads/{id}/parts` cho biết phần đã lên R2 để tiếp tục sau khi bị ngắt. `POST /uploads/{id}/complete` do CMS tự liệt kê phần trên R2 (không tin danh sách từ client), kiểm tra đủ phần và đúng kích thước rồi mới yêu cầu R2 ghép; gọi lại an toàn. `DELETE /uploads/{id}` hủy lượt tải chưa xác nhận. Phiên bỏ dở quá `R2_MULTIPART_SESSION_TTL` (mặc định 3 ngày) cộng 24 giờ, hoặc nội dung bị xóa, được hủy trên R2 qua hàng đợi dọn dẹp (loại MULTIPART) vì liệt kê đối tượng không thấy các phần chưa ghép. Quy tắc gắn video vào phiên bản, thay video và xóa tệp cũ giống tải một lần.
-   BR-11m: Chống chịu vận hành. (1) Phiên bản ở trạng thái "đang xử lý" quá `MEDIA_STUCK_AFTER` (mặc định 24 giờ) mà không có callback được tự đánh dấu lỗi kèm lý do để QC bấm Transcode lại; callback đến muộn sau đó bị bỏ qua. (2) Sự kiện outbox không dựng được thông điệp (dữ liệu hỏng, không phải lỗi Kafka) bị thử 10 lần rồi chuyển sang "bỏ cuộc" để không chặn hàng đợi; lỗi Kafka tạm thời thử lại vô hạn và giữ thứ tự. (3) Mọi thời điểm trong snapshot gửi OTT ở UTC (phiên CSDL đặt `TIME ZONE UTC`).
-   BR-11n: Giới hạn lớp HTTP. Thân yêu cầu tối đa 1 MB (vượt thì 413; tệp không bao giờ đi qua API). Mỗi tài khoản đã xác thực tối đa 600 yêu cầu mỗi phút (`RATE_LIMIT_PER_MINUTE`, Redis, cửa sổ cố định; vượt thì 429 kèm Retry-After; lỗi Redis thì bỏ qua giới hạn); đăng nhập có bộ giới hạn riêng. Header an toàn: nosniff, X-Frame-Options DENY, Referrer-Policy no-referrer, HSTS, CSP `default-src 'none'`. Sau reverse proxy phải đặt `FORWARD_HEADERS_STRATEGY=framework` để IP client (giới hạn đăng nhập) là IP thật.
-   BR-11o: Quan sát. Cổng quản trị riêng (`MANAGEMENT_PORT`, không expose ra Internet) cung cấp health (liveness, readiness gồm Postgres và Redis) và `/actuator/prometheus`. Chỉ số nghiệp vụ: `cms_outbox_pending`, `cms_outbox_oldest_age_seconds`, `cms_outbox_dead`, `cms_purge_pending`, `cms_purge_abandoned`, `cms_media_processing`, `cms_media_failed`, `cms_uploads_open`, `cms_workflow_transitions_total{action}`. Quy tắc cảnh báo ở `cms_api/deploy/prometheus/cms-api-alerts.yml`.
-   BR-11p: Hai màn hình tổng hợp xem chéo mọi nội dung (menu "Lịch phát hành" và "Bản quyền nội dung", chỉ dành cho ADMIN và QC) là **chỉ đọc**: `GET /release-schedules` (lọc q theo tiêu đề, contentId, status 0 chờ / 1 đang hiệu lực / 2 hết, from, to) và `GET /copyrights` (lọc q, contentId, territory, status). Trạng thái tính theo thời điểm hiện tại. Lãnh thổ rỗng nghĩa là áp dụng mọi quốc gia nên cũng khớp mọi `territory`. Thêm, sửa, xóa vẫn làm trên từng nội dung (`/contents/{id}/release-schedules`, `/contents/{id}/copyrights`) để mọi thay đổi đi qua cùng quy tắc (chống chồng lấn, giữ lịch cho nội dung đang xuất bản, sự kiện cho OTT). Nhà cung cấp chỉ thấy nội dung của mình.
-   BR-11q: Phân quyền tập trung nhiều hệ thống. (1) Mỗi quyền / menu thuộc một hệ thống; mã quyền duy nhất toàn cục, hệ thống khác CMS phải dùng tiền tố `{mã hệ thống thường}.`. (2) Quyền và menu tạo cây cha-con trong cùng hệ thống; menu chỉ nằm dưới menu; quyền API nằm dưới menu hoặc nhóm quyền. (3) Quyền API khai báo (http\_method, mau\_duong\_dan) để hệ thống khác kiểm tra theo đường dẫn; mẫu theo cú pháp PathPattern của Spring (`{id}`, `*`, `**`). (4) Quyền mới tạo tự cấp cho vai trò ADMIN; chống nâng quyền áp dụng cho mọi hệ thống (chỉ cấp được quyền mình có). (5) **Frontend** của từng hệ thống (đăng nhập qua CMS như mọi nhân sự) gọi `GET /authz/menus?system=` (cây menu được truy cập: nút `truyCap=false` chỉ là nhóm chứa mục con hoặc quyền được cấp), `GET /authz/permissions?system=` và `POST /authz/check`; chỉ trả lời về chính người dùng trong token. (6) **Backend** xác minh JWT bằng `GET /.well-known/jwks.json` (ES256, `kid`) rồi quyết định quyền cục bộ theo BR-11r; API đồng bộ `POST /internal/authz/check` kèm khóa dịch vụ chỉ là dự phòng và cho thao tác cần kết quả tức thì (header `X-System-Key`, dạng `sk.{MÃ}.{bí mật}`, chỉ lưu SHA-256, cấp lại thì khóa cũ mất hiệu lực); mỗi khóa chỉ hỏi được hệ thống của nó. `GET /internal/authz/users/{id}/access` trả toàn bộ quyền và cây menu. (7) Kiểm tra theo ba dạng: mã quyền, (method, đường dẫn) hoặc đường dẫn menu; **mặc định từ chối**; được phép khi có ít nhất một quyền khớp mà người dùng được cấp. Lý do từ chối: NOT\_GRANTED, UNKNOWN\_PERMISSION, NO\_MATCHING\_ROUTE, NO\_MATCHING\_MENU, SYSTEM\_DISABLED, USER\_INACTIVE. (8) Hệ thống tạm ngưng: menu 403, mọi kiểm tra bị từ chối, khóa dịch vụ bị từ chối. (9) Quyền của người dùng nạp từ Redis (5 phút) và bị xóa khi vai trò / quyền / tài khoản đổi; chỉ mục đường dẫn lưu đệm 30 giây trên mỗi instance. JWT chỉ mang định danh (15 phút); khóa tài khoản có hiệu lực ngay ở API kiểm tra, còn backend chỉ xác minh chữ ký sẽ thấy sau tối đa 15 phút nên cần gọi API kiểm tra với thao tác nhạy cảm.
-   BR-11s: Múi giờ. Mọi thời gian trong hệ thống hiển thị theo **GMT+7** (`Asia/Ho_Chi_Minh`, cấu hình `APP_TIME_ZONE`): JSON của API, thông điệp Kafka (`occurredAt`, thời gian trong snapshot), log và múi giờ phiên PostgreSQL, dạng ISO-8601 có offset (ví dụ `2026-10-10T08:47:03.123+07:00`). Dữ liệu lưu bằng `timestamptz` (thời điểm tuyệt đối) nên không phụ thuộc múi giờ hiển thị; hệ thống nhận (OTT, Affiliate, Media API) phải phân tích đúng offset, không giả định `Z`. JWT dùng epoch giây nên không bị ảnh hưởng.
-   BR-11t: Phiên đăng nhập theo từng hệ thống (thay mô hình dùng chung JWT do CMS cấp). **Mỗi hệ thống (CMS, OTT, Affiliate) tự quản lý phiên và token của chính nó**: có bản sao (replica) phiên / refresh token riêng chỉ chứa phiên thuộc hệ thống đó; CMS ghi nhận phiên của **tất cả** hệ thống trong bảng `phien_he_thong` (chỉ định danh phiên, không lưu token của hệ thống khác) và chỉ quản trị viên (`phien_dang_nhap.xem`, `phien_dang_nhap.thu_hoi`) mới xem / thu hồi được. Đăng nhập, làm mới và đăng xuất ở một hệ thống **không ảnh hưởng** phiên ở hệ thống khác. **Thứ tự ghi**: hệ thống ghi bản sao của mình trước (cùng transaction với outbox), sau đó phát sự kiện lên Kafka topic `session-events` (khóa = `userId`, gửi ít nhất một lần): `session.started` (`startedAt`, `expiresAt`, `ip`, `userAgent`), `session.refreshed` (`expiresAt`), `session.ended` (`endedAt`, `reason`). Mọi thông điệp có `heThong`, `sessionId`, `userId`, `occurredAt`, `eventType`; CMS nhận, ghi nhận idempotent và đơn điệu (đã kết thúc thì không mở lại, thông điệp đến muộn / sai thứ tự không làm sai), bỏ qua thông điệp hỏng và thông điệp `heThong=CMS`. **Chiều ngược lại** (CMS → hệ thống) dùng `authz.session.revoked` trên topic `authz-events`: quản trị viên thu hồi một phiên (`scope=SESSION`, kèm `heThong` và `sessionId`) hoặc thay đổi của chính tài khoản (đổi / đặt lại mật khẩu, khóa, xóa) thu hồi **mọi** phiên (`scope=ALL`, không có `heThong`, kèm `revokedBeforeEpoch`); CMS ghi bản sao của mình trước rồi mới phát. Hệ thống nhận hủy phiên / refresh token của mình rồi phát `session.ended` xác nhận. Đăng nhập / đăng xuất thông thường **không** phát `authz.session.revoked`. **Xác minh mật khẩu**: backend của hệ thống khác không giữ mật khẩu; lúc đăng nhập gọi `POST /internal/auth/verify` (X-System-Key, dùng chung giới hạn đăng nhập sai) để CMS xác minh và trả `userId` cùng quyền / menu trong hệ thống đó (403 nếu chưa được cấp quyền); hệ thống tự tạo phiên, tự cấp access token / refresh token của mình (khóa ký riêng, không dùng JWKS của CMS) rồi phát `session.started`. Access token của CMS có claim `sid` trùng `sessionId` trong sổ phiên; **mỗi yêu cầu tới CMS tra sổ phiên theo `sid` (một truy vấn theo chỉ mục duy nhất, không đệm)** nên đăng xuất, thu hồi của quản trị viên, đổi / đặt lại mật khẩu, khóa, xóa tài khoản có hiệu lực ngay với access token còn hạn (401). Token không có `sid` hoặc phiên cấp trước V16 chưa có trong sổ được coi là hợp lệ đến khi hết hạn.
-   BR-11r: Phân quyền tập trung theo kiến trúc sự kiện (không để OTT / Affiliate gọi CMS mỗi yêu cầu). CMS phát lên Kafka topic `authz-events` (qua transactional outbox, ghi cùng transaction với thay đổi) hai loại sự kiện: **`authz.user.access.changed`** khi quyền của một người dùng đổi (tạo / sửa / khóa / mở khóa / xóa tài khoản, đổi vai trò, đổi quyền của vai trò, thêm / xóa quyền, đổi trạng thái nhà cung cấp): mỗi sự kiện sinh một thông điệp cho **từng hệ thống** gồm `heThong`, `heThongTrangThai`, `userId`, `active` và `quyen` (danh sách đầy đủ mã quyền / menu người dùng được cấp trong hệ thống đó), khóa Kafka là `userId` để giữ thứ tự; **`authz.definition.changed`** khi định nghĩa quyền / menu / trạng thái của một hệ thống đổi (kèm `daXoa`), khóa là mã hệ thống. Snapshot dựng lúc gửi từ CSDL (không qua cache) nên luôn mới nhất; sự kiện chưa gửi cùng loại và cùng khóa được gộp; gửi ít nhất một lần nên phía nhận khử trùng theo `eventId` (id tăng dần) và chỉ áp dụng thông điệp của hệ thống mình. **Mô hình cho OTT / Affiliate**: (1) xác minh token do chính hệ thống cấp (BR-11t; JWKS của CMS chỉ để xác minh token của CMS); (2) khi khởi động nạp định nghĩa bằng `GET /internal/authz/definitions` (kèm `phienBan`) và nạp lại khi nhận `authz.definition.changed` với `heThong` của mình; (3) giữ bản đồ `userId → tập mã quyền` thay thế toàn bộ theo mỗi `authz.user.access.changed` của hệ thống mình (`active=false` thì xóa), người dùng chưa có bản sao thì nạp một lần bằng `GET /internal/authz/users/{id}/access`; (4) quyết định cục bộ: mã quyền thì tra tập; (method, đường dẫn) thì khớp mẫu từ định nghĩa rồi tra tập; mặc định từ chối. Độ trễ thu hồi quyền cỡ giây (độ trễ outbox + Kafka); thao tác nhạy cảm có thể gọi `POST /internal/authz/check` để có kết quả tức thì. Nếu Kafka lỗi, sự kiện nằm lại trong outbox và gửi bù theo thứ tự.
-   BR-11g: QC kiểm tra nội dung rồi chủ động bấm "Duyệt"; hệ thống không tự đưa nội dung sang SAN\_SANG. Để QC biết media đã đầy đủ chưa, CMS cung cấp trạng thái media của từng phiên bản và danh sách tệp của nội dung trên R2.
-   BR-12: Mỗi lần encode lại sinh một phiên bản mới trong *phien\_ban\_noi\_dung*. Cột *phien\_ban\_mac\_dinh* trong *noi\_dung* trỏ tới phiên bản đang phục vụ.
-   BR-13: Chuyển trạng thái nội dung ghi vào *lich\_su\_quy\_trinh* kèm người thực hiện và thời điểm. Bảng này chỉ ghi thêm, không sửa.

***Media pipeline:***

-   BR-14: Một *tai\_nguyen\_media* có nhiều *cong\_viec\_media*. Mỗi công việc có nhiều lần thử (*lan\_thu\_cong\_viec*) với backoff 30 giây, 2 phút, 10 phút; quá năm lần chuyển *THAT\_BAI*.
-   BR-15: Callback từ nhà cung cấp ghi vào *phan\_hoi\_ncc* với *ma\_su\_kien* duy nhất. Sự kiện trùng *ma\_su\_kien* bị bỏ qua để bảo đảm idempotency.
-   BR-16: Chỉ khi bước xác minh đầu ra tìm đủ manifest và các playlist con thì *tai\_nguyen\_media* mới chuyển *SAN\_SANG*.

***Affiliate:***

-   BR-17: Click và lượt xem được ghi nhận theo *ma\_an\_danh* (cookie hoặc anonymous id). Quy gán dùng mô hình first-click với cửa sổ quy gán 30 ngày. Khi định danh được người dùng, Affiliate cập nhật *ma\_nguoi\_dung* nhưng không thay đổi attribution first-click đã chốt.
-   BR-18: Chuyển đổi (*chuyen\_doi*) được tạo từ sự kiện tài chính có ý nghĩa, ưu tiên PAYMENT\_SUCCESS nhận qua Kafka. SUBSCRIPTION\_CREATED chỉ dùng để bổ sung trạng thái hoặc fallback khi không có bản ghi thanh toán tương ứng; Affiliate không truy vấn *ott\_db*. Một khoản thanh toán chỉ tạo tối đa một conversion; xử lý idempotent theo *ma\_thanh\_toan* và *ma\_su\_kien\_nguon*.
-   BR-19: Hoa hồng tính theo *quy\_tac\_hoa\_hong* hiệu lực tại thời điểm chuyển đổi. *giao\_dich\_hoa\_hong* lưu snapshot quy tắc, cơ sở tính và số tiền cơ sở; thay đổi quy tắc sau đó không làm thay đổi lịch sử. Giao dịch hoa hồng ở trạng thái *CHO\_DUYET* trước khi đối soát và chi trả.

**Xóa mềm và kiểm toán:**

-   BR-20: Bảng dữ liệu chủ (tài khoản, nội dung, gói cước, nhà xuất bản) dùng xóa mềm qua *da\_xoa*. Bảng nhật ký và sự kiện (*lich\_su\_xem, phan\_hoi\_ncc, lich\_su\_quy\_trinh, event payment/affiliate*) chỉ ghi thêm, không xóa mềm.
-   BR-21: Hoàn tiền, chargeback hoặc điều chỉnh không sửa/xóa conversion và commission gốc; tạo bản ghi *dieu\_chinh\_hoa\_hong* tham chiếu giao dịch hoa hồng để truy vết.
-   BR-22: Conversion, commission và settlement lưu snapshot nghiệp vụ tại thời điểm phát sinh; thay đổi publisher, campaign, link hoặc commission rule về sau không làm thay đổi lịch sử.
-   BR-23: Một *ma\_thanh\_toan* chỉ được ghi nhận một conversion trong Affiliate. Duplicate Kafka event không tạo thêm doanh thu/hoa hồng.
-   BR-24: Chính sách hoa hồng cho thanh toán gia hạn được xác định trong *quy\_tac\_hoa\_hong*; hệ thống phân biệt thanh toán mới, gia hạn và loại giao dịch khác.
-   BR-25: Kỳ đối soát được khóa trước khi chi trả. Sau khi khóa, điều chỉnh phát sinh phải tạo bản ghi điều chỉnh mới, không sửa số liệu đã khóa.
-   BR-26: *ott\_db* là nguồn sự thật về doanh thu/thanh toán; *affiliate\_db* là nguồn sự thật về attribution, conversion, commission, settlement và payout.
-   BR-27: Thu hồi phiên phát hoặc phiên đăng nhập là thao tác duy nhất của quản trị viên trên hai bảng này (không xóa). Mỗi lần thu hồi bắt buộc nhập lý do và ghi một dòng vào *nhat\_ky\_thu\_hoi*. Thu hồi phiên phát chuyển *phien\_phat.trang\_thai* sang 4 và giảm bộ đếm phiên đồng thời trên Redis; thu hồi phiên đăng nhập chuyển *phien\_dang\_nhap.trang\_thai* sang 2, có thể áp dụng cho cả *family\_id*.

***Quản trị OTT (OTT Backend quản trị):***

-   BR-28: Phạm vi và quyền cơ sở dữ liệu. OTT Backend quản trị chỉ đọc *nguoi\_dung, ho\_so, ho\_so\_ngon\_ngu, thiet\_bi, dang\_ky, thanh\_toan, quyen\_xem, lich\_su\_xem*; ghi được các bảng cấu hình (*goi\_cuoc, goi\_cuoc\_da\_ngon\_ngu, gia\_goi\_cuoc, vung\_gia, tinh\_nang, tinh\_nang\_da\_ngon\_ngu, tinh\_nang\_goi*); với *phien\_phat* và *phien\_dang\_nhap* chỉ được đổi trạng thái khi thu hồi (BR-27) và ghi *nhat\_ky\_thu\_hoi*. Ràng buộc này được ép bằng GRANT theo bảng và cột của tài khoản kết nối riêng, không chỉ ở tầng API. Dịch vụ này sở hữu migration của cơ sở dữ liệu; OTT User API không tự migrate và đổi schema qua dịch vụ này theo kiểu mở rộng rồi thu hẹp (thêm trước, bỏ sau khi cả hai bên đã chuyển).
-   BR-29: Cổng đăng nhập của OTT Backend quản trị (áp dụng BR-11t, chi tiết ở `yeu_cau_ott_phien_dang_nhap.md`). (1) `POST /auth/login` nhận tên đăng nhập, mật khẩu; OTT không giữ mật khẩu mà gọi `POST {CMS}/internal/auth/verify` (X-System-Key, kèm `ip` của người dùng) **một lần**; 401 / 403 / 429 của CMS chuyển nguyên cho client, CMS lỗi hoặc khóa dịch vụ sai thì trả 503 và không tạo phiên. (2) Trong một transaction: tạo phiên (`sessionId` UUID v4) trong bản sao *adm\_phien\_quan\_tri*, lưu SHA-256 của refresh token vào *adm\_refresh\_token*, ghi outbox `session.started` (topic session-events). (3) Cấp access token ngắn hạn (mặc định 15 phút) ký ES256 bằng **khóa của OTT** (`iss` OTT, `sub` = userId, `sid` = sessionId, `jti`) và refresh token ngẫu nhiên 256 bit dùng một lần, xoay vòng; token của CMS không dùng cho API của OTT và JWKS của CMS không dùng để xác minh token của OTT. (4) Mỗi yêu cầu: xác minh chữ ký, `iss`, `exp` offline; kiểm `sid` còn đang hoạt động (đệm Redis hạn vài giây, xóa ngay khi phiên kết thúc; Redis lỗi thì tra thẳng CSDL, CSDL lỗi thì từ chối); kiểm quyền bằng bản sao quyền (`authz.user.access.changed`), mặc định từ chối. (5) Refresh dùng lại token đã xoay: kết thúc phiên, phát `session.ended` lý do `REFRESH_REUSE_DETECTED`, trả 401. (6) Đăng xuất chỉ kết thúc phiên hiện tại, phát `session.ended` lý do `LOGOUT`; gọi lặp vẫn thành công. (7) Nhận `authz.session.revoked` (`scope=SESSION` kèm `heThong=OTT` và `sessionId`; `scope=ALL` không có `heThong`, kèm `revokedBeforeEpoch`) và `authz.user.access.changed` có `active=false`: kết thúc phiên liên quan, thu hồi refresh token, phát `session.ended`; phiên bắt đầu sau `revokedBeforeEpoch` không bị ảnh hưởng; khử trùng theo `eventId`. (8) Kafka lỗi không chặn đăng nhập (sự kiện nằm lại outbox).
-   BR-30: Giá gói cước ở màn quản trị. (1) Máy chủ tính *gia\_goc* (= giá chuẩn × kỳ hạn) và *gia\_ban* (= gia\_goc × (1 − %/100)); làm tròn: VND về bội 1.000, tiền tệ khác 2 chữ số thập phân. API không nhận *gia\_goc*, *gia\_ban* từ client (gửi lên bị từ chối 400); form lấy giá trị xem trước từ `GET /plans/{id}/prices/preview`. Ngày kết thúc của mức giá là ngày cuối còn hiệu lực (tính cả ngày đó), lưu là 00:00 giờ Việt Nam của ngày kế tiếp. (2) Mức giá đã được *dang\_ky* tham chiếu: khóa *ma\_vung\_gia, so\_thang, gia\_moi\_thang, gia\_goc, phan\_tram\_khuyen\_mai, gia\_ban*; chỉ đổi được *ngay\_ket\_thuc* và *trang\_thai*. Muốn đổi giá thì kết thúc mức cũ và tạo mức mới. (3) Không đổi *vung\_gia.don\_vi\_tien* khi vùng đã có mức giá; muốn đổi thì tạo vùng mới. (4) Không xóa hoặc ngừng bán gói còn đăng ký đang hoạt động; *ma\_tat\_goi* không đổi sau khi tạo; ba gói mặc định FREE, VIP, VIP\_PRO không xóa được.
-   BR-31: Dữ liệu cá nhân. Màn quản trị che email và số điện thoại mặc định; quyền `ott.user.xem_pii` mới xem đủ, và mỗi lần xem đủ được ghi vào *adm\_nhat\_ky\_kiem\_toan*. Không đưa dữ liệu cá nhân vào log, URL, chỉ số vận hành. Không bao giờ trả *play\_token* và *token\_id* ra API quản trị.
-   BR-32: Múi giờ. Mọi thời gian của OTT Backend quản trị theo Asia/Ho\_Chi\_Minh (GMT+7): JVM, phiên cơ sở dữ liệu, JSON, thông điệp Kafka (ISO-8601 có offset +07:00). Cột vẫn là TIMESTAMPTZ. Biên "ngày" và "tháng" tính từ 00:00 đến 24:00 giờ Việt Nam; thời hạn gói cộng *so\_thang* tháng theo giờ Việt Nam. CMS dùng cùng múi giờ nên không có bước quy đổi.
-   BR-33: Hiệu lực thu hồi phiên. Ngoài BR-27: (1) chỉ thu hồi khi *trang\_thai* hiện tại là 1; gọi lặp trả lại kết quả cũ, không ghi thêm nhật ký, không giảm bộ đếm lần hai (UNIQUE trên *nhat\_ky\_thu\_hoi*); (2) sau commit ghi khóa từ chối trên Redis (`revoked:play:{ma\_phien}` với TTL đến khi token hết hạn nhưng không dưới `REVOKE_PLAY_DENY_MIN_TTL`; `revoked:login:session:{id phiên}` và, khi thu hồi cả family, `revoked:login:family:{family\_id}` với TTL `REVOKE_LOGIN_DENY_TTL` nên không nhỏ hơn hạn access token của người dùng cuối; dịch vụ quản trị không đọc được `token\_id`) và phát `playback.session.revoked` / `auth.session.revoked` qua outbox lên topic user-events để OTT User API và Cloudflare Worker chặn token đã thu hồi; (3) giảm bộ đếm `phien:{ma\_nguoi\_dung}` bằng Lua, không xuống dưới 0.
-   BR-34: Bộ lọc danh sách ở màn quản trị. Mọi danh sách theo thời gian mặc định hôm nay (giờ Việt Nam), khoảng tối đa 30 ngày, từ ngày không sau đến ngày, đến ngày không ở tương lai; máy chủ kiểm bắt buộc và từ chối bằng 400 (DATE\_RANGE\_INVALID, DATE\_RANGE\_TOO\_LARGE). Giá trị bộ lọc (trạng thái, loại thiết bị, tiền tệ...) theo danh sách trắng; từ khóa tìm kiếm chỉ qua tham số hóa.

# 3\. Media Pipeline (Sequence flows)

## 3.1 Flow tổng quát

-   **Upload nguồn**: Admin/Content Provider upload video gốc (ví dụ movie001.mkv) lên R2 bucket **original-private** thông qua presigned URL do CMS API cấp. (CMS gọi R2 API để lấy URL PUT; user upload trực tiếp).
-   **Ingest**: R2 phát sinh event ObjectCreated, đẩy vào queue. Media API consumer bắt event, gọi HEAD để xác nhận file. Cập nhật trạng thái tai\_nguyen\_media.trang\_thai=READY. Thông báo CMS (webhook hoặc callback API) file đã sẵn sàng; CMS chỉ đánh dấu phiên bản có video gốc sẵn sàng (phien\_ban\_noi\_dung.trang\_thai\_media = 1), **không tự đổi trạng thái nội dung**. Nhà cung cấp hoặc biên tập viên chủ động bấm "Gửi QC" (BAN\_NHAP → CHO\_QC) khi đã có video gốc sẵn sàng. Tệp chỉ được tải lên R2 bằng presigned URL do CMS cấp; backend không nhận nội dung tệp.
-   **QC Approve**: QA xem qua CMS. Từ chối transcode thì cập nhật noi\_dung.trang\_thai=QC\_REJECTED. Nếu đồng ý, Chọn nút “Transcode+DRM”. CMS gửi POST /media/jobs với contentId, source key, profile, v.v.
-   **Transcode**: Media API tạo cong\_viec\_media, và cong\_viec\_ncc lưu trang\_thai = CREATED. Gọi adapter Bitmovin để cấu hình input từ R2 (S3 bucket), output đến R2 **streaming** bucket, cài đặt profile (1080p/720p...), và DRM.
    -   **Không dùng DRM**: Bitmovin chỉ encode & package HLS/DASH mà không mã hóa; EZDRM không dùng.
    -   **Nếu dùng DRM**: Bitmovin encoding cấu hình với EZDRM KeyZ API (Widevine/PlayReady, Apple FairPlay qua Common Encryption). EZDRM sẽ cung cấp **CENC Key IDs** cho Bitmovin.
-   **Callback bitmovin**: Khi Bitmovin thực hiện ecode xong, nó gửi webhook đến Media API. Media API lưu lại dữ liệu vào phan\_hoi\_ncc. Dựa vào dữ liệu đó, hệ thống phân tích kết quả và cập nhật trạng thái tương ứng vào bảng cong\_viec\_ncc
-   **Verify output**: Media API kiểm tra R2 streaming bucket có đủ manifest/file (HEAD hoặc lấy danh sách). Kiểm tra master.m3u8, manifest.mpd, các sub-playlists (1080p,720p…). Nếu thiếu, báo lỗi & retry.
-   **Media ready**: Nếu ok, cập nhật tai\_nguyen\_media.trang\_thai=READY. Gửi callback tới CMS API: thông tin các manifest và chất lượng, DRM flag. CMS lưu manifest, chất lượng, cờ DRM vào phiên bản và đặt làm phiên bản phục vụ nếu nội dung chưa có; **nội dung không tự chuyển sang SAN\_SANG**. QC kiểm tra (trạng thái media và danh sách tệp trên Cloudflare R2 xem qua API media-status / storage-files của CMS) rồi bấm "Duyệt" (CHO\_QC → SAN\_SANG); nút chỉ bật khi phiên bản phục vụ có media sẵn sàng.
-   **Publish**: Admin chọn publish trong CMS (hoặc tự động publish). noi\_dung.trang\_thai=PUBLISHED. Ở OTT API sẽ chỉ liệt kê các nội dung có trạng thái này.
-   **Playback**: User bấm play trên web/app. OTT API xác thực => check subscription/region/device => tạo session. OTT API trả **URL manifest (CDN)** và DRM license endpoints (từ EZDRM RightsZ). Ví dụ:

manifest: {
hls: "https://media.example.com/c10001/v1/hls/master.m3u8",
dash: "https://media.example.com/c10001/v1/dash/manifest.mpd"
},
drm: {
widevine: { licenseUrl: "https://license.example.com/eZVm64" },
playready: { licenseUrl: "https://license.example.com/ePRa78" },
f airplay: { licenseUrl: "https://license.example.com/eFpL12" }
}

Player fetch manifest from CDN, then license from EZDRM. Play token (HMAC JWT) xác thực.

![Word image 1](assets/image-001.png)

## 3.2 Chi tiết adapter/provider (anti lock-in)

-   **Adapter Pattern**: Tạo interface TranscoderProvider và DrmProvider. Triển khai BitmovinProvider, EzdrmProvider. Config mỗi provider bằng YAML/DB. Tương tự StorageProvider cho R2 (S3-compatible).
-   **Provider contract**:
    -   *TranscoderProvider.encode(MediaJob job)*: nạp nguồn (bucket/khoa\_doi\_tuong) từ tep\_nguon, output bucket, profile list. Trả external\_job\_id.
    -   *DrmProvider.configure(MediaJob job)*: liên kết với Bitmovin encoder thông qua KeyZ (đăng ký asset, trả ID).
-   **Job & Callback**: provider\_job lưu provider, external\_id, type, status. Khi Bitmovin webhook đến media-api/webhooks/bitmovin, tìm provider\_job theo external\_id, cập nhật trạng thái. Tương tự nếu EZDRM cần callback (thường EZDRM KeyZ dùng synchronous trong encode).
-   **Idempotency**:
    -   Dùng header Idempotency-Key cho các call tạo job/upload. Nếu CMS gửi 2 lần, trả cùng jobId.
    -   Lưu phan\_hoi\_ncc(external\_event\_id) để không xử lý sự kiện provider trùng lặp.
-   **Retry**: Có backoff policy (ví dụ 30s, 2min, 10min). Mỗi lần retry cập nhật lan\_thu\_cong\_viec. Nếu quá ngưỡng (VD 5 lần), đánh trạng thái FAILURE, gửi báo lỗi.
-   **Verify steps**: Dùng HEAD/GET object từ R2: kiểm tra manifest tồn tại, kích thước hợp lý (checksum). Kiểm tra playlist syntax nếu cần (regex). Nếu DRM bật, verify license request (EZDRM logs).

# 4\. Thiết kế Cơ sở dữ liệu

## 4.1 ott\_db

-   ott\_db là nguồn sự thật về tài khoản người dùng, hồ sơ, gói cước thương mại, đăng ký, thanh toán, quyền xem, phiên phát và lịch sử xem.
-   Metadata phim, ảnh, thể loại, phụ đề và chính sách quyền xem theo nội dung thuộc cms\_db; ott\_db chỉ lưu các khóa logic khi cần.
-   Các tham chiếu từ ott\_db sang cms\_db không đặt FK vật lý.
-   Gói cước (goi\_cuoc) là định nghĩa thương mại; giá bán được tách thành gia\_goi\_cuoc để hỗ trợ nhiều kỳ hạn, vùng và khuyến mại.
-   vung\_gia xác định thị trường và đơn vị tiền tệ. Mặc định: Việt Nam (VN) dùng VND; thị trường còn lại (GLOBAL) dùng USD.
-   Nội dung hiển thị đa ngôn ngữ của gói/tính năng được tách khỏi dữ liệu nghiệp vụ vào các bảng \*\_da\_ngon\_ngu.
-   Ngôn ngữ được tham chiếu logic tới danh mục cms\_db.ngon\_ngu; không tạo FK xuyên database.
-   Khi phát sinh đăng ký/thanh toán, các thông tin giá và khuyến mại được snapshot để lịch sử không thay đổi khi bảng giá tương lai được

![Word image 2](assets/image-002.png)

[ott\_db.md](https://drive.google.com/file/d/1fRkevpf7B54giBcqXHeA6VRI5V_4S0V_/view?usp=drive_link)

### Bảng nguoi\_dung (tài khoản người dùng cuối)

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh người dùng |
| email | VARCHAR(255) | NOT NULL | Email đăng nhập, UNIQUE trên bản ghi da\_xoa=false |
| mat\_khau\_hash | VARCHAR(255) | NOT NULL | Mật khẩu đã băm, thuật toán bcrypt hoặc Argon2 |
| so\_dien\_thoai | VARCHAR(20) |  | Số điện thoại |
| ho\_ten | VARCHAR(150) |  | Họ tên hiển thị |
| ma\_vung | VARCHAR(20) | NOT NULL | Vùng/thị trường của người dùng.<br><br>VN, GLOBAL; tham chiếu logic vung\_gia.ma\_vung |
| ma\_ngon\_ngu | BIGINT |  | Ngôn ngữ mặc định<br><br>Tham chiếu logic cms\_db.ngon\_ngu.id; ví dụ vi, en |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 0 | Trạng thái tài khoản:<br><br>0: chờ kích hoạt,<br>1: hoạt động,<br>2: khóa |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Cập nhật gần nhất |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Cờ xóa mềm |

### Bảng ho\_so (hồ sơ người dùng trong một tài khoản) : tối đa 5 hồ sơ

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh hồ sơ |
| ma\_nguoi\_dung | BIGINT | FK → nguoi\_dung.id,<br><br>NOT NULL | Tài khoản chứa hồ sơ |
| ten\_hien\_thi | VARCHAR(100) | NOT NULL | Tên hồ sơ |
| avatar\_key | VARCHAR(255) |  | Khóa ảnh đại diện trên R2 |
| loai\_ho\_so | SMALLINT | NOT NULL, DEFAULT 0 | Loại hồ sơ<br>0: người lớn,<br><br>1: trẻ em |
| ma\_ngon\_ngu | BIGINT | DEFAULT 'vi' | Ngôn ngữ giao diện<br><br>Tham chiếu logic cms\_db.ngon\_ngu.id; mặc định vi |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Cập nhật gần nhất |
| nguoi\_tao | BIGINT |  | Tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL,<br><br>DEFAULT false | Cờ xóa mềm |

### Bảng ho\_so\_ngon\_ngu

Danh sách ngôn ngữ người dùng/hồ sơ ưu tiên, dùng khi hệ thống cần chọn ngôn ngữ phụ đề/audio theo thứ tự ưu tiên. Có thể chưa bật trong MVP nhưng cấu trúc được chuẩn bị sẵn.

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh |
| ma\_ho\_so | BIGINT | FK → ho\_so.id,<br><br>NOT NULL | Hồ sơ |
| ma\_ngon\_ngu | BIGINT | NOT NULL | Ngôn ngữ ưu tiên<br><br>Tham chiếu logic cms\_db.ngon\_ngu.id |
| thu\_tu\_uu\_tien | SMALLINT | NOT NULL | Thứ tự ưu tiên: 1 là cao nhất |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Cờ xóa mềm |

Unique đề xuất: (ma\_ho\_so, ma\_ngon\_ngu) và (ma\_ho\_so, thu\_tu\_uu\_tien) trên bản ghi chưa xóa.

###

### **Bảng thiet\_bi** (thiết bị đã đăng nhập):

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh bản ghi thiết bị |
| ma\_nguoi\_dung | BIGINT | FK → nguoi\_dung.id, NOT NULL | Chủ thiết bị |
| ma\_ho\_so | BIGINT | FK → ho\_so.id, NOT NULL | Hồ sơ đang hoặc lần cuối sử dụng thiết bị |
| ma\_thiet\_bi | VARCHAR(128) | NOT NULL | Vân tay thiết bị<br><br>UNIQUE cùng ma\_nguoi\_dung |
| loai\_thiet\_bi | VARCHAR(50) |  | Loại thiết bị: web, android, ios, smarttv |
| ten\_thiet\_bi | VARCHAR(150) |  | Tên do người dùng đặt |
| lan\_cuoi\_dang\_nhap | TIMESTAMPTZ |  | Lần đăng nhập gần nhất |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | Trạng thái thiết bị<br><br>1: hoạt động |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Cập nhật gần nhất |
| nguoi\_tao | BIGINT |  | Tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL,<br><br>DEFAULT false | Cờ xóa mềm |

###

### Bảng goi\_cuoc (gói cước):

Định nghĩa gói thương mại, không lưu một mức giá cố định và không lưu nội dung tên/mô tả theo ngôn ngữ.

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh gói |
| ma\_tat\_goi | VARCHAR(50) | NOT NULL, UNIQUE | Mã nghiệp vụ của gói<br><br>FREE, VIP, VIP\_PRO... |
| ten\_mac\_dinh | VARCHAR(150) | NOT NULL | Tên mặc định của gói, dùng khi người dùng chưa có cấu hình trong goi\_cuoc\_da\_ngon\_ngu (BR-05) |
| mo\_ta\_mac\_dinh | TEXT |  | Mô tả mặc định của gói, dùng cùng điều kiện với ten\_mac\_dinh |
| so\_thiet\_bi\_dong\_thoi | SMALLINT | NOT NULL, DEFAULT 1 | Số thiết bị phát đồng thời |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | Trạng thái gói<br><br>0: ngừng bán,<br><br>1: đang bán |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Cập nhật gần nhất |
| nguoi\_tao | BIGINT |  | Tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Cờ xóa mềm |

### Bảng goi\_cuoc\_da\_ngon\_ngu (gói cước đa ngôn ngữ):

Tên, mô tả và nội dung hiển thị của gói theo từng ngôn ngữ

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh gói vùng |
| ma\_goi\_cuoc | BIGINT | FK → goi\_cuoc.id<br>NOT NULL | Mã gói cước |
| ma\_ngon\_ngu | BIGINT | NOT NULL | Ngôn ngữ<br><br>Logical ref → cms\_db.ngon\_ngu.id |
| ten\_goi | VARCHAR(150) | NOT NULL | Tên gói |
| mo\_ta | TEXT |  | Mô tả phim |
| ghi\_chu | TEXT |  | Nội dung bổ sung hiển thị |
| trang\_thai | SMALLINT |  | Trạng thái<br><br>0: vô hiệu<br>1: kích hoạt |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Cập nhật gần nhất |
| nguoi\_tao | BIGINT |  | Tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Cờ xóa mềm |

Unique: (ma\_goi\_cuoc, ma\_ngon\_ngu) trên bản ghi chưa xóa

### Bảng vung\_gia

Xác định thị trường và đồng tiền dùng để định giá gói.

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh vùng giá |
| ma\_vung | VARCHAR(20) | NOT NULL, UNIQUE | Mã vùng giá<br><br>VN, GLOBAL |
| ten | VARCHAR(150) | NOT NULL | Tên vùng<br><br>Việt Nam, Quốc tế |
| ma\_quoc\_gia | VARCHAR(2) |  | Quốc gia đại diện<br><br>VN cho vùng Việt Nam; NULL hoặc \* cho GLOBAL tùy quy ước |
| don\_vi\_tien | VARCHAR(3) | NOT NULL | Đồng tiền<br><br>VN → VND; GLOBAL → USD |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | Trạng thái<br><br>0: ngừng dùng, 1: hoạt động |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Cập nhật |
| nguoi\_tao | BIGINT |  | Tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Cờ xóa mềm |

Quy tắc mặc định:

-   VN → VND.
-   GLOBAL → USD.
-   Có thể mở rộng thêm vùng/currency trong tương lai mà không đổi cấu trúc gia\_goi\_cuoc.
-   ma\_vung không đổi sau khi tạo; don\_vi\_tien không đổi khi vùng đã được gia\_goi\_cuoc tham chiếu (BR-30). Vùng không có thao tác xóa, chỉ ngừng dùng (trang\_thai = 0).

### Bảng gia\_goi\_cuoc

Bảng giá theo **gói + vùng + kỳ hạn**. Đây là bảng trung tâm cho mô hình 1/3/6/12 tháng và khuyến mại theo phần trăm

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh bảng giá |
| ma\_goi\_cuoc | BIGINT | FK → goi\_cuoc.id,<br><br>NOT NULL | Gói áp dụng |
| ma\_vung\_gia | BIGINT | FK → vung\_gia.id,<br><br>NOT NULL | Vùng áp dụng<br><br>Quyết định currency |
| so\_thang | SMALLINT | NOT NULL, CHECK > 0 | Kỳ hạn<br><br>1, 3, 6, 12 |
| gia\_moi\_thang | NUMERIC(12,2) | NOT NULL, CHECK >= 0 | Giá chuẩn/tháng<br><br>Ví dụ 90.000 VND |
| gia\_goc | NUMERIC(12,2) | NOT NULL, CHECK >= 0 | Giá trước khuyến mại<br><br>Thường = gia\_moi\_thang × so\_thang |
| phan\_tram\_khuyen\_mai | NUMERIC(5,2) | NOT NULL, DEFAULT 0, CHECK 0–100 | Tỷ lệ giảm<br><br>Ví dụ 10.00 |
| gia\_ban | NUMERIC(12,2) | NOT NULL, CHECK >= 0 | Giá thực tế<br><br>gia\_goc × (1 - %/100); làm tròn theo currency/payment policy |
| ngay\_bat\_dau | TIMESTAMPTZ | NOT NULL | Bắt đầu hiệu lực |
| ngay\_ket\_thuc | TIMESTAMPTZ |  | Kết thúc hiệu lực<br><br>NULL = không giới hạn |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | Trạng thái bảng giá<br><br>0: ngừng bán, 1: đang bán |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Cập nhật |
| nguoi\_tao | BIGINT |  | Tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Cờ xóa mềm |

Ràng buộc: không cho phép hai bảng giá cùng (ma\_goi\_cuoc, ma\_vung\_gia, so\_thang) cùng hiệu lực tại một thời điểm. Service kiểm trước; chốt cuối bằng exclusion constraint (btree\_gist) trên (ma\_goi\_cuoc, ma\_vung\_gia, so\_thang, tstzrange(ngay\_bat\_dau, ngay\_ket\_thuc)) với trang\_thai = 1 và da\_xoa = false. gia\_goc, gia\_ban do máy chủ tính, bỏ qua giá trị client gửi (BR-30).

Ví dụ: VIP tại VN: 1 tháng = 90.000; 3 tháng = 270.000, khuyến mại 10% → giá bán 243.000.

gia\_goi\_cuoc không lưu don\_vi\_tien; tiền tệ lấy từ vung\_gia.don\_vi\_tien để tránh dữ liệu mâu thuẫn.

### Bảng tinh\_nang (danh mục tính năng):

Danh mục tính năng kỹ thuật/nghiệp vụ của gói. Tách mã kỹ thuật khỏi nội dung hiển thị.

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh tính năng |
| ma\_tinh\_nang | VARCHAR(50) | NOT NULL, UNIQUE | Mã tắt tính năng, ví dụ: MAX\_QUALITY, DOWNLOAD, AD\_FREE |
| gia\_tri\_mac\_dinh | VARCHAR(200) |  | Tên tính năng, ví dụ: tải offline, không quảng cáo |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | Trạng thái<br>0: ngừng dùng, 1: hoạt động |

### Bảng tinh\_nang\_da\_ngon\_ngu

Nội dung hiển thị của tính năng theo ngôn ngữ.

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh |
| ma\_tinh\_nang | BIGINT | FK → tinh\_nang.id,<br><br>NOT NULL | Tính năng |
| ma\_ngon\_ngu | BIGINT | NOT NULL | Ngôn ngữ<br><br>Logical ref → cms\_db.ngon\_ngu.id |
| ten | VARCHAR(150) | NOT NULL | Tên hiển thị |
| mo\_ta | TEXT |  | Mô tả |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Cập nhật |
| nguoi\_tao | BIGINT |  | Tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Cờ xóa mềm |

Unique: (ma\_tinh\_nang, ma\_ngon\_ngu) trên bản ghi chưa xóa.

### Bảng tinh\_nang\_goi (tính năng theo gói):

Ánh xạ tính năng vào gói và cho phép mỗi gói có giá trị tham số riêng.

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh |
| ma\_goi\_cuoc | BIGINT | FK → goi\_cuoc.id, NOT NULL | Gói |
| ma\_tinh\_nang | BIGINT | FK → tinh\_nang.id,<br><br>NOT NULL | Tính năng |
| gia\_tri | VARCHAR(255) |  | Giá trị áp dụng cho gói<br><br>Ví dụ 1080p, true, 5 |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| nguoi\_tao | BIGINT |  | Tài khoản tạo |

Unique: (ma\_goi\_cuoc, ma\_tinh\_nang). Bảng ánh xạ, không có da\_xoa: xóa cứng; vết thay đổi nằm ở adm\_nhat\_ky\_kiem\_toan.

###

### Bảng dang\_ky (đăng ký gói của người dùng):

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh đăng ký |
| ma\_nguoi\_dung | BIGINT | FK → nguoi\_dung.id,<br><br>NOT NULL | Người đăng ký |
| ma\_goi\_cuoc | BIGINT | FK → goi\_cuoc.id, NOT NULL | Gói đã chọn |
| ma\_gia\_goi\_cuoc | BIGINT | FK → gia\_goi\_cuoc.id, NOT NULL | Mã gói cước đăng ký chi tiết |
| so\_thang | SMALLINT | NOT NULL | Số tháng đã mua<br><br>Snapshot |
| gia\_goc | NUMERIC(12,2) | NOT NULL | Giá trước giảm<br><br>Snapshot |
| phan\_tram\_khuyen\_mai | NUMERIC(5,2) | NOT NULL, DEFAULT 0 | % khuyến mại<br><br>Snapshot |
| gia\_ban | NUMERIC(12,2) | NOT NULL | Giá phải trả<br><br>Snapshot |
| don\_vi\_tien | VARCHAR(3) | NOT NULL | Currency<br><br>Snapshot từ vung\_gia |
| ngay\_bat\_dau | TIMESTAMPTZ |  | Thời điểm bắt đầu hiệu lực. NULL khi đang chờ thanh toán |
| ngay\_ket\_thuc | TIMESTAMPTZ |  | Thời điểm hết hạn. NULL khi đăng ký đang chờ thanh toán, được chốt khi kích hoạt |
| ngay\_huy | TIMESTAMPTZ |  | Thời điểm hủy gói. NULL nếu chưa hủy |
| tu\_dong\_gia\_han | BOOLEAN | NOT NULL, DEFAULT true | Có tự động gia hạn không |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 0 | Trạng thái đăng ký<br><br>0: chờ thanh toán,<br><br>1: đang hoạt động,<br><br>2: hết hạn,<br><br>3: đã hủy |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Cập nhật |
| nguoi\_tao | BIGINT |  | Tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Cờ xóa mềm |

###

Ràng buộc: UNIQUE một phần ux\_dang\_ky\_hoat\_dong trên (ma\_nguoi\_dung) WHERE trang\_thai = 1 AND da\_xoa = false (mỗi người dùng chỉ có một đăng ký đang hoạt động, xem mục 4.5). Index: (ma\_nguoi\_dung, ngay\_tao DESC), (ma\_goi\_cuoc, ngay\_tao DESC).

### Bảng thanh\_toan (giao dịch thanh toán):

Giao dịch thanh toán. so\_tien và don\_vi\_tien là snapshot thực tế gửi/nhận qua cổng thanh toán

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh giao dịch |
| ma\_nguoi\_dung | BIGINT | FK → nguoi\_dung.id, NOT NULL | Người thanh toán |
| ma\_dang\_ky | BIGINT | FK → dang\_ky.id, NOT NULL | Đăng ký liên quan |
| so\_tien | NUMERIC(12,2) | NOT NULL | Số tiền |
| don\_vi\_tien | VARCHAR(3) | NOT NULL | Currency giao dịch<br><br>VND hoặc USD |
| phuong\_thuc | VARCHAR(50) | NOT NULL | Hình thức thanh toán: the, vidien tu, chuyen khoan |
| ma\_giao\_dich\_cong | VARCHAR(255) | NOT NULL, UNIQUE | Mã giao dịch từ cổng thanh toán.<br><br>Khóa idempotency của webhook |
| ma\_thanh\_toan\_goc | BIGINT | FK → thanh\_toan.id | Giao dịch gốc khi hoàn tiền.<br><br>NULL nếu là giao dịch thường |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 0 | Trạng thái<br><br>0: chờ xử lý,<br><br>1: thành công, 2: thất bại,<br><br>3: hoàn tiền |
| tg\_bat\_dau | TIMESTAMPTZ |  | Thời gian bắt đầu giao dịch |
| tg\_ket\_thuc | TIMESTAMPTZ |  | Thời gian kết thúc giao dịch |
| tg\_gui\_cong | TIMESTAMPTZ |  | Thời gian gửi sang cổng thanh toán |
| tg\_cong\_xac\_nhan | TIMESTAMPTZ |  | Thời gian cổng gửi xác nhận về hệ thống |
| ngay\_tao | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Thời điểm tạo bản ghi. Khóa lọc theo ngày ở màn quản trị (dùng COALESCE(tg\_bat\_dau, ngay\_tao)) vì tg\_bat\_dau có thể NULL |

### Bảng quyen\_xem:

Quyền xem nội dung cụ thể, có thể phát sinh từ gói hoặc mua lẻ. Chính sách cấp độ/quyền xem của nội dung vẫn thuộc cms\_db

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh quyền xem |
| ma\_nguoi\_dung | BIGINT | FK → nguoi\_dung.id, NOT NULL | Người được cấp quyền |
| ma\_noi\_dung | BIGINT | NOT NULL | Logical ref → cms\_db.noi\_dung.id |
| ma\_tai\_nguyen\_media | BIGINT |  | Logical ref → media\_db.tai\_nguyen\_media.id |
| den\_han | TIMESTAMPTZ |  | NULL nếu vô thời hạn |
| nguon | SMALLINT | NOT NULL, DEFAULT 0 | 0: theo gói, 1: mua lẻ, 2: tặng/quyền khác |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm cấp |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Cập nhật |
| nguoi\_tao | BIGINT |  | Tài khoản/dịch vụ cấp |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Cờ xóa mềm |

### Bảng phien\_phat (phiên phát video):

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh phiên |
| ma\_phien | UUID | NOT NULL, UNIQUE | Mã phiên công khai dùng trong play token |
| ma\_nguoi\_dung | BIGINT | FK → nguoi\_dung.id,<br><br>NOT NULL | Người phát |
| ma\_noi\_dung | BIGINT | NOT NULL | Nội dung đang phát.<br><br>Khóa logic sang cms\_db.noi\_dung |
| ma\_thiet\_bi | BIGINT | FK → thiet\_bi.id | Thiết bị phát |
| play\_token | VARCHAR(512) | NOT NULL | Token phát HMAC/JWT.<br><br>Cloudflare kiểm trước khi phục vụ manifest |
| bat\_dau\_luc | TIMESTAMPTZ | NOT NULL | Thời điểm bắt đầu |
| het\_han\_luc | TIMESTAMPTZ | NOT NULL | Thời điểm token hết hạn |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | Trạng thái phiên<br><br>1: đang phát,<br><br>2: kết thúc,<br><br>3: hết hạn,<br><br>4: đã thu hồi (quản trị viên thu hồi, ghi tại nhat\_ky\_thu\_hoi) |

###

### Bảng lich\_su\_xem

Bảng append-only, ghi lịch sử xem và vị trí xem tiếp

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh bản ghi |
| ma\_nguoi\_dung | BIGINT | FK → nguoi\_dung.id, NOT NULL | Người xem |
| ma\_ho\_so | BIGINT | FK → ho\_so.id | Hồ sơ xem |
| ma\_noi\_dung | BIGINT | NOT NULL | Nội dung<br><br>Logical ref → cms\_db.noi\_dung.id |
| vi\_tri\_giay | INT | NOT NULL, DEFAULT 0 | Vị trí xem theo giây<br><br>Dùng cho xem tiếp |
| thoi\_luong\_giay | INT |  | Tổng thời lượng |
| phat\_luc | TIMESTAMPTZ | NOT NULL | Thời điểm phát |

### Bảng phien\_dang\_nhap (quản lý truy cập):

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh phiên |
| ma\_nguoi\_dung | BIGINT | FK → nguoi\_dung.id,<br><br>NOT NULL | Mã người dùng |
| ma\_ho\_so | BIGINT | FK → ho\_so.id,<br><br>NOT NULL | Mã hồ sơ người dùng |
| ma\_thiet\_bi | BIGINT | FK → thiet\_bi.id,<br><br>NOT NULL | Mã thiết bị |
| ip\_dang\_nhap | VARCHAR(50) |  | IP truy cập |
| user\_agent | VARCHAR(500) |  |  |
| thoi\_gian\_dang\_nhap | TIMESTAMPTZ |  | Thời gian đăng nhập |
| thoi\_gian\_het\_han | TIMESTAMPTZ |  | Thời gian hết hạn |
| thoi\_gian\_dang\_xuat | TIMESTAMPTZ |  | Thời gian đăng xuất |
| trang\_thai | SMALLINT |  | Trạng thái truy cập<br><br>1: đang sử dụng<br><br>2: đã thu hồi |
| token\_id | VARCHAR(500) | UNIQUE | Hash của refresh token, không lưu token thô |
| family\_id | VARCHAR(500) | NOT NULL; index một phần WHERE trang\_thai = 1; **không UNIQUE** | Mã gán theo thiết bị, dùng để xoay token\_id. Nhiều phiên có thể cùng family nên mới thu hồi được cả family (BR-27) |

### Bảng nhat\_ky\_thu\_hoi (nhật ký thu hồi phiên)

Bảng chỉ ghi thêm, không sửa, không xóa mềm (BR-20). Mỗi lần quản trị viên thu hồi phiên phát hoặc phiên đăng nhập ghi một dòng để truy vết.

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh |
| loai\_doi\_tuong | SMALLINT | NOT NULL | 1: phien\_phat, 2: phien\_dang\_nhap |
| ma\_doi\_tuong | BIGINT | NOT NULL | id của phien\_phat hoặc phien\_dang\_nhap bị thu hồi (khóa logic theo loai\_doi\_tuong) |
| ma\_nguoi\_dung | BIGINT | FK → nguoi\_dung.id, NOT NULL | Người dùng bị thu hồi phiên |
| pham\_vi | SMALLINT | NOT NULL, DEFAULT 0 | 0: một phiên, 1: toàn bộ phiên cùng family\_id (chỉ áp dụng cho phiên đăng nhập) |
| nguoi\_thuc\_hien | BIGINT | NOT NULL | Logical ref → cms\_db.tai\_khoan\_he\_thong.id |
| ly\_do | VARCHAR(500) | NOT NULL | Lý do thu hồi, bắt buộc nhập |
| thoi\_diem | TIMESTAMPTZ | NOT NULL | Thời điểm thu hồi |

Ràng buộc: UNIQUE (loai\_doi\_tuong, ma\_doi\_tuong): một phiên chỉ bị thu hồi một lần, là chốt cuối của idempotency (BR-33). Index (ma\_nguoi\_dung, thoi\_diem DESC).

### Bảng bổ sung cho OTT Backend quản trị

Các bảng kỹ thuật do dịch vụ OTT Backend quản trị sở hữu, đặt tiền tố `adm\_` để không va chạm với bảng cùng chức năng của OTT User API trong cùng cơ sở dữ liệu `film_ai_ott`.

| **Bảng** | **Cột chính** | **Mục đích** |
| --- | --- | --- |
| adm\_idempotency\_key | ma\_tai\_khoan, khoa, method, path, request\_hash, trang\_thai (0 đang xử lý, 1 hoàn tất), http\_status, response\_body JSONB, het\_han\_luc; UNIQUE (ma\_tai\_khoan, khoa) | Idempotency cho API tạo mới và thao tác có tác dụng phụ |
| adm\_su\_kien\_cho\_gui | topic, khoa\_thong\_diep, loai\_su\_kien, noi\_dung JSONB, trang\_thai (0 chờ, 1 đã gửi, 2 bỏ cuộc), so\_lan\_thu, loi | Transactional outbox: thu hồi phiên, thay đổi cấu hình gói |
| adm\_su\_kien\_da\_nhan | nguon, ma\_su\_kien, noi\_dung\_goc JSONB; UNIQUE (nguon, ma\_su\_kien) | Inbox khử trùng sự kiện authz-events, cms-events |
| adm\_nhat\_ky\_kiem\_toan | ma\_tai\_khoan, hanh\_dong, doi\_tuong, ma\_doi\_tuong, truoc, sau JSONB, trace\_id, thoi\_diem | Chỉ ghi thêm: ai sửa gì, và mỗi lần xem thông tin cá nhân đầy đủ (BR-31) |
| adm\_phien\_quan\_tri | session\_id VARCHAR(64) PK (= sessionId trong session-events), ma\_tai\_khoan, ten\_dang\_nhap, trang\_thai (1 hoạt động, 2 đã kết thúc), ly\_do\_ket\_thuc, bat\_dau\_luc, het_han\_luc, ket\_thuc\_luc, ip, user\_agent | Bản sao phiên đăng nhập quản trị của OTT (BR-29); chỉ phiên thuộc OTT |
| adm\_refresh\_token | id, session\_id FK, refresh\_hash VARCHAR(64) UNIQUE (SHA-256), trang\_thai (1 hoạt động, 2 đã xoay, 3 đã thu hồi), het\_han\_luc | Refresh token xoay vòng, không lưu token thô |
| ngon\_ngu\_tham\_chieu | id (= cms\_db.ngon\_ngu.id, không FK vật lý), ma\_ngon\_ngu, ten, trang\_thai, cap\_nhat\_luc | Bản đọc danh mục ngôn ngữ của CMS để xác thực *ma\_ngon\_ngu* của bản dịch và trả cho ô chọn; seed vi = 1, en = 2, đồng bộ từ CMS ở giai đoạn sau |
| noi\_dung\_tham\_chieu | id (= cms\_db.noi\_dung.id, không FK vật lý), loai, tieu\_de, trang\_thai\_cms, su\_kien\_cuoi, cap\_nhat\_luc | Bản đọc cục bộ nạp từ cms-events để hiển thị và tìm theo tên nội dung ở màn phiên phát, quyền xem, lịch sử xem |

Cột row\_version INT NOT NULL DEFAULT 0 được thêm vào goi\_cuoc, goi\_cuoc\_da\_ngon\_ngu, gia\_goi\_cuoc, vung\_gia, tinh\_nang, tinh\_nang\_da\_ngon\_ngu, tinh\_nang\_goi để khóa lạc quan (ETag / If-Match) vì mỗi form con lưu độc lập.

Chỉ mục phục vụ màn quản trị: nguoi\_dung (GIN pg\_trgm trên lower(email), ho\_ten, so\_dien\_thoai; ngay\_tao DESC WHERE da\_xoa = false), thiet\_bi (ma\_nguoi\_dung; lan\_cuoi\_dang\_nhap DESC), thanh\_toan (ma\_nguoi\_dung; ma\_dang\_ky; ma\_thanh\_toan\_goc; ngay\_tao DESC; trigram trên ma\_giao\_dich\_cong), phien\_phat (bat\_dau\_luc DESC; ma\_nguoi\_dung, bat\_dau\_luc DESC; trang\_thai, bat\_dau\_luc DESC), phien\_dang\_nhap (thoi\_gian\_dang\_nhap DESC; ma\_nguoi\_dung; family\_id WHERE trang\_thai = 1), lich\_su\_xem (ma\_nguoi\_dung, phat\_luc DESC).

Mọi cột thời gian dùng TIMESTAMPTZ; phiên cơ sở dữ liệu đặt TIME ZONE 'Asia/Ho_Chi_Minh' (BR-32).

## 4.2 cms\_db (CMS)

**Cấu trúc**
├── ***Quản trị / RBAC***

│ ├── he\_thong

│ ├── tai\_khoan\_he\_thong

│ ├── vai\_tro

│ ├── quyen\_han

│ ├── tai\_khoan\_vai\_tro

│ └── vai\_tro\_quyen\_han

│

├── ***Nhà cung cấp***

│ └── nha\_cung\_cap\_noi\_dung

│

├── ***Nội dung***

│ ├── noi\_dung

│ ├── noi\_dung\_da\_ngon\_ngu

│ ├── thong\_tin\_noi\_dung

│ ├── phien\_ban\_noi\_dung

│ │

│ ├── mua\_phim

│ │

│ ├── dao\_dien

│ ├── dien\_vien

│ ├── thong\_tin\_noi\_dung\_dao\_dien

│ └── thong\_tin\_noi\_dung\_dien\_vien

│

├── ***Phân loại***

│ ├── danh\_muc

│ ├── danh\_muc\_da\_ngon\_ngu

│ └── noi\_dung\_danh\_muc

│

├── ***Media metadata***

│ ├── phu\_de

│ └── anh\_bia

│

├── ***Workflow***

│ ├── quy\_trinh\_noi\_dung

│ └── lich\_su\_quy\_trinh

│

└── ***Phát hành / bản quyền***

├── lich\_phat\_hanh

└── ban\_quyen\_noi\_dung

## ![Word image 3](assets/image-003.png)

[cms\_db.md](https://drive.google.com/file/d/11R2bBnOR5uFewEWoWHyamgk2FYqCXpxM/view?usp=drive_link)

● noi\_dung.loai: 0 = PHIM\_LE, 1 = SERIES, 2 = EPISODE.

● Cấu trúc series: Series → Season → Episode.

● mua\_phim.ma\_noi\_dung tham chiếu noi\_dung.id của Series.

● noi\_dung.ma\_mua\_phim tham chiếu mua\_phim.id đối với Episode.

● noi\_dung.ma\_noi\_dung\_cha giữ quan hệ cha/con tổng quát.

● thong\_tin\_noi\_dung có quan hệ 1-1 với noi\_dung.

● Đạo diễn và diễn viên được chuẩn hóa thành bảng riêng; quan hệ n-n thông qua:

○ thong\_tin\_noi\_dung\_dao\_dien

○ thong\_tin\_noi\_dung\_dien\_vien

● ngon\_ngu là bảng dùng chung cho toàn bộ dữ liệu đa ngôn ngữ trong cms\_db.

● thong\_tin\_noi\_dung.ma\_ngon\_ngu\_goc là FK tới ngon\_ngu.id, biểu diễn ngôn ngữ gốc của nội dung.

● Các bảng noi\_dung\_da\_ngon\_ngu, danh\_muc\_da\_ngon\_ngu, noi\_dung\_danh\_muc, phu\_de, anh\_bia sử dụng FK tới ngon\_ngu.id.

● ma\_tai\_nguyen\_media trong phien\_ban\_noi\_dung là khóa logic sang media\_db, không đặt FK vật lý do khác database.

### Bảng tai\_khoan\_he\_thong

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh tài khoản |
| ten_dang_nhap | VARCHAR(50) | NOT NULL | Tên đăng nhập CMS, 3–50 ký tự [A-Za-z0-9._-]; UNIQUE không phân biệt hoa thường trên bản ghi da_xoa=false |
| email | VARCHAR(255) |  | Email liên hệ, không dùng để đăng nhập, không bắt buộc duy nhất |
| mat\_khau\_hash | VARCHAR(255) | NOT NULL | Mật khẩu đã băm |
| ho\_ten | VARCHAR(150) | NOT NULL | Họ tên |
| loai | SMALLINT | NOT NULL, DEFAULT 0 | 0: nội bộ;<br><br>1: nhà cung cấp |
| ma\_nha\_cung\_cap | BIGINT | FK → nha\_cung\_cap\_noi\_dung.id | Nhà cung cấp gắn với tài khoản;<br><br>NULL nếu tài khoản nội bộ |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | 0: khóa;<br><br>1: hoạt động |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

### Bảng vai\_tro

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh vai trò |
| ma\_vai\_tro | VARCHAR(50) | NOT NULL, UNIQUE | Mã vai trò, ví dụ ADMIN, EDITOR, QC, PROVIDER |
| ten | VARCHAR(150) | NOT NULL | Tên vai trò |
| mo\_ta | VARCHAR(500) |  | Mô tả |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

### Bảng he\_thong

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh hệ thống |
| ma\_he\_thong | VARCHAR(30) | NOT NULL, UNIQUE (chưa xóa), ^[A-Z][A-Z0-9\_]{1,29}$ | CMS, OTT, AFFILIATE... không đổi sau khi tạo |
| ten | VARCHAR(150) | NOT NULL | Tên hệ thống |
| mo\_ta | VARCHAR(500) |  | Mô tả |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | 0 tạm ngưng, 1 hoạt động |
| khoa\_dich\_vu\_hash | VARCHAR(64) |  | SHA-256 của khóa dịch vụ; khóa thô chỉ hiện một lần lúc cấp |
| khoa\_dich\_vu\_luc | TIMESTAMPTZ |  | Thời điểm cấp khóa |
| ngay\_tao, ngay\_cap\_nhat, nguoi\_tao, da\_xoa, row\_version | | | Cột chung |

Dữ liệu khởi tạo: CMS, OTT, AFFILIATE. Không xóa hay tạm ngưng được CMS; không xóa được hệ thống còn quyền / menu.

### Bảng quyen\_han

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh quyền |
| ma\_he\_thong | BIGINT | FK → he\_thong.id, NOT NULL | Hệ thống sở hữu quyền / menu; không đổi sau khi tạo |
| ma\_quyen | VARCHAR(100) | NOT NULL, UNIQUE toàn cục | Mã quyền, ví dụ<br><br>noi\_dung.tao, noi\_dung.duyet. Hệ thống khác CMS: bắt đầu bằng mã hệ thống viết thường, ví dụ ott.user.xem |
| ten | VARCHAR(150) | NOT NULL | Tên quyền |
| duong\_dan | VARCHAR(500) |  | Đường dẫn giao diện của menu (bắt buộc với menu) |
| la\_menu | SMALLINT |  | 0: quyền API / hành động,<br>1: Menu chức năng hiển thị trên frontend của hệ thống |
| ma\_quyen\_cha | BIGINT | FK → quyen\_han.id | Cây cha-con trong cùng hệ thống; menu chỉ là con của menu; không tạo vòng |
| http\_method | VARCHAR(10) |  | Quyền API: GET, POST, PUT, PATCH, DELETE hoặc * |
| mau\_duong\_dan | VARCHAR(500) |  | Quyền API: mẫu PathPattern, ví dụ /users/{id}, /reports/\*\*. Phải khai báo cùng http\_method hoặc bỏ trống cả hai (quyền hành động thuần túy); menu không khai báo |
| thu\_tu | INT | NOT NULL, DEFAULT 0 | Thứ tự hiển thị trong cây |
| bieu\_tuong | VARCHAR(50) |  | Tên biểu tượng menu cho frontend |
| mo\_ta | VARCHAR(500) |  | Mô tả |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

### Bảng tai\_khoan\_vai\_tro

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh |
| ma\_tai\_khoan\_he\_thong | BIGINT | FK → tai\_khoan\_he\_thong.id,<br><br>NOT NULL | Tài khoản |
| ma\_vai\_tro | BIGINT | FK → vai\_tro.id,<br><br>NOT NULL | Vai trò |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |

Ràng buộc: UNIQUE (ma\_tai\_khoan\_he\_thong, ma\_vai\_tro).

### Bảng vai\_tro\_quyen\_han

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh |
| ma\_vai\_tro | BIGINT | FK → vai\_tro.id,<br><br>NOT NULL | Vai trò |
| ma\_quyen\_han | BIGINT | FK → quyen\_han.id, NOT NULL | Quyền hạn |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |

Ràng buộc: UNIQUE (ma\_vai\_tro, ma\_quyen\_han).

### Bảng nha\_cung\_cap\_noi\_dung

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh nhà cung cấp |
| ten | VARCHAR(255) | NOT NULL | Tên nhà cung cấp |
| loai | VARCHAR(50) |  | Loại:<br><br>studio, creator, in\_house |
| lien\_he | VARCHAR(255) |  | Thông tin liên hệ |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | 0: ngừng hợp tác;<br><br>1: hoạt động |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

### Bảng ngon\_ngu

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh ngôn ngữ |
| ma\_ngon\_ngu | VARCHAR(10) | NOT NULL, UNIQUE | Mã ngôn ngữ, ví dụ vi, en, ko |
| ten | VARCHAR(100) | NOT NULL | Tên ngôn ngữ |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | 0: ngừng sử dụng;<br><br>1: đang sử dụng |
| thu\_tu | SMALLINT |  | Thứ tự hiển thị |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

### Bảng noi\_dung

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh nội dung |
| ma\_nha\_cung\_cap | BIGINT | FK → nha\_cung\_cap\_noi\_dung.id, NOT NULL | Nhà cung cấp nội dung |
| tieu\_de | VARCHAR(500) | NOT NULL | Tiêu đề mặc định/fallback |
| mo\_ta | TEXT |  | Mô tả mặc định/fallback |
| loai | SMALLINT | NOT NULL, DEFAULT 0 | 0: phim lẻ;<br><br>1: series;<br><br>2: episode |
| ma\_noi\_dung\_cha | BIGINT | FK → noi\_dung.id | Nội dung cha tổng quát |
| ma\_mua\_phim | BIGINT | FK → mua\_phim.id | Season của Episode |
| so\_tap | INT |  | Số tập; chủ yếu dùng cho Series |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 0 | 0: bản nháp;<br><br>1: chờ QC;<br><br>2: sẵn sàng;<br><br>3: xuất bản;<br><br>4: gỡ xuất bản |
| ma\_phien\_ban\_mac\_dinh | BIGINT | FK → phien\_ban\_noi\_dung.id | Phiên bản media mặc định |
| nam\_phat\_hanh | SMALLINT |  | Năm phát hành |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |
| da\_tung\_xuat\_ban | BOOLEAN | NOT NULL, DEFAULT false | Đặt true ở lần xuất bản đầu tiên; chặn xóa nội dung (BR-11a) |
| row\_version | INT | NOT NULL, DEFAULT 0 | Khóa lạc quan; tăng mỗi lần cập nhật, dùng ETag/If-Match |
| trace\_id | UUID | NOT NULL | Dùng để theo dõi toàn bộ quá trình xử lý nội dung khi gửi sang Media API |

### Bảng mua\_phim

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh Season |
| ma\_noi\_dung | BIGINT | FK → noi\_dung.id,<br><br>NOT NULL | Series chứa Season |
| so\_mua | INT | NOT NULL | Số thứ tự Season |
| tieu\_de | VARCHAR(500) |  | Tiêu đề Season |
| mo\_ta | TEXT |  | Mô tả Season |
| nam\_phat\_hanh | SMALLINT |  | Năm phát hành |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL,<br><br>DEFAULT false | Xóa mềm |

Ràng buộc: UNIQUE (ma\_noi\_dung, so\_mua).

### Bảng noi\_dung\_da\_ngon\_ngu

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh bản dịch |
| ma\_noi\_dung | BIGINT | FK → noi\_dung.id, NOT NULL | Nội dung |
| ma\_ngon\_ngu | BIGINT | FK → ngon\_ngu.id, NOT NULL | Ngôn ngữ |
| tieu\_de | VARCHAR(500) | NOT NULL | Tiêu đề theo ngôn ngữ |
| mo\_ta\_ngan | VARCHAR(1000) |  | Mô tả ngắn |
| mo\_ta | TEXT |  | Mô tả chi tiết |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL,<br><br>DEFAULT false | Xóa mềm |

Ràng buộc: UNIQUE (ma\_noi\_dung, ma\_ngon\_ngu).

### Bảng thong\_tin\_noi\_dung

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh metadata |
| ma\_noi\_dung | BIGINT | FK → noi\_dung.id, NOT NULL, UNIQUE | Nội dung |
| quoc\_gia | VARCHAR(100) |  | Quốc gia sản xuất |
| ma\_ngon\_ngu\_goc | BIGINT | FK → ngon\_ngu.id, NOT NULL | Ngôn ngữ gốc của nội dung |
| the\_loai\_tag | JSONB |  | Danh sách tag thể loại |
| thoi\_luong\_phut | INT |  | Thời lượng tính bằng phút |
| do\_tuoi | SMALLINT |  | Mức độ tuổi |
| xep\_hang | VARCHAR(20) |  | Xếp hạng nội dung |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

### Bảng dao\_dien

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh đạo diễn |
| ho\_ten | VARCHAR(255) | NOT NULL | Họ tên |
| ten\_khac | VARCHAR(255) |  | Tên khác/bí danh |
| quoc\_tich | VARCHAR(100) |  | Quốc tịch |
| ngay\_sinh | DATE |  | Ngày sinh |
| gioi\_thieu | TEXT |  | Tiểu sử/giới thiệu |
| anh\_dai\_dien | VARCHAR(255) |  | Khóa ảnh đại diện trên R2 |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

### Bảng dien\_vien

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh diễn viên |
| ho\_ten | VARCHAR(255) | NOT NULL | Họ tên |
| ten\_khac | VARCHAR(255) |  | Tên khác/bí danh |
| quoc\_tich | VARCHAR(100) |  | Quốc tịch |
| ngay\_sinh | DATE |  | Ngày sinh |
| gioi\_thieu | TEXT |  | Tiểu sử/giới thiệu |
| anh\_dai\_dien | VARCHAR(255) |  | Khóa ảnh đại diện trên R2 |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

### Bảng noi\_dung\_dao\_dien

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh quan hệ |
| ma\_noi\_dung | BIGINT | FK → noi\_dung.id, NOT NULL | Nội dung |
| ma\_dao\_dien | BIGINT | FK → dao\_dien.id, NOT NULL | Đạo diễn |
| thu\_tu | SMALLINT |  | Thứ tự hiển thị |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |

Ràng buộc: UNIQUE (ma\_noi\_dung, ma\_dao\_dien).

### Bảng noi\_dung\_dien\_vien

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh quan hệ |
| ma\_noi\_dung | BIGINT | FK → noi\_dung.id,<br><br>NOT NULL | Metadata nội dung |
| ma\_dien\_vien | BIGINT | FK → dien\_vien.id,<br><br>NOT NULL | Diễn viên |
| thu\_tu | SMALLINT |  | Thứ tự hiển thị |
| vai\_dien | VARCHAR(255) |  | Vai diễn |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |

Ràng buộc: UNIQUE (ma\_noi\_dung, ma\_dien\_vien).

### Bảng phien\_ban\_noi\_dung

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh phiên bản |
| ma\_noi\_dung | BIGINT | FK → noi\_dung.id,<br><br>NOT NULL | Nội dung |
| so\_phien\_ban | INT | NOT NULL | Số thứ tự phiên bản |
| ma\_tai\_nguyen\_media | BIGINT |  | ID tài nguyên media, khóa logic sang media\_db |
| ma\_cong\_viec\_media | BIGINT |  | ID công việc transcode, khóa logic sang media\_db |
| ma\_tep\_nguon | BIGINT | FK → tep\_tai\_len.id | Video gốc đã tải lên và xác nhận |
| trang\_thai\_media | SMALLINT | NOT NULL, DEFAULT 0 | Ảnh chụp trạng thái media lần cuối nhận từ Media API (nguồn sự thật vẫn ở media\_db): 0: chưa có nguồn sẵn sàng; 1: nguồn sẵn sàng; 2: đang xử lý; 3: sẵn sàng; 4: lỗi |
| khoa\_hls, khoa\_dash | VARCHAR(500) |  | Khóa manifest HLS / DASH do Media API trả về |
| chat\_luong | JSONB |  | Danh sách chất lượng, ví dụ ["1080p","720p"] |
| co\_drm | BOOLEAN | NOT NULL, DEFAULT false | Phiên bản có mã hóa DRM |
| loi\_media | VARCHAR(1000) |  | Thông báo lỗi gần nhất |
| ghi\_chu | VARCHAR(500) |  | Ghi chú phiên bản |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| row\_version | INT | NOT NULL, DEFAULT 0 | Khóa lạc quan |

Quy tắc: số phiên bản do máy chủ cấp tuần tự dưới khóa dòng nội dung. Không xóa phiên bản đã có tài nguyên media hoặc đang là phiên bản phục vụ. Callback từ Media API chỉ chuyển trạng thái theo chiều tiến; callback đến muộn không kéo lùi trạng thái.

Luồng chuyển trạng thái do callback (hợp đồng ở mục "Hợp đồng giữa CMS và Media API"): `MEDIA_SOURCE_READY` đưa phiên bản sang 1 (nội dung giữ nguyên trạng thái, người dùng chủ động "Gửi QC"); QC bấm Transcode (CMS gọi `/media/jobs`, phiên bản sang 2); `MEDIA_READY` đưa phiên bản sang 3 (và đặt làm phiên bản phục vụ nếu nội dung chưa có); `MEDIA_FAILED` đưa sang 4. Nội dung chỉ sang SAN\_SANG khi QC bấm "Duyệt" ở bước workflow, và điều kiện là phiên bản phục vụ đã có media sẵn sàng.

Ràng buộc: UNIQUE (ma\_noi\_dung, so\_phien\_ban).

###

### Bảng danh\_muc

dùng cho cấu trúc phân loại chính thức, Ví dụ: Category: Phim > Hàn Quốc

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh danh mục |
| ten | VARCHAR(150) | NOT NULL | Tên mặc định/fallback |
| mo\_ta | TEXT |  | Mô tả mặc định/fallback |
| ma\_danh\_muc\_cha | BIGINT | FK → danh\_muc.id | Danh mục cha |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

###

### Bảng danh\_muc\_da\_ngon\_ngu

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh bản dịch |
| ma\_danh\_muc | BIGINT | FK → danh\_muc.id, NOT NULL | Danh mục |
| ma\_ngon\_ngu | BIGINT | FK → ngon\_ngu.id, NOT NULL | Ngôn ngữ |
| ten | VARCHAR(150) | NOT NULL | Tên danh mục theo ngôn ngữ |
| mo\_ta | TEXT |  | Mô tả theo ngôn ngữ |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL,<br><br>DEFAULT false | Xóa mềm |

Ràng buộc: UNIQUE (ma\_danh\_muc, ma\_ngon\_ngu).

### Bảng noi\_dung\_danh\_muc

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh quan hệ |
| ma\_noi\_dung | BIGINT | FK → noi\_dung.id, NOT NULL | Nội dung |
| ma\_danh\_muc | BIGINT | FK → danh\_muc.id, NOT NULL | Danh mục |
| ma\_ngon\_ngu | BIGINT | FK → ngon\_ngu.id, NOT NULL | Ngôn ngữ hiển thị của danh mục khi gán cho nội dung (tên danh mục lấy từ danh\_muc\_da\_ngon\_ngu theo ngôn ngữ này) |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |

Ràng buộc: UNIQUE (ma\_noi\_dung, ma\_danh\_muc, ma\_ngon\_ngu).

### Bảng phu\_de

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh phụ đề |
| ma\_noi\_dung | BIGINT | FK → noi\_dung.id, NOT NULL | Nội dung |
| ma\_ngon\_ngu | BIGINT | FK → ngon\_ngu.id, NOT NULL | Ngôn ngữ phụ đề |
| khoa\_tep | VARCHAR(255) | NOT NULL | Khóa tệp phụ đề trên R2 |
| dinh\_dang | VARCHAR(10) | NOT NULL | SRT, VTT, TTML |
| loai\_phu\_de | VARCHAR(20) | NOT NULL, DEFAULT NORMAL | NORMAL, FORCED, SDH |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

### Bảng anh\_bia

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh ảnh |
| ma\_noi\_dung | BIGINT | FK → noi\_dung.id,<br><br>NOT NULL | Nội dung |
| loai | SMALLINT | NOT NULL | 0: poster;<br><br>1: thumbnail;<br><br>2: backdrop |
| khoa\_tep | VARCHAR(255) | NOT NULL | Khóa tệp trên R2 |
| ten\_tep | VARCHAR(255) |  | Tên tệp |
| mime\_type | VARCHAR(100) |  | MIME type |
| kich\_thuoc | BIGINT |  | Kích thước tệp, byte |
| chieu\_rong | INT |  | Chiều rộng ảnh |
| chieu\_cao | INT |  | Chiều cao ảnh |
| checksum | VARCHAR(128) |  | Checksum |
| ma\_ngon\_ngu | BIGINT | FK → ngon\_ngu.id | Ngôn ngữ của ảnh;<br><br>NULL nếu ảnh dùng chung |
| la\_mac\_dinh | BOOLEAN | NOT NULL, DEFAULT false | Ảnh mặc định của loại ảnh |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

###

### Bảng quy\_trinh\_noi\_dung

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh |
| ma\_noi\_dung | BIGINT | FK → noi\_dung.id,<br><br>NOT NULL, UNIQUE | Nội dung |
| buoc | VARCHAR(50) | NOT NULL | BAN\_NHAP,<br><br>QC,<br><br>DUYET,<br><br>XUAT\_BAN,<br><br>GO\_XUAT\_BAN |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 0 | 0: đang xử lý;<br><br>1: hoàn tất;<br><br>2: trả lại |
| ma\_nguoi\_phu\_trach | BIGINT | FK → tai\_khoan\_he\_thong.id | Người phụ trách |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |

### Bảng lich\_su\_quy\_trinh (Đây là bảng chỉ ghi thêm)

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh lịch sử |
| ma\_noi\_dung | BIGINT | FK → noi\_dung.id,<br><br>NOT NULL | Nội dung |
| buoc\_tu | VARCHAR(50) |  | Bước trước |
| buoc\_den | VARCHAR(50) | NOT NULL | Bước sau |
| ma\_nguoi\_thuc\_hien | BIGINT | FK → tai\_khoan\_he\_thong.id | Người thực hiện |
| thoi\_diem | TIMESTAMPTZ | NOT NULL | Thời điểm chuyển |
| ghi\_chu | VARCHAR(500) |  | Ghi chú |

### Bảng lich\_phat\_hanh

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh lịch |
| ma\_noi\_dung | BIGINT | FK → noi\_dung.id,<br><br>NOT NULL | Nội dung |
| phat\_hanh\_luc | TIMESTAMPTZ | NOT NULL | Thời điểm bắt đầu phát hành |
| het\_hieu\_luc\_luc | TIMESTAMPTZ |  | Thời điểm kết thúc |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 0 | 0: chờ;<br><br>1: đang hiệu lực;<br><br>2: đã gỡ |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

### Bảng ban\_quyen\_noi\_dung

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh bản quyền |
| ma\_noi\_dung | BIGINT | FK → noi\_dung.id, NOT NULL | Nội dung |
| bat\_dau\_luc | TIMESTAMPTZ | NOT NULL | Thời điểm bắt đầu quyền phân phối |
| ket\_thuc\_luc | TIMESTAMPTZ |  | Thời điểm kết thúc quyền phân phối |
| lanh\_tho | JSONB |  | Danh sách mã quốc gia được phép phân phối |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | ID tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

### Bảng cap\_quyen\_xem

Danh mục các cấp quyền xem nội dung.

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh |
| ma\_cap\_quyen | VARCHAR(50) | NOT NULL, UNIQUE | Mã cấp quyền |
| thu\_tu | SMALLINT | NOT NULL, UNIQUE | Thứ tự cấp quyền |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | 0: ngừng,<br><br>1: hoạt động |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Cập nhật gần nhất |
| nguoi\_tao | BIGINT |  | Tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

**Dữ liệu mặc định:**

| **ma\_cap\_quyen** | **thu\_tu** | **Ý nghĩa** |
| --- | --- | --- |
| FREE | 1 | Quyền miễn phí |
| VIP | 2 | Quyền VIP |
| VIP\_PRO | 3 | Quyền VIP PRO |

Quy tắc kế thừa:

VIP\_PRO

↓ kế thừa

VIP

↓ kế thừa

FREE

Tức người có VIP\_PRO được hưởng quyền của VIP và FREE.

### Bảng chinh\_sach\_quyen\_xem

Bảng xác định một nội dung yêu cầu cấp quyền nào và được phép xem như thế nào.

| **Trường** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh |
| ma\_noi\_dung | BIGINT | NOT NULL | → noi\_dung.id |
| ma\_cap\_quyen | BIGINT | NOT NULL | → cap\_quyen\_xem.id |
| duoc\_xem | BOOLEAN | NOT NULL, DEFAULT true | Có được xem hay không |
| chat\_luong\_toi\_da | VARCHAR(20) |  | Chất lượng tối đa |
| so\_tap\_toi\_da | INT |  | Số tập tối đa được xem |
| ngay\_bat\_dau | TIMESTAMPTZ |  | Bắt đầu hiệu lực |
| ngay\_ket\_thuc | TIMESTAMPTZ |  | Kết thúc hiệu lực |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | 0: không áp dụng, 1: áp dụng |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Cập nhật |
| nguoi\_tao | BIGINT |  | Tài khoản tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

Ví dụ phim lẻ

Phim A miễn phí nhưng giới hạn 480p:

| **Nội dung** | **Cấp** | **Xem** | **Chất lượng** |
| --- | --- | --- | --- |
| Phim A | FREE | Có | 480p |
| Phim A | VIP | Có | 1080p |
| Phim A | VIP\_PRO | Có | 1080p |

Ví dụ series

Series B:

| **Nội dung** | **Cấp** | **Xem** | **Số tập tối đa** | **Chất lượng** |
| --- | --- | --- | --- | --- |
| Series B | FREE | Có | 2 | 480p |
| Series B | VIP | Có | NULL | 1080p |
| Series B | VIP\_PRO | Có | NULL | 1080p |

NULL ở so\_tap\_toi\_da = không giới hạn.

Ràng buộc bổ sung: FK tới noi\_dung.id và cap\_quyen\_xem.id; UNIQUE (ma\_noi\_dung, ma\_cap\_quyen) trên bản ghi da\_xoa=false; so\_tap\_toi\_da chỉ hợp lệ khi nội dung là series; khoảng ngay\_bat\_dau–ngay\_ket\_thuc của cùng (nội dung, cấp quyền) không chồng lấn.

### Bảng bổ sung cho CMS API

Các bảng kỹ thuật phục vụ idempotency, sự kiện, xác thực và kiểm toán. Cấu trúc cột xem cms\_db.md.

| **Bảng** | **Mục đích** | **Ràng buộc chính** |
| --- | --- | --- |
| idempotency\_key | Lưu kết quả request có header Idempotency-Key (POST tạo mới, transcode, presign) | UNIQUE (ma\_tai\_khoan, khoa); hết hạn thì dọn định kỳ |
| su\_kien\_cho\_gui | Transactional outbox, ghi cùng transaction với dữ liệu, tiến trình riêng đẩy lên Kafka (topic cms-events). Cột loi lưu lỗi gần nhất | Chỉ ghi thêm; trạng thái 0 chờ gửi, 1 đã gửi, 2 bỏ cuộc (không dựng được thông điệp) |
| su\_kien\_media\_da\_nhan | Inbox chống trùng callback từ Media API | UNIQUE (nguon, ma\_su\_kien) |
| phien\_dang\_nhap | Refresh token xoay vòng, thu hồi theo family\_id, phát hiện tái sử dụng | Lưu hash, không lưu token thô |
| nhat\_ky\_kiem\_toan | Ghi ai sửa gì, giá trị trước/sau | Chỉ ghi thêm |
| viec\_don\_dep | Hàng đợi dọn tệp khi xóa nội dung: loại PREFIX (xóa tệp tiền tố c{id}/ trên R2), OBJECT (xóa một tệp), MULTIPART (hủy phiên tải nhiều phần, mục tiêu dạng khóa#uploadId) và MEDIA\_RESOURCE (yêu cầu Media API xóa tài nguyên). Ghi cùng transaction, chạy sau commit, tự thử lại với backoff (tối đa 10 lần) | Trạng thái 0 chờ, 1 xong, 2 bỏ cuộc; xử lý bằng SELECT ... FOR UPDATE SKIP LOCKED |
| tep\_tai\_len | Theo dõi presigned URL đã cấp, xác nhận upload, dọn tệp mồ côi. Thêm ten\_tep, mime\_type, ma\_phien\_ban (video gốc). Trạng thái: 0 đã cấp URL, 1 đã xác nhận, 2 hết hạn, 3 đã gắn vào phụ đề / ảnh bìa / phiên bản. Tải nhiều phần thêm ma\_phien\_nhieu\_phan (uploadId của R2), kich\_thuoc\_phan, so\_phan | UNIQUE khoa\_tep; khóa tệp do server sinh, không nhận từ client |

Quy ước chung: unique của bảng chủ dùng unique index một phần WHERE da\_xoa = false; các FK vòng (noi\_dung.ma\_phien\_ban\_mac\_dinh, noi\_dung.ma\_mua\_phim) khai báo DEFERRABLE INITIALLY DEFERRED.

### Hợp đồng giữa CMS và Media API

Xác thực bằng khóa dịch vụ (header X-Service-Key). Callback ký HMAC-SHA256 trên body thô (header X-Signature, X-Timestamp; lệch quá 5 phút bị từ chối).

CMS gọi Media: POST /media/jobs, header Idempotency-Key bắt buộc.

Quy ước truy cập tệp (đã chốt): CMS chỉ cấp presigned URL cho client tải lên bucket original-private. CMS không cấp URL nào cho Media API; yêu cầu job chỉ mang bucket và khóa tệp, Media API tự đọc bucket original-private bằng khóa R2 riêng (quyền đọc) và tự ghi bucket streaming. CMS không có quyền đọc bucket streaming.

```json
{ "contentId": 10001, "versionNo": 1, "traceId": "uuid",
  "source": { "bucket": "original-private", "key": "c10001/v1/movie001.mkv", "checksum": "sha256:..." },
  "profile": "HLS_DASH_1080P", "drm": false }
```

Phản hồi 202: { "jobId": 55, "mediaResourceId": 77, "status": "CREATED" }. Gửi lại cùng Idempotency-Key trả đúng jobId cũ.

Media gọi CMS: POST /internal/media/callbacks.

```json
{ "eventId": "evt-123", "type": "MEDIA_SOURCE_READY | MEDIA_READY | MEDIA_FAILED",
  "contentId": 10001, "versionNo": 1, "mediaResourceId": 77, "traceId": "uuid",
  "manifests": { "hls": "c10001/v1/hls/master.m3u8", "dash": "c10001/v1/dash/manifest.mpd" },
  "qualities": ["1080p", "720p"], "drm": false, "error": null }
```

CMS ghi su\_kien\_media\_da\_nhan (trùng eventId thì bỏ qua và trả 200) rồi cập nhật ma\_tai\_nguyen\_media của phiên bản. MEDIA\_SOURCE\_READY đánh dấu phiên bản có video gốc sẵn sàng (không đổi trạng thái nội dung); MEDIA\_READY đánh dấu phiên bản sẵn sàng; MEDIA\_FAILED ghi lỗi. Trạng thái nội dung chỉ đổi qua workflow do người dùng thao tác.

CMS gọi Media khi xóa nội dung: `DELETE /media/resources/{mediaResourceId}`, header Idempotency-Key và X-Service-Key. Idempotent: tài nguyên không còn thì trả 404 và CMS coi là thành công. Media xóa các tệp đầu ra ở bucket streaming (CMS không có quyền ghi bucket này).

## 4.3 affiliate\_db (Publisher) [affiliate\_db.md](https://drive.google.com/file/d/19lnG9tGb3Ni_F4ywY23nOJGwgg6wdWU3/view?usp=drive_link)

Các cấu phần:

## ![Word image 4](assets/image-004.png)

##

![Word image 5](assets/image-005.png)

### Bảng nha\_xuat\_ban

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh nhà xuất bản |
| email | VARCHAR(255) | NOT NULL, UNIQUE | Email đăng nhập |
| mat\_khau\_hash | VARCHAR(255) | NOT NULL | Mật khẩu đã băm |
| ten | VARCHAR(255) | NOT NULL | Tên nhà xuất bản |
| lien\_he | VARCHAR(255) |  | Thông tin liên hệ |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 0 | Trạng thái<br><br>0: chờ duyệt,<br><br>1: hoạt động,<br><br>2: tạm ngưng |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | Người tạo. Khóa logic tới *cms.tai\_khoan\_he\_thong* |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

### Bảng chien\_dich

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh chiến dịch |
| ma\_chien\_dich | VARCHAR(64) | NOT NULL, UNIQUE | Mã nghiệp vụ chiến dịch |
| ten | VARCHAR(255) | NOT NULL | Tên chiến dịch |
| bat\_dau\_luc | TIMESTAMPTZ | NOT NULL | Thời điểm bắt đầu |
| ket\_thuc\_luc | TIMESTAMPTZ |  | Thời điểm kết thúc<br><br>NULL nếu không giới hạn |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | Trạng thái<br><br>0: tạm dừng,<br><br>1: đang chạy,<br><br>2: kết thúc |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | Người quản trị tạo chiến dịch<br><br>*Khóa logic → cms\_db.tai\_khoan\_he\_thong.id* |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

### Bảng gan\_chien\_dich\_nha\_xuat\_ban

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh lần gán |
| ma\_chien\_dich | BIGINT | FK → chien\_dich.id, NOT NULL | Chiến dịch được gán |
| ma\_nha\_xuat\_ban | BIGINT | FK → nha\_xuat\_ban.id, NOT NULL | Nhà xuất bản được gán |
| gan\_luc | TIMESTAMPTZ | NOT NULL | Thời điểm gán |
| huy\_luc | TIMESTAMPTZ |  | NULL nếu còn hiệu lực |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | Trạng thái gán<br><br>0: hủy,<br><br>1: đang hoạt động |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ | NOT NULL | Thời điểm cập nhật |
| nguoi\_tao | BIGINT | NOT NULL | Quản trị viên thực hiện gán<br><br>Khóa logic → cms\_db.tai\_khoan\_he\_thong.id |

**Ràng buộc:** check UNIQUE(ma\_chien\_dich, ma\_nha\_xuat\_ban) trạng thái = 1, không xóa dữ liệu, chỉ được chuyển về trạng thái hủy khi không hoạt động nữa

### Bảng chien\_dich\_noi\_dung

Dùng để cấu hình chiến dịch có thể áp dụng cho **một hoặc nhiều phim**.

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh |
| ma\_chien\_dich | BIGINT | FK → chien\_dich.id, NOT NULL | Chiến dịch được gán |
| ma\_noi\_dung | BIGINT | NOT NULL | Nội dung phim được áp dụng<br><br>Logical FK → cms\_db.noi\_dung.id |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | Trạng thái gán<br><br>0: tạm dừng,<br><br>1: đang hoạt động |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ | NOT NULL | Thời điểm cập nhật |
| nguoi\_tao | BIGINT | NOT NULL | Quản trị viên thực hiện gán<br><br>Khóa logic → cms\_db.tai\_khoan\_he\_thong.id |

Ràng buộc: check UNIQUE(ma\_chien\_dich, ma\_noi\_dung) trên trạng thái = 1

### Bảng lien\_ket\_theo\_doi

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh link |
| ma\_gan\_chien\_dich\_nha\_xuat\_ban | BIGINT | FK → gan\_chien\_dich\_nha\_xuat\_ban .id, NOT NULL | Lần gán campaign cho publisher |
| ma\_noi\_dung | BIGINT |  | Nội dung được quảng bá<br>Khóa logic sang cms\_db; NULL nếu quảng bá trang chủ |
| ma\_theo\_doi | VARCHAR(64) | NOT NULL, UNIQUE | Mã theo dõi trong URL, sử dụng CSPRNG / SecureRandom |
| url\_dich | VARCHAR(1000) | NOT NULL | URL đích |
| nguon | VARCHAR(100) |  | Nguồn traffic: facebook, tiktok, blog... |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | Trạng thái link<br>0: tắt, 1: hoạt động |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | Người tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

### Bảng gan\_ket\_nguon

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh attribution |
| ma\_an\_danh | VARCHAR(64) | NOT NULL, UNIQUE | Anonymous ID: First-click attribution |
| ma\_nguoi\_dung | BIGINT |  | Người dùng: Khóa logic sang ott\_db.nguoi\_dung; điền khi biết danh tính |
| ma\_gan\_chien\_dich\_nha\_xuat\_ban | BIGINT | FK → gan\_chien\_dich\_nha\_xuat\_ban.id,<br><br>NOT NULL | Lần gán campaign–publisher |
| ma\_lien\_ket | BIGINT | FK → lien\_ket\_theo\_doi.id, NOT NULL | Link tạo ra attribution |
| ma\_theo\_doi | VARCHAR(64) |  | Mã tracking: Snapshot từ link |
| ma\_nha\_xuat\_ban | BIGINT | FK → nha\_xuat\_ban.id, NOT NULL | Snapshot nhà xuất bản |
| ma\_chien\_dich | BIGINT | FK → chien\_dich.id, NOT NULL | Snapshot Chiến dịch |
| nguon\_dau | VARCHAR(100) |  | Nguồn của click đầu tiên |
| mo\_hinh\_quy\_gan | VARCHAR(30) | NOT NULL, DEFAULT 'FIRST\_CLICK' | Mô hình attribution: FIRST\_CLICK |
| quy\_gan\_luc | TIMESTAMPTZ | NOT NULL | Thời điểm attribution |
| het\_han\_luc | TIMESTAMPTZ | NOT NULL | Hết cửa sổ attribution: Mặc định 30 ngày |

### Bảng luot\_nhap

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh lượt nhấp |
| ma\_lien\_ket | BIGINT | FK → *lien\_ket\_theo\_doi*.*id*, NOT NULL | Link được nhấp |
| ma\_an\_danh | VARCHAR(64) | NOT NULL | Mã ẩn danh: Cookie/anonymous ID |
| dia\_chi\_ip | VARCHAR(45) |  | Địa chỉ IP, hỗ trợ IPv6 |
| user\_agent | VARCHAR(500) |  | User agent |
| nguon | VARCHAR(100) |  | Nguồn traffic |
| nhap\_luc | TIMESTAMPTZ | NOT NULL | Thời điểm click |

### Bảng luot\_xem

Dùng để ghi nhận VIEW độc lập với chuyển đổi thanh toán. Việc event có đủ điều kiện nhận tiền hay không do quy\_tac\_hoa\_hong quyết định.

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh lượt xem |
| ma\_lien\_ket | BIGINT | FK → *lien\_ket\_theo\_doi*.*id*, NOT NULL | Link được nhấp |
| ma\_nguoi\_dung | BIGINT |  | Người dùng: Khóa logic sang ott\_db.nguoi\_dung; điền khi biết danh tính |
| ma\_noi\_dung | BIGINT | NOT NULL | Nội dung phim được áp dụng<br><br>Logical FK → cms\_db.noi\_dung.id |
| ma\_an\_danh | VARCHAR(64) | NOT NULL | Mã ẩn danh: Cookie/anonymous ID |
| thoi\_gian\_xem | INTEGER | NOT NULL | Số giây xem hợp lệ |
| ty\_le\_xem | NUMERIC(5,2) |  | Tỷ lệ % đã xem |
| xem\_luc | TIMESTAMPTZ | NOT NULL | Thời điểm ghi nhận |
| ma\_su\_kien\_nguon | VARCHAR(255) | UNIQUE | ID event nguồn |

### Bảng chuyen\_doi

Bản ghi conversion là snapshot của Affiliate từ event thanh toán; ott\_db vẫn là source of truth của doanh thu.

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh conversion |
| ma\_gan\_ket | BIGINT | FK → gan\_ket\_nguon.id, NOT NULL | Attribution |
| ma\_nha\_xuat\_ban | BIGINT | FK → nha\_xuat\_ban.id, NOT NULL | Snapshot - Nhà xuất bản hưởng conversion |
| ma\_chien\_dich | BIGINT | FK → chien\_dich.id | Snapshot - Chiến dịch |
| ma\_lien\_ket | BIGINT | FK → lien\_ket\_theo\_doi.id | Snapshot - Link tracking |
| ma\_theo\_doi | VARCHAR(64) |  | Snapshot - Mã tracking |
| ma\_an\_danh | VARCHAR(64) | NOT NULL | Anonymous ID |
| ma\_nguoi\_dung | BIGINT |  | Người dùng<br>Khóa logic → ott\_db.nguoi\_dung.id |
| ma\_dang\_ky | BIGINT |  | Đăng ký<br><br>Khóa logic → ott\_db.dang\_ky.id |
| ma\_goi\_cuoc | BIGINT |  | Gói cước<br>Khóa logic → ott\_db.goi\_cuoc.id |
| ma\_thanh\_toan | BIGINT | NOT NULL, UNIQUE | ID thanh toán<br>Khóa logic → ott\_db.thanh\_toan.id |
| ma\_giao\_dich\_cong | VARCHAR(255) |  | Mã giao dịch cổng thanh toán |
| ma\_su\_kien\_nguon | VARCHAR(255) | NOT NULL, UNIQUE | Event ID Kafka: Dùng chống xử lý trùng |
| loai\_giao\_dich | SMALLINT | NOT NULL | Loại giao dịch<br><br>0: đăng ký mới, 1: gia hạn |
| so\_tien\_gross | NUMERIC(12,2) | NOT NULL | Giá trị thanh toán trước giảm trừ |
| so\_tien\_giam\_gia | NUMERIC(12,2) | NOT NULL, DEFAULT 0 | Giảm giá |
| so\_tien\_thue | NUMERIC(12,2) | NOT NULL, DEFAULT 0 | Thuế |
| so\_tien\_net | NUMERIC(12,2) | NOT NULL | Giá trị thực tính hoa hồng |
| don\_vi\_tien | VARCHAR(3) | NOT NULL | Đơn vị tiền |
| chuyen\_doi\_luc | TIMESTAMPTZ | NOT NULL | Thời điểm conversion |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 0 | Trạng thái: 0: chờ xử lý, 1: hợp lệ, 2: nghi ngờ, 3: loại |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |

### Bảng quy\_tac\_hoa\_hong

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh rule |
| ma\_chien\_dich | BIGINT | FK → chien\_dich.id | Chiến dịch |
| ma\_nha\_xuat\_ban | BIGINT | FK → nha\_xuat\_ban.id | Nhà xuất bản áp dụng. NULL = rule mặc định |
| loai\_su\_kien | SMALLINT | NOT NULL, DEFAULT 0 | Sự kiện tạo hoa hồng<br><br>0: CLICK,<br><br>1: VIEW,<br><br>2: DANG\_KY\_MOI,<br><br>3: GIA\_HAN,<br>4: KHAC |
| loai\_tinh | SMALLINT | NOT NULL | Phương thức tính<br><br>0 cố định, 1 phần trăm |
| gia\_tri | NUMERIC(12,2) | NOT NULL | Giá trị rule: % hoặc số tiền |
| co\_so\_tinh | SMALLINT | NOT NULL | Cơ sở tính<br><br>0: gross,<br><br>1: gross sau giảm giá,<br><br>2: net,<br><br>3: net không thuế |
| don\_vi\_tien | VARCHAR(3) |  | Đơn vị tiền<br><br>Bắt buộc khi loai = loai\_tinh |
| gioi\_han\_so\_luong | INTEGER |  | Giới hạn số lần<br><br>NULL = không giới hạn |
| gioi\_han\_so\_tien | NUMERIC(12,2) |  | Giới hạn tiền |
| thoi\_gian\_xem\_toi\_thieu | INTEGER |  | Điều kiện view tính theo giây |
| ty\_le\_xem\_toi\_thieu | NUMERIC(5,2) |  | Điều kiện view tính theo % |
| hieu\_luc\_tu | TIMESTAMPTZ | NOT NULL | Hiệu lực từ |
| hieu\_luc\_den | TIMESTAMPTZ |  | Hiệu lực đến<br><br>NULL nếu còn hiệu lực |
| trang\_thai | SMALLINT |  |  |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |
| nguoi\_tao | BIGINT |  | Người tạo |
| da\_xoa | BOOLEAN | NOT NULL, DEFAULT false | Xóa mềm |

### Bảng giao\_dich\_hoa\_hong

Ledger hoa hồng phát sinh từ một conversion; lưu snapshot cách tính để audit lịch sử.

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh giao dịch hoa hồng |
| ma\_nha\_xuat\_ban | BIGINT | FK → nha\_xuat\_ban.id, NOT NULL | Nhà xuất bản nhận |
| ma\_chuyen\_doi | BIGINT | FK → chuyen\_doi.id, NOT NULL, UNIQUE | Conversion phát sinh: Một conversion → một ledger hoa hồng |
| ma\_quy\_tac\_hoa\_hong | BIGINT | FK → quy\_tac\_hoa\_hong.id | Rule đã áp dụng |
| loai\_su\_kien | SMALLINT | NOT NULL, DEFAULT 0 | Sự kiện tạo hoa hồng<br><br>0: CLICK,<br><br>1: VIEW,<br><br>2: DANG\_KY\_MOI,<br><br>3: GIA\_HAN,<br>4: KHAC |
| loai\_tinh | SMALLINT | NOT NULL | Snapshot từ quy\_tac\_hoa\_hong.loai\_tinh |
| co\_so\_tinh | SMALLINT | NOT NULL | Snapshot từ quy\_tac\_hoa\_hong.co\_so\_tinh |
| so\_tien\_co\_so | NUMERIC(12,2) | NOT NULL | Số tiền làm cơ sở |
| gia\_tri\_tinh | NUMERIC(12,2) | NOT NULL | snapshot từ quy\_tac\_hoa\_hong.gia\_tri |
| so\_tien\_hoa\_hong | NUMERIC(12,2) | NOT NULL | Hoa hồng phải trả |
| don\_vi\_tien | VARCHAR(3) | NOT NULL | Snapshot từ quy\_tac\_hoa\_hong.don\_vi\_tien |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 0 | Trạng thái: 0: chờ duyệt, 1: đã duyệt, 2: đã chi trả, 3: từ chối |
| ky\_doi\_soat | VARCHAR(7) |  | Kỳ đối soát: YYYYMM |
| ma\_doi\_soat\_hoa\_hong | BIGINT | FK → doi\_soat\_hoa\_hong.id | Bản đối soát |
| nguoi\_duyet | BIGINT |  | Người duyệt |
| duyet\_luc | TIMESTAMPTZ |  | Thời điểm duyệt |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |

### Bảng dieu\_chinh\_hoa\_hong

Điều chỉnh tăng/giảm nghĩa vụ hoa hồng do refund, chargeback, hủy hoặc điều chỉnh thủ công. Không sửa ledger gốc.

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh điều chỉnh |
| ma\_giao\_dich\_hoa\_hong | BIGINT | FK → giao\_dich\_hoa\_hong.id, NOT NULL | Ledger bị điều chỉnh |
| ma\_chuyen\_doi | BIGINT | FK → chuyen\_doi.id | Conversion liên quan |
| loai | SMALLINT | NOT NULL | Loại điều chỉnh<br><br>0: refund, 1: chargeback,<br><br>2: hủy, 3: thủ công, 4: khác |
| so\_tien | NUMERIC(12,2) | NOT NULL | Giá trị điều chỉnh |
| don\_vi\_tien | VARCHAR(3) | NOT NULL | Đơn vị tiền |
| ly\_do | VARCHAR(500) | NOT NULL | Lý do |
| ma\_tham\_chieu | VARCHAR(255) |  | Mã tham chiếu<br><br>Refund/event/payment... |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| nguoi\_tao | BIGINT |  | Người tạo |

###

### Bảng ky\_doi\_soat

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh kỳ |
| ma\_ky | VARCHAR(7) | NOT NULL, UNIQUE | Mã kỳ: YYYYMM |
| tu\_ngay | TIMESTAMPTZ | NOT NULL | Bắt đầu kỳ |
| den\_ngay | TIMESTAMPTZ | NOT NULL | Kết thúc kỳ |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 0 | Trạng thái: 0: mở, 1: đang đối soát, 2: đã khóa, 3: đã hoàn tất |
| ngay\_khoa | TIMESTAMPTZ |  | Thời điểm khóa |
| nguoi\_khoa | BIGINT |  | Người khóa |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |

###

### Bảng doi\_soat\_hoa\_hong

Tổng hợp hoa hồng của 1 publisher trong 1 kỳ thanh toán

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh bản đối soát |
| ma\_ky\_doi\_soat | BIGINT | FK → ky\_doi\_soat.id, NOT NULL | Kỳ đối soát |
| ma\_nha\_xuat\_ban | BIGINT | FK → nha\_xuat\_ban.id, NOT NULL | Nhà xuất bản |
| tong\_gia\_tri\_giao\_dich | NUMERIC(14,2) | NOT NULL, DEFAULT 0 | Tổng giá trị conversion |
| tong\_hoa\_hong | NUMERIC(14,2) | NOT NULL, DEFAULT 0 | Tổng hoa hồng |
| tong\_dieu\_chinh | NUMERIC(14,2) | NOT NULL, DEFAULT 0 | Tổng điều chỉnh |
| so\_tien\_phai\_tra | NUMERIC(14,2) | NOT NULL, DEFAULT 0 | Số tiền cuối cùng phải trả |
| don\_vi\_tien | VARCHAR(3) | NOT NULL | Đơn vị tiền |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 0 | Trạng thái: 0: chờ đối soát, 1: đã xác nhận, 2: đã khóa, 3: đã thanh toán, 4: có vấn đề |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |

Ràng buộc đề xuất: UNIQUE(ma\_ky\_doi\_soat, ma\_nha\_xuat\_ban, don\_vi\_tien).

### Bảng chi\_tiet\_doi\_soat

là bảng liệt kê/snapshot từng khoản giao dịch được đưa vào kỳ đối soát, còn doi\_soat\_hoa\_hong là bản tổng hợp của kỳ đó

Cụ thể:

giao\_dich\_hoa\_hong

│

│ các khoản hoa hồng

▼

chi\_tiet\_doi\_soat

▲

│ các khoản điều chỉnh

│

dieu\_chinh\_hoa\_hong

Sau đó:

chi\_tiet\_doi\_soat

│

│ tổng hợp

▼

doi\_soat\_hoa\_hong

│

▼

chi\_tra

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh chi tiết đối soát |
| ma\_doi\_soat\_hoa\_hong | BIGINT | FK → doi\_soat\_hoa\_hong.id, NOT NULL | Kỳ đối soát chứa khoản này |
| ma\_giao\_dich\_hoa\_hong | BIGINT | FK → giao\_dich\_hoa\_hong.id | Giao dịch hoa hồng nguồn:<br><br>Một trong hai trường ma\_giao\_dich\_hoa\_hong hoặc ma\_dieu\_chinh\_hoa\_hong phải có giá trị |
| ma\_dieu\_chinh\_hoa\_hong | BIGINT | FK → dieu\_chinh\_hoa\_hong.id | Giao dịch điều chỉnh nguồn:<br>Một trong hai trường ma\_giao\_dich\_hoa\_hong hoặc ma\_dieu\_chinh\_hoa\_hong phải có giá trị |
| loai | SMALLINT | NOT NULL | Loại khoản đối soát<br>0: hoa hồng; 1: điều chỉnh |
| so\_tien\_hoa\_hong | NUMERIC(12,2) | NOT NULL, DEFAULT 0 | Giá trị hoa hồng đưa vào kỳ<br>Thường > 0 |
| so\_tien\_dieu\_chinh | NUMERIC(12,2) | NOT NULL, DEFAULT 0 | Giá trị điều chỉnh đưa vào kỳ<br>Có thể âm hoặc dương |
| so\_tien\_thanh\_toan | NUMERIC(12,2) | NOT NULL | Giá trị cuối cùng của dòng đối soát<br>so\_tien\_hoa\_hong + so\_tien\_dieu\_chinh |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 1 | Trạng thái dòng đối soát<br>0: loại; 1: hợp lệ |
| ly\_do | VARCHAR(500) |  | Lý do loại/điều chỉnh |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo snapshot đối soát |
| nguoi\_tao | BIGINT |  | Người tạo hoặc service tạo đối soát<br>Khóa logic → cms\_db.tai\_khoan\_he\_thong.id |

**Ràng buộc:**

CHECK (

(ma\_giao\_dich\_hoa\_hong IS NOT NULL AND ma\_dieu\_chinh\_hoa\_hong IS NULL)

OR

(ma\_giao\_dich\_hoa\_hong IS NULL AND ma\_dieu\_chinh\_hoa\_hong IS NOT NULL)

)

UNIQUE(ma\_doi\_soat\_hoa\_hong, ma\_giao\_dich\_hoa\_hong)

UNIQUE(ma\_doi\_soat\_hoa\_hong, ma\_dieu\_chinh\_hoa\_hong)

###

### Bảng chi\_tra

Tiền thực tế chi trả cho publisher

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh lần chi trả |
| ma\_doi\_soat\_hoa\_hong | BIGINT | FK → doi\_soat\_hoa\_hong.id, NOT NULL | Bản đối soát được chi |
| ma\_nha\_xuat\_ban | BIGINT | FK → nha\_xuat\_ban.id, NOT NULL | Nhà xuất bản nhận |
| ma\_phuong\_thuc | VARCHAR(50) | NOT NULL | Phương thức chi trả: bank\_transfer, ví... |
| so\_tien | NUMERIC(14,2) | NOT NULL | Số tiền chi |
| don\_vi\_tien | VARCHAR(3) | NOT NULL | Đơn vị tiền |
| ma\_giao\_dich\_chi\_tra | VARCHAR(255) |  | Mã giao dịch chi trả: Dùng cho idempotency khi cần |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 0 | Trạng thái: 0: chờ chi, 1: đang xử lý, 2: thành công, 3: thất bại |
| yeu\_cau\_luc | TIMESTAMPTZ | NOT NULL | Thời điểm yêu cầu chi |
| chi\_tra\_luc | TIMESTAMPTZ |  | Thời điểm chi thành công |
| ly\_do\_that\_bai | VARCHAR(500) |  | Lý do thất bại |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật |

## 4.4 media\_db (Media Orchestrator)

[media\_db.md](https://drive.google.com/file/d/1SJDsJMVlOv4WMiwoZbJz3jql29856S7i/view?usp=drive_link)![Word image 6](assets/image-006.png)

###

### Bảng tai\_nguyen\_media (tài nguyên media của một nội dung):

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh tài nguyên |
| ma\_noi\_dung | BIGINT | NOT NULL | Nội dung (khóa logic sang cms\_db) |
| so\_phien\_ban | INT | NOT NULL, DEFAULT 1 | Số phiên bản |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 0 | Trạng thái: 0: khởi tạo, 1: nguồn sẵn sàng, 2: đang xử lý, 3: sẵn sàng, 4: lỗi |
| ngay\_tao | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| ngay\_cap\_nhat | TIMESTAMPTZ | NOT NULL | Thời điểm cập nhật gần nhất |

### Bảng doi\_tuong\_luu\_tru (bản ghi đối tượng trên R2)

| Tên cột | Kiểu dữ liệu | Ràng buộc | Mô tả |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh |
| bucket | VARCHAR(100) | NOT NULL | Tên bucket |
| khoa\_doi\_tuong | VARCHAR(500) | NOT NULL | Khóa đối tượng: UNIQUE cùng bucket |
| kich\_thuoc | BIGINT |  | Kích thước (byte) |
| checksum | VARCHAR(128) |  | Mã băm |

### Bảng tep\_nguon (tệp video gốc trên R2):

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh tệp nguồn |
| ma\_tai\_nguyen\_media | BIGINT | FK → tai\_nguyen\_media.id, NOT NULL | Tài nguyên media |
| ma\_doi\_tuong\_luu\_tru | BIGINT | FK → doi\_tuong\_luu\_tru.id, NOT NULL | Đối tượng lưu trữ |

### Bảng dau\_ra\_media (manifest đầu ra)

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh đầu ra |
| ma\_tai\_nguyen\_media | BIGINT | FK → tai\_nguyen\_media.id, NOT NULL | Tài nguyên media |
| ma\_doi\_tuong\_luu\_tru | BIGINT | FK → doi\_tuong\_luu\_tru.id, NOT NULL | Đối tượng lưu trữ |
| loai | SMALLINT | NOT NULL | Loại manifest<br>0: hls, 1: dash |

###

### Bảng cong\_viec\_media (công việc encode/DRM tổng):

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh công việc |
| ma\_tai\_nguyen\_media | BIGINT | FK → tai\_nguyen\_media.id, NOT NULL | Tài nguyên media |
| loai | SMALLINT | NOT NULL | Loại công việc:<br><br>0: transcode, 1: drm |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 0 | Trạng thái:<br><br>0: tạo, 1: đã gửi,<br><br>2: ncc hoàn thành,<br><br>3: xử lý drm,<br><br>4: xác minh,<br><br>5: sẵn sàng,<br><br>6: thất bại |
| idempotency\_key | VARCHAR(128) | NOT NULL, UNIQUE | Khóa idempotency khi tạo job |
| tao\_luc | TIMESTAMPTZ | NOT NULL | Thời điểm tạo |
| bat\_dau\_luc | TIMESTAMPTZ |  | Thời điểm bắt đầu chạy |
| hoan\_thanh\_luc | TIMESTAMPTZ |  | Thời điểm hoàn thành |
| trace\_id | VARCHAR(128) |  | Mã truy vết - dùng chung với xuyên suốt worker/NCC/Kafka/log pipeline |
| request\_id | VARCHAR(128) |  | Mã request khởi tạo/thao tác |
| du\_kien\_thu\_lai\_luc | TIMESTAMPTZ |  | Thời điểm NULL nếu không chờ retry tiếp retry theo |
| ngay\_cap\_nhat | TIMESTAMPTZ |  | Thời điểm cập nhật gần nhất |

### Bảng lan\_thu\_cong\_viec (lịch sử lần thử, chỉ ghi thêm):

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh lần thử |
| ma\_cong\_viec\_media | BIGINT | FK → cong\_viec\_media.id, NOT NULL | Công việc |
| ma\_cong\_viec\_ncc | BIGINT | FK → cong\_viec\_ncc.id, NOT NULL | Công việc nhà cung cấp |
| lan\_thu | SMALLINT | NOT NULL | Số thứ tự lần thử, Tối đa 5 lần |
| trang\_thai | SMALLINT | NOT NULL | Kết quả lần thử, 0: đang chạy, 1: thành công, 2: thất bại |
| bat\_dau\_luc | TIMESTAMPTZ |  | Bắt đầu |
| hoan\_thanh\_luc | TIMESTAMPTZ |  | Hoàn thành |
| thong\_bao\_loi | TEXT |  | Thông báo lỗi |
| du\_kien\_thu\_lai\_luc | TIMESTAMPTZ |  | Thời điểm dự kiến retry |
| ly\_do\_thu\_lai | VARCHAR(500) |  | Lý do phát sinh retry |

### Bảng cong\_viec\_ncc (công việc khi gọi sang nhà cung cấp):

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh |
| ma\_cong\_viec\_media | BIGINT | FK → cong\_viec\_media.id, NOT NULL | Công việc tổng |
| nha\_cung\_cap | VARCHAR(50) | NOT NULL | Nhà cung cấp: BITMOVIN, EZDRM |
| ma\_ngoai | VARCHAR(255) | NOT NULL | Mã job bên nhà cung cấp: UNIQUE cùng nha\_cung\_cap |
| loai | VARCHAR(50) |  | Loại job ngoài: encoding, drm-key |
| trang\_thai | SMALLINT | NOT NULL, DEFAULT 0 | Trạng thái<br>0: gửi, 1: đang chạy,<br><br>2: hoàn thành, 3: lỗi |
| gui\_luc | TIMESTAMPTZ |  | Thời điểm gửi request |
| payload\_gui | JSONB | NOT NULL | Nội dung callback gốc |
| request\_id | VARCHAR | UUID | Định danh request đã gửi |

### Bảng phan\_hoi\_ncc (callback nhà cung cấp, chỉ ghi thêm, đảm bảo idempotency):

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh |
| nha\_cung\_cap | VARCHAR(50) | NOT NULL | Nhà cung cấp |
| ma\_su\_kien | VARCHAR(255) | NOT NULL | Mã sự kiện bên ngoài<br>UNIQUE cùng nha\_cung\_cap |
| ma\_cong\_viec\_media | BIGINT | FK → cong\_viec\_media.id | Công việc liên quan |
| noi\_dung\_goc | JSONB | NOT NULL | Nội dung callback gốc |
| nhan\_luc | TIMESTAMPTZ | NOT NULL | Thời điểm nhận |
| trace\_id | VARCHAR(100) |  | ID truy vết |
| request\_id | VARCHAR | UUID | Định danh request đã gửi từ Media |

### Bảng lich\_su\_cong\_viec\_media

cho biết toàn bộ chuỗi trạng thái đã đi qua của 1 công việc media

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh lịch sử công việc |
| ma\_cong\_viec\_media | BIGINT | FK → cong\_viec\_media.id, NOT NULL | Công việc |
| trang\_thai\_tu | SMALLINT | NOT NULL | Trạng thái trước |
| trang\_thai\_den | SMALLINT | NOT NULL | Trạng thái sau |
| su\_kien | VARCHAR(500) |  | Nguyên nhân chuyển |
| ma\_lan\_thu | BIGINT | FK → lan\_thu\_cong\_viec.id, NOT NULL | Mã lần thử công việc |
| ma\_cong\_viec\_ncc | BIGINT | FK → cong\_viec\_ncc.id | 1 trong 2 trường ma\_cong\_viec\_ncc, ma\_phan\_hoi\_ncc phải có giá trị |
| ma\_phan\_hoi\_ncc | BIGINT | FK → phan\_hoi\_ncc.id | 1 trong 2 trường ma\_cong\_viec\_ncc, ma\_phan\_hoi\_ncc phải có giá trị |
| thong\_bao | VARCHAR(500) |  | Ghi chú lỗi |
| thoi\_diem | TIMESTAMPTZ |  | Thời điểm ghi nhận |
| nguon | VARCHAR(100) |  | Nguồn tiếp nhận:<br>WORKER / CALLBACK / SCHEDULER / API |
| trace\_id | VARCHAR(100) |  | ID truy vết |
| request\_id | VARCHAR(100) |  | Request tương ứng |

### Bảng lan\_xac\_minh\_media

Lịch sử từng lần xác minh đầu ra trên R2.

| **Tên cột** | **Kiểu dữ liệu** | **Ràng buộc** | **Mô tả** |
| --- | --- | --- | --- |
| id | BIGINT | PK, IDENTITY | Định danh lần xác minh |
| ma\_cong\_viec\_media | BIGINT | FK →<br><br>cong\_viec\_media.id, NOT NULL | Công việc |
| lan\_thu | SMALLINT | NOT NULL | Số lần xác minh |
| trang\_thai | SMALLINT | NOT NULL | Kết quả 0: đang kiểm tra; 1: hợp lệ; 2: không hợp lệ |
| so\_tap\_tin\_kiem\_tra | INT |  | Số tệp đã kiểm tra |
| so\_tap\_tin\_hop\_le | INT |  | Số tệp hợp lệ |
| so\_tap\_tin\_thieu | INT |  | Số tệp còn thiếu |
| chi\_tiet\_kiem\_tra | JSONB |  | Kết quả chi tiết từng nhóm/file |
| thong\_bao\_loi | TEXT |  | Lỗi xác minh |
| bat\_dau\_luc | TIMESTAMPTZ |  | Bắt đầu kiểm tra |
| hoan\_thanh\_luc | TIMESTAMPTZ |  | Hoàn thành kiểm tra |
| trace\_id | VARCHAR(100) |  | ID truy vết |

## 4.5. Luồng dữ liệu

Phần này mô tả ba nghiệp vụ có nhiều bước ghi dữ liệu và có nguy cơ xung đột: đăng ký kèm thanh toán, cấp phép phát video, và pipeline media. Mỗi nghiệp vụ nêu các bước thay đổi dữ liệu, ranh giới transaction, và cơ chế chống xung đột.

### Đăng ký gói kèm thanh toán

Luồng này có hai giai đoạn tách rời về thời gian: người dùng chọn gói và khởi tạo thanh toán, sau đó cổng thanh toán gọi webhook xác nhận. Giữa hai giai đoạn, đăng ký ở trạng thái chờ.

Giai đoạn khởi tạo, trong một transaction: tạo bản ghi dang\_ky trạng thái CHO\_THANH\_TOAN (0), tạo bản ghi *thanh\_toan* trạng thái CHO\_XU\_LY (0) với ma\_giao\_dich\_cong do cổng cấp. Nếu người dùng đã có đăng ký DANG\_HOAT\_DONG, quy tắc **BR-03** chặn tạo đăng ký hoạt động thứ hai; đăng ký mới vẫn tạo được ở trạng thái chờ và chỉ tranh chấp khi kích hoạt.

Giai đoạn webhook xác nhận, cổng thanh toán gọi lại và có thể gọi nhiều lần cho cùng một giao dịch. Xử lý theo trình tự sau:

**BEGIN**;

\-- Khóa hàng thanh toán theo khóa idempotency của cổng (ma\_giao\_dich\_cong).

\-- Bản ghi thanh toán đã tạo ở giai đoạn khởi tạo.

**SELECT** id, ma\_dang\_ky, trang\_thai **FROM** thanh\_toan

**WHERE** ma\_giao\_dich\_cong = **:ma\_giao\_dich\_cong**

**FOR** **UPDATE**;

\-- Nếu trang\_thai đã = 1 thì COMMIT và dừng: webhook đã xử lý trước đó.

\-- Khóa hàng đăng ký để tránh hai webhook cùng kích hoạt.

**SELECT** id, trang\_thai **FROM** dang\_ky

**WHERE** id = **:ma\_dang\_ky**

**FOR** **UPDATE**;

**UPDATE** thanh\_toan

**SET** trang\_thai = 1, thoi\_diem\_thanh\_toan = **now**(), ngay\_cap\_nhat = **now**()

**WHERE** ma\_giao\_dich\_cong = **:ma\_giao\_dich\_cong** **AND** trang\_thai = 0;

**UPDATE** dang\_ky

**SET** trang\_thai = 1, ngay\_cap\_nhat = **now**()

**WHERE** id = **:ma\_dang\_ky** **AND** trang\_thai = 0;

**COMMIT**;

Idempotency ở đây dựa trên hai điểm. Cột ma\_giao\_dich\_cong có ràng buộc UNIQUE, nên một giao dịch cổng chỉ có một dòng thanh\_toan. Điều kiện AND trang\_thai = 0 trong câu UPDATE khiến lần gọi webhook thứ hai tác động **0 dòng**, không kích hoạt lại đăng ký. Chỉ mục UNIQUE một phần ux\_dang\_ky\_hoat\_dong trên dang\_ky (ma\_nguoi\_dung) WHERE trang\_thai = 1 là chốt chặn cuối. Nếu hai luồng cùng cố kích hoạt hai đăng ký của một người dùng, cơ sở dữ liệu từ chối bản ghi thứ hai bằng lỗi vi phạm UNIQUE, và ứng dụng xử lý lỗi này thành thông báo nghiệp vụ. Sau khi COMMIT, dịch vụ phát sự kiện PAYMENT\_SUCCESS và SUBSCRIPTION\_CREATED lên Kafka để Affiliate ghi nhận chuyển đổi.

Sự kiện Kafka phát sau khi COMMIT chứ không trong transaction, để tránh trường hợp gửi sự kiện nhưng transaction sau đó rollback. Mẫu transactional outbox áp dụng ở đây: ghi sự kiện vào bảng su\_kien\_cho\_gui trong cùng transaction, một tiến trình riêng đọc bảng này và đẩy lên Kafka, đánh dấu đã gửi. Cách này bảo đảm sự kiện phát ra khi và chỉ khi dữ liệu đã cam kết

### Cấp phép phát video

Khi người dùng bấm play, OTT API kiểm tra quyền và giới hạn thiết bị đồng thời trước khi cấp play token. Giới hạn thiết bị đồng thời là điểm dễ xảy ra, vượt ngưỡng khi người dùng mở nhiều thiết bị gần như cùng lúc.

Trình tự kiểm tra: xác thực người dùng, kiểm tra đăng ký hoạt động hoặc quyền xem còn hạn (**BR-08**), kiểm tra lãnh thổ (**BR-09**), rồi kiểm tra và tăng bộ đếm phiên đồng thời. Bộ đếm phiên đồng thời đặt ở Redis vì đây là dữ liệu ngắn hạn, đọc ghi tần suất cao, và cần thao tác nguyên tử.

Bộ đếm dùng khóa Redis theo mẫu phien:{ma\_nguoi\_dung}. Thao tác tăng và kiểm tra ngưỡng thực hiện bằng một script Lua chạy nguyên tử trên Redis:

\-- KEYS\[1\] = phien:{ma\_nguoi\_dung}

\-- ARGV\[1\] = so\_thiet\_bi\_dong\_thoi (ngưỡng)

\-- ARGV\[2\] = thoi\_gian\_song (giây)

**local** hien\_tai = tonumber(redis.**call**('GET', KEYS\[1\]) **or** '0')

**if** hien\_tai >= tonumber(ARGV\[1\]) **then**

**return** -1 -- vượt ngưỡng, từ chối

**end**

**local** moi = redis.**call**('INCR', KEYS\[1\])

redis.**call**('EXPIRE', KEYS\[1\], tonumber(ARGV\[2\]))

**return** moi

Script trả về -1 khi đã đạt ngưỡng, khi đó OTT API từ chối cấp phiên. Trả về số dương thì tiếp tục ghi phien\_phat và sinh play token. Khi phiên kết thúc hoặc token hết hạn, một tiến trình giảm bộ đếm bằng DECR. Thời gian sống của khóa (EXPIRE) là đảm bảo nếu client thoát đột ngột không giảm bộ đếm, khóa tự hết hạn nên bộ đếm không kẹt ở mức cao vĩnh viễn.

Redis giữ bộ đếm thay vì cột trong PostgreSQL vì hai lý do. Redis xử lý thao tác nguyên tử INCR ở độ trễ dưới miligiây. Đặt bộ đếm trong PostgreSQL buộc dùng SELECT ... FOR UPDATE trên hàng người dùng cho mỗi lần phát, gây tranh chấp khóa trên bảng nguoi\_dung. Bảng phien\_phat vẫn lưu bản ghi phiên để đối soát và thống kê, còn Redis chỉ giữ con số đếm nhanh.

### Pipeline media: transcode rồi xác minh

Pipeline media là chuỗi bước bất đồng bộ, chạy qua nhà cung cấp ngoài (Bitmovin, EZDRM) và nhận kết quả qua webhook. Hai rủi ro chính: callback trùng lặp và chuyển trạng thái đồng thời từ nhiều luồng.

Các trạng thái của cong\_viec\_media: TAO (0) sang DA\_GUI (1) khi đẩy job xuống Bitmovin, sang NCC\_HOAN\_THANH (2) khi nhận callback FINISHED, sang XU\_LY\_DRM (3) nếu có bước DRM, sang XAC\_MINH (4) khi kiểm tra đầu ra, sang SAN\_SANG (5) khi đủ manifest, hoặc THAT\_BAI (6) khi lỗi. Mỗi lần chuyển trạng thái kiểm tra trạng thái nguồn hợp lệ trước khi ghi, để một callback đến muộn không kéo trạng thái lùi.

Xử lý callback từ Bitmovin theo trình tự:

**BEGIN**;

\-- Chống trùng callback theo (nha\_cung\_cap, ma\_su\_kien).

**INSERT** **INTO** phan\_hoi\_ncc(nha\_cung\_cap, ma\_su\_kien, ma\_cong\_viec\_media, noi\_dung\_goc, nhan\_luc)

**VALUES** ('BITMOVIN', **:ma\_su\_kien**, **:ma\_cong\_viec\_media**, **:payload**, **now**())

**ON** CONFLICT (nha\_cung\_cap, ma\_su\_kien) DO NOTHING;

\-- Số dòng = 0 thì COMMIT và dừng: sự kiện đã xử lý.

\-- Khóa hàng công việc để serialize chuyển trạng thái.

**SELECT** trang\_thai **FROM** cong\_viec\_media

**WHERE** id = **:ma\_cong\_viec\_media**

**FOR** **UPDATE**;

\-- Chỉ chuyển khi trạng thái nguồn hợp lệ (đang ở DA\_GUI = 1).

**UPDATE** cong\_viec\_media

**SET** trang\_thai = 2, ngay\_cap\_nhat = **now**()

**WHERE** id = **:ma\_cong\_viec\_media** **AND** trang\_thai = 1;

**UPDATE** cong\_viec\_ncc

**SET** trang\_thai = 2, ngay\_cap\_nhat = **now**()

**WHERE** nha\_cung\_cap = 'BITMOVIN' **AND** ma\_ngoai = **:ma\_ngoai**;

**COMMIT**;

Hai lớp chống trùng hoạt động cùng nhau. Ràng buộc UNIQUE (nha\_cung\_cap, ma\_su\_kien) trên phan\_hoi\_ncc chặn xử lý cùng một sự kiện hai lần. Câu SELECT ... FOR UPDATE trên cong\_viec\_media serialize các luồng cùng thao tác trên một công việc, nên hai callback khác ma\_su\_kien nhưng tác động cùng công việc không ghi đè lẫn nhau. Điều kiện AND trang\_thai = 1 trong câu UPDATE bảo đảm chỉ chuyển từ trạng thái nguồn đúng.

Sau khi nhận FINISHED, worker chạy bước xác minh: gọi HEAD lên R2 streaming bucket kiểm tra master.m3u8, manifest.mpd và các playlist con theo độ phân giải. Đủ tệp thì chuyển cong\_viec\_media sang SAN\_SANG (5) và tai\_nguyen\_media sang SAN\_SANG (3), rồi thông báo CMS. Thiếu tệp thì ghi lan\_thu\_cong\_viec mới và lên lịch retry theo backoff 30 giây, 2 phút, 10 phút. Quá năm lần chuyển THAT\_BAI (6) và cảnh báo (BR-14).

Tác vụ đối soát định kỳ quét cong\_viec\_media ở trạng thái DA\_GUI quá lâu mà không có callback. Với mỗi công việc chưa hoàn thành, worker gọi API Bitmovin theo cong\_viec\_ncc.ma\_ngoai để lấy trạng thái thực, rồi cập nhật cho khớp. Cơ chế này xử lý trường hợp webhook thất lạc, không phụ thuộc hoàn toàn vào việc nhà cung cấp gọi lại.

### Cổng đăng nhập quản trị: phiên do OTT tự quản lý (BR-11t, BR-29)

Mục tiêu: quyền truy cập OTT Backend quản trị chấm dứt ngay khi CMS thu hồi phiên hoặc khóa tài khoản, và không phải gọi CMS ở mỗi yêu cầu.

| Bước | Hành động |
| --- | --- |
| 1 | Giao diện gọi `POST /api/v1/ott-admin/auth/login` { tenDangNhap, matKhau } |
| 2 | OTT: giới hạn đăng nhập sai theo IP và tên đăng nhập (Redis); gọi `POST {CMS}/internal/auth/verify` với `X-System-Key` và `ip` người dùng, nhận `userId` và quyền / menu hệ thống OTT |
| 3 | OTT, một transaction: ghi `adm\_phien\_quan\_tri` (active), `adm\_refresh\_token` (hash), outbox `session.started`; nạp bản sao quyền của người dùng vào Redis từ phản hồi của CMS |
| 4 | OTT trả `accessToken` (ES256 do OTT ký, `sid`), `refreshToken`, `expiresIn` |
| 5 | Mỗi yêu cầu: giải mã token offline, kiểm `sid` active (đệm ngắn), kiểm quyền từ bản sao; sai một trong ba là 401 / 403 |
| 6 | `POST /auth/refresh`: khóa dòng refresh token, đã xoay thì kết thúc phiên (`REFRESH_REUSE_DETECTED`), còn hạn thì xoay và phát `session.refreshed` |
| 7 | `POST /auth/logout`: kết thúc phiên hiện tại, phát `session.ended` (`LOGOUT`) |
| 8 | Consumer `authz-events`: `authz.session.revoked` và `authz.user.access.changed` (`active=false`) kết thúc phiên, xóa đệm `sid`, phát `session.ended` xác nhận |

Hệ quả: đăng nhập / đăng xuất ở CMS hoặc ở OTT độc lập nhau; thu hồi của quản trị viên CMS, đổi hoặc đặt lại mật khẩu, khóa và xóa tài khoản có hiệu lực ở OTT trong cỡ giây (outbox CMS + Kafka); thao tác nhạy cảm gọi thêm `POST /internal/authz/check`.

### Thu hồi phiên phát từ trang quản trị

Trong một transaction: khóa hàng *phien\_phat* (`FOR UPDATE`); `UPDATE ... SET trang\_thai = 4 WHERE id = :id AND trang\_thai = 1` (0 dòng thì trả kết quả cũ hoặc 409); `INSERT nhat\_ky\_thu\_hoi` (UNIQUE loai\_doi\_tuong, ma\_doi\_tuong); ghi outbox `playback.session.revoked`. Sau commit: Lua giảm `phien:{ma\_nguoi\_dung}` không xuống dưới 0 và ghi khóa từ chối. Thu hồi phiên đăng nhập tương tự, `WHERE family\_id = :f AND trang\_thai = 1` khi phạm vi là cả family, mỗi phiên một dòng nhật ký.

# 5\. Deployment & Scaling

## 5.1 Topology sơ đồ

-   **Edge (Cloudflare)**: CDN, Load Balancer, WAF. Domain phân tách:
    -   api.example.com → OTT API kết nối ott\_db & Redis.
    -   cms-api.example.com → CMS API kết nối cms\_db.
    -   affiliate-api.example.com → Affiliate API kết nối affiliate\_db.
    -   media-api.example.com → Media Orchestrator.
    -   media.example.com → CDN cho R2 streaming (qua WAF token).
-   **Backends**:
    -   **PostgreSQL**: 4 DB riêng. Có read-replica nếu cần. Sao lưu đều đặn.
    -   **Redis Cluster**: Redis để session/rate-limit/cache, có persistence/AOF.
    -   **Kafka**: cho sự kiện, cluster gồm vài node; topic chính: user-events, payment-events, affiliate-events, media-events, cms-events, authz-events.
    -   **Workers**: Pool của Media Orchestrator (Java/K8s), scale tách biệt các service.
-   **Monitoring**: Prometheus/Grafana, theo dõi CPU/memory, queue length, encoding job durations, CDN cache hit ratio. Alert: encode fail, webhook fail, DB full.

## 5.2 Chiến lược scale

-   **OTT API**: Scale ngang với Load Balancer. Mỗi instance kết nối Redis/Kafka, ít state.
-   **CMS API**: Quy mô nhỏ, dùng caching dữ liệu metadata.
-   **Media API**: Scale cao khi cần, elastic-worker. Chủ yếu khi có hàng loạt job encoding.
-   **R2 & CDN**: Multi-CDN (Cloudflare toàn cầu). Có thể dùng R2 Multi-Region (Cloudflare báo R2 GA multi-region).

# 6\. Vendor lock-in & chiến lược thay thế

-   **Abstraction**: Tất cả tích hợp bên thứ 3 đi qua layer adapter. Ví dụ, BitmovinProvider bao bọc API Bitmovin. Thay Bitmovin bằng mã khác chỉ cần viết adapter mới.
-   **Config-driven**: Loại provider (Bitmovin/EZDRM) trong config. Cho phép chuyển sang nhà cung cấp khác (VD Bitmovin → Zencoder, EZDRM → BUYDRM) bằng thay đổi config + triển khai adapter mới.
-   **POC Checklist**: Trước triển khai thực tế cần kiểm tra:
    -   (1) Bitmovin S3 input/output với R2: tạo input (sử dụng R2 S3 endpoint) và output (s3, region, keys).
    -   (2) Bitmovin+EZDRM: đảm bảo cài được KeyZ profile, tạo asset ID, bitmovin session nhận drm keys.
    -   (3) R2 event notification: cấu hình gửi event đến queue.
-   **Dự phòng**: Nếu Bitmovin/EZDRM ngưng, Media API tiếp tục với adapter mới. Cơ sở dữ liệu giữ logic chung (job, external\_id) nên không reset.

# 7\. Các giai đoạn triển khai

-   **Phase 0: POC**
    -   **Đặc điểm**: Không DRM (skip EZDRM). Có thể dùng Bitmovin free tier (giới hạn) hoặc tự triển khai FFmpeg để giảm chi phí.
    -   **Deliverables**: Upload video, encode HLS/DASH 1080p bằng Bitmovin, lưu R2, trả manifest. Playback ổn định.
    -   **Rủi ro**: Chưa bảo vệ bằng DRM (tạm chấp nhận nội dung không quan trọng cao). Giá encode test thấp. Không cần nhiều infra (dùng 1 DB nhỏ, 1 OTT, 1 CMS).
    -   **Infra**: 1 instances OTT/CMS/Media, PostgreSQL, Redis, Kafka (có thể tạm tự host S3 storage nếu R2 chưa dùng), Cloudflare CDN.
-   **Phase 1: Production Encoding**
    -   **Đặc điểm**: Bitmovin + Cloudflare R2 hoàn chỉnh, vẫn không bật DRM. Tích hợp với ngân hàng/payment, bắt đầu chia traffic thực.
    -   **Deliverables**: Auto-scaling, monitoring setup, backup, giảm chi phí R2 vs self-host. Thử đa region read R2 (nếu có).
    -   **Rủi ro**: Traffic tăng, có thể đột biến. Cân nhắc giới hạn băng thông (tính fair use).
    -   **Infra**: Scale OTT/API 2-3 node, CMS 1 node, Media 2 node. CDN full.
-   **Phase 2: Bật DRM (EZDRM)**
    -   **Đặc điểm**: Kích hoạt EZDRM KeyZ. Người dùng vừa xem vừa phải lấy license từ EZDRM.
    -   **Deliverables**: Mã hóa HLS/DASH, trả license URLs trong playback. Kiểm thử user/copy link.
    -   **Rủi ro**: Chi phí DRM (thông thường tính theo số lượt play). Bảo đảm devices hỗ trợ; debug license failure.
    -   **Infra**: Thêm EZDRM integration, theo dõi fail licenses, watch caching vs repeated licenses. Setup secure domain cho license server (CORS).
-   **Phase 3: Multi-region/Scale cao**
    -   **Đặc điểm**: Triển khai nhiều region Cloudflare Workers nếu hỗ trợ R2 multi-region, hoặc dùng multi-CDN. Đa DB replication.
    -   **Deliverables**: Đảm bảo latency thấp toàn cầu, deploy database geo.
    -   **Rủi ro**: Phức tạp mạng, chi phí tăng (CDN enterprise).

![Word image 7](assets/image-007.png)

# 8\. Bảo mật và vận hành

-   **Quyền truy cập Storage**: R2 buckets private. Dùng Access Key/Secret ít đặc quyền. Ví dụ: upload key chỉ PutObject vào folder original/, encode key chỉ GetObject ở original/ và PutObject ở streaming/. Trên server, mã lưu keys trong env hoặc Vault; không commit.
-   **Token/RLS**: Dùng HMAC hoặc JWT để bảo vệ CDN. Cloudflare có WAF snippets xác nhận token header trên worker.
-   **DRM flow**: EZDRM cung cấp RightsZ license server. OTT API trả đúng endpoints. Player gửi license request (kèm session token). EZDRM xác thực rồi cấp license.
-   **Xử lý lỗi (Runbook)**:
    -   *Transcode fail*: Báo log, alert, set media\_job=FAILED, kêu alert, QA tạo lại job.
    -   *Provider timeout*: Retry tự động; nếu liên tục fail, fallback sang tắt DRM (nếu Phase >=2).
    -   *Missing output*: Nếu verify không tìm đủ files, retry encode hoặc chuyển config.
    -   *Corrupt manifest*: Media API có thể lấy manifest từ Bitmovin API, parse thử. Nếu lỗi, fail job.
    -   *License error*: Ghi logs EZDRM error. Đảm bảo test Widevine/PlayReady/FairPlay devices.
    -   *Webhook retry*: Cấu hình người nhận webhook (Media API) phải trả 2xx. Dùng request idempotency (ghi event\_id) để tránh trùng xử lý.
    -   *Reconciliation*: Định kỳ quét media\_job kẹt (no callback), so với provider\_job. Nếu mismatch, gọi API Bitmovin kiểm tra/trả dữ liệu.