# Hướng dẫn sử dụng Quản lý Phim

Tài liệu dành cho người dùng cuối của hệ thống quản trị nội dung **Quản lý Phim** (giao diện web `cms_frontend`).

Tài liệu chia theo vai trò, đi từ vai trò **ít quyền nhất** đến **đầy đủ quyền**:

1. [Dùng chung cho mọi vai trò](#dung-chung)
2. [Nhà cung cấp](#vai-tro-nha-cung-cap)
3. [Biên tập viên](#vai-tro-bien-tap-vien)
4. [Kiểm duyệt viên (QC)](#vai-tro-qc)
5. [Quản trị viên](#vai-tro-quan-tri)
6. [Phụ lục: trạng thái, mã lỗi, xử lý sự cố](#phu-luc)

**Quy ước đọc:** một chức năng chỉ được mô tả một lần, ở vai trò có ít quyền nhất dùng được nó. Các vai trò có nhiều quyền hơn chỉ có đường dẫn tham chiếu ⤴ tới mục đó, và phần riêng của vai trò mình. Nếu bạn là Kiểm duyệt viên hay Quản trị viên, hãy đọc qua các phần của vai trò thấp hơn mà bạn được tham chiếu tới.

<a id="tong-quan-vai-tro"></a>

## Tổng quan các vai trò

| Vai trò | Dành cho | Thấy gì |
| --- | --- | --- |
| **Nhà cung cấp** | Studio, creator đối tác | Chỉ nội dung của chính mình. Menu **Nội dung** và **Workflow** |
| **Biên tập viên** | Nhân sự nội bộ biên tập | Nội dung do mình tạo; quản lý thể loại phim, đạo diễn, diễn viên; xem danh mục ngôn ngữ |
| **Kiểm duyệt viên (QC)** | Nhân sự duyệt và xuất bản | Mọi nội dung; duyệt, xuất bản, transcode; bản quyền, lịch phát hành, chính sách quyền xem; các màn tổng hợp |
| **Quản trị viên** | Quản trị hệ thống | Mọi thứ, gồm tài khoản, vai trò, quyền hạn, hệ thống, phiên đăng nhập |

Các menu hiện ra theo quyền của tài khoản. Nếu một menu không thấy, tài khoản của bạn không có quyền đó; hãy nhờ quản trị viên ([gán vai trò](#admin-tai-khoan)). Một tài khoản có thể giữ nhiều vai trò; khi đó bạn có quyền của tất cả vai trò đó.

Bảng tóm tắt menu theo vai trò (✓ có, "xem" chỉ xem không sửa):

| Menu | Nhà cung cấp | Biên tập viên | QC | Quản trị viên |
| --- | :-: | :-: | :-: | :-: |
| Trang chủ | ✓ | ✓ | ✓ | ✓ |
| Nội dung | ✓ (của mình) | ✓ (của mình) | ✓ (tất cả) | ✓ (tất cả) |
| Workflow (hàng đợi) | ✓ | | ✓ | ✓ |
| Quy trình hiện tại, Lịch sử quy trình | | | ✓ | ✓ |
| Phiên bản nội dung | | | ✓ | ✓ |
| Thể loại phim, đạo diễn, diễn viên | | ✓ (sửa) | xem | ✓ |
| Danh mục ngôn ngữ | | xem | xem | ✓ |
| Lịch phát hành, Bản quyền nội dung (xem chéo) | | | ✓ | ✓ |
| Cấp quyền xem | | | xem | ✓ |
| Chính sách quyền xem | | | ✓ | ✓ |
| Nhà cung cấp (danh sách) | | | | ✓ |
| Tài khoản, Vai trò, Quyền hạn & Menu, Hệ thống, Phiên đăng nhập | | | | ✓ |

---

<a id="dung-chung"></a>

# 1. Dùng chung cho mọi vai trò

<a id="dang-nhap"></a>

## 1.1. Đăng nhập, đăng xuất, hết phiên

1. Mở địa chỉ của CMS, nhập **Tên đăng nhập** và **Mật khẩu**, bấm **Đăng nhập**. Đăng nhập bằng tên đăng nhập, không phải email.
2. Sai thông tin, hệ thống chỉ báo chung "Tên đăng nhập hoặc mật khẩu không đúng" (không cho biết tên có tồn tại hay không). Nhập sai nhiều lần liên tiếp sẽ bị chặn tạm thời: chờ ít phút rồi thử lại.
3. Đăng xuất: bấm **Đăng xuất** ở góc phải thanh trên cùng.
4. Tải lại trang (F5) hoặc mở thêm tab mới của cùng trình duyệt thì vẫn đang đăng nhập, không phải đăng nhập lại. Đăng xuất (hoặc phiên hết hạn) ở một tab thì các tab khác cũng thoát theo. Đóng hẳn trình duyệt rồi mở lại vẫn giữ phiên cho tới khi phiên hết hạn hoặc bạn bấm **Đăng xuất**; trên máy dùng chung, hãy luôn bấm **Đăng xuất** khi dùng xong.
5. Khi phiên hết hạn hoặc bị thu hồi (khóa tài khoản, đổi mật khẩu, quản trị viên thu hồi), hệ thống đưa về trang đăng nhập kèm thông báo.

<a id="doi-mat-khau"></a>

## 1.2. Đổi mật khẩu

1. Bấm **Đổi mật khẩu** ở thanh trên cùng.
2. Nhập **Mật khẩu hiện tại**, **Mật khẩu mới** (hai lần).
3. Bấm **Đổi mật khẩu**. Nhập sai mật khẩu hiện tại sẽ báo lỗi ngay tại form và bạn vẫn đang đăng nhập; nhập lại không khớp bị chặn trước khi gửi.
4. Thành công thì **mọi phiên đăng nhập bị thu hồi**, bạn được đưa về trang đăng nhập để đăng nhập lại bằng mật khẩu mới.

<a id="giao-dien"></a>

## 1.3. Bố cục giao diện

- **Thanh bên trái:** logo, **Trang chủ** và các nhóm menu (Nội dung, Tài khoản hệ thống, Danh mục hệ thống, Workflow, Phát hành & bản quyền). Bấm tên nhóm để thu gọn hoặc mở rộng. Mục đang mở có vạch tím bên trái.
- **Thanh trên cùng:** vết điều hướng (ví dụ *CMS / Nội dung / Nội dung / #14*, bấm các phần gạch chân để quay lại), tên bạn và vai trò, **Đổi mật khẩu**, **Đăng xuất**.
- **Màn hình hẹp (điện thoại, máy tính bảng):** thanh bên được thu lại, bấm nút ☰ để mở; chọn mục là tự đóng.
- Bàn phím: phím Tab đầu tiên đưa tới liên kết **Bỏ qua điều hướng**; Esc đóng hộp thoại.
- **Giờ hiển thị luôn theo GMT+7** (giờ Việt Nam), không phụ thuộc múi giờ máy tính.
- Gõ địa chỉ không có thật, hoặc địa chỉ của chức năng mà vai trò của bạn không được cấp (ví dụ gõ thẳng `/accounts` khi không phải quản trị viên), sẽ ra trang **Không tìm thấy trang** với hai nút **Về trang chủ** và **Quay lại**. Chỉ các chức năng có trong thanh bên của bạn mới mở được.
- **Tiêu đề tab trình duyệt** cho biết màn hình đang mở (ví dụ *Nội dung · #14*, *Thể loại phim*), giúp phân biệt khi mở nhiều tab.

<a id="danh-sach"></a>

## 1.4. Làm việc với danh sách

Hầu hết màn hình quản lý có cùng một kiểu:

- **Tìm kiếm:** gõ vào ô tìm rồi bấm **Tìm** (hoặc Enter).
- **Bộ lọc:** các ô chọn kiểu "Trạng thái: tất cả"; đổi giá trị là danh sách lọc lại.
- **Sắp xếp:** bấm tiêu đề cột có mũi tên; bấm lần nữa để đảo chiều, lần thứ ba để bỏ sắp xếp.
- **Phân trang:** dưới bảng có số bản ghi, ô chọn số dòng mỗi trang (10, 20, 50, 100) và nút **Trước/Sau**.
- **Mở một bản ghi:** bấm dòng đó hoặc nút **Mở** để sang trang chi tiết riêng (địa chỉ dạng `/ten-man-hinh/{id}`). Nút **← Tên danh sách** quay lại.
- **Thêm mới:** nút **+ Thêm mới** (hoặc tên riêng, ví dụ **+ Thêm nội dung**) ở góc phải đầu trang; điền form rồi **Tạo mới**. Chỉ hiện khi bạn có quyền tạo.
- **Xóa:** nút **Xóa** ở cuối dòng hoặc ở đầu trang chi tiết; có hộp xác nhận. Nếu bản ghi còn dữ liệu tham chiếu, hệ thống từ chối và liệt kê.
- **Chỉ đọc:** nếu bạn không có quyền sửa, form hiện ở dạng không sửa được và không có nút Lưu.

**Hai người cùng sửa một bản ghi:** nếu có người lưu trước, lần Lưu của bạn báo "Dữ liệu đã bị người khác thay đổi. Hãy tải lại rồi thao tác lại" và form tự nạp lại bản mới. Kiểm tra rồi nhập lại thay đổi của bạn.

<a id="o-chon"></a>

## 1.5. Các ô nhập đặc biệt trong form

- **Ô chọn có tìm kiếm** (viền chàm, biểu tượng ⛓): bấm vào ô, gõ vài chữ (không cần đúng dấu), dùng ↑ ↓ rồi Enter hoặc bấm dòng cần chọn. Chọn "— Không chọn —" để bỏ chọn. Nút **↗** cạnh ô mở bản ghi đang chọn.
- **Ô chọn nhiều** (đạo diễn, diễn viên, danh mục, vai trò, quyền): gõ để tìm, bấm kết quả để thêm; mỗi mục đã chọn là một thẻ có nút **×** để bỏ.
- **Ô thẻ** (lãnh thổ, thể loại): gõ giá trị rồi nhấn **Enter** hoặc dấu phẩy để thành thẻ; **Backspace** khi ô trống để xóa thẻ cuối. Mã quốc gia tự viết hoa.
- **Công tắc Có/Không:** bấm hoặc nhấn phím cách.
- **Ngày giờ:** chọn theo giờ GMT+7.
- Trường bắt buộc có dấu **\***. Sai định dạng thì báo ngay dưới ô và không gửi form.

<a id="o-tai-tep"></a>

## 1.6. Tải tệp lên và bảo vệ khi đang tải

Tệp (video, phụ đề, ảnh bìa) được tải **thẳng lên kho lưu trữ** từ trình duyệt của bạn, không qua máy chủ CMS.

Trong lúc còn tệp đang tải, giao diện **khóa mọi thao tác khác** để tránh mất tệp:

- Thanh xanh đậm trên cùng hiện *"Đang tải tệp (n), vui lòng chờ. Không đóng hoặc chuyển trang"* kèm nút **Hủy tải**.
- Bấm link, nút, tab hoặc gửi form khác đều bị chặn kèm thông báo.
- Đóng tab, tải lại trang hoặc rời sang trang khác: trình duyệt cảnh báo; chuyển trang trong CMS thì hỏi *"Rời trang và hủy tải?"*. Nếu đồng ý rời đi, các lượt tải chưa xong bị hủy và phải tải lại từ đầu. Chọn ở lại thì không có gì bị hủy.
- Lượt tải đã hoàn tất không bị hủy khi bạn rời trang.
- Nếu đóng tab đột ngột mà yêu cầu hủy không kịp gửi, hệ thống tự dọn lượt tải dở sau một thời gian.

<a id="thong-bao-loi"></a>

## 1.7. Hiểu thông báo của hệ thống

- Thông báo xanh ở góc dưới phải (ví dụ "Đã lưu thay đổi.") tự mất sau vài giây; thông báo đỏ lưu lâu hơn.
- "**Bạn không có quyền thực hiện thao tác này**": tài khoản không đủ quyền.
- "**Đối tượng không còn tồn tại hoặc không thuộc về bạn**": nội dung đã bị xóa, hoặc thuộc nhà cung cấp khác.
- "**Thao tác tương tự đang được xử lý**": bạn bấm hai lần; chờ vài giây rồi kiểm tra lại kết quả, đừng bấm lại ngay.
- Lỗi hệ thống có **mã theo dõi** trong ngoặc; khi báo lỗi cho quản trị, hãy gửi kèm mã này.

---

<a id="vai-tro-nha-cung-cap"></a>

# 2. Vai trò: Nhà cung cấp

Tài khoản nhà cung cấp gắn với một đơn vị cung cấp nội dung. Bạn **chỉ thấy và chỉ thao tác trên nội dung của đơn vị mình**, chỉ có hai menu: **Nội dung** và **Workflow**. Bạn tạo nội dung, tải video, phụ đề, ảnh bìa rồi **gửi cho kiểm duyệt (QC)**. Bạn không được xóa nội dung, không duyệt, không xuất bản.

Đọc trước [Mục 1](#dung-chung) về đăng nhập, danh sách, ô nhập và tải tệp.

<a id="trang-chu"></a>

## 2.1. Trang chủ: việc cần xử lý

Trang chủ hiện các khối, mỗi khối có tổng số, 5 mục mới nhất và liên kết **Xem tất cả**:

| Khối | Ý nghĩa |
| --- | --- |
| **Chờ kiểm duyệt (QC)** | Nội dung đã gửi QC, đang chờ Duyệt hoặc Trả lại |
| **Bản nháp** | Đang biên tập hoặc vừa bị trả lại; hoàn thiện rồi Gửi QC |
| **Sẵn sàng xuất bản** | QC đã duyệt, chờ xuất bản (chỉ người có quyền xuất bản thấy) |
| **Media lỗi** | Có phiên bản xử lý media bị lỗi; hiện kèm lý do |

Bấm một mục để mở nội dung đó. Trang chủ chỉ tải một lần khi mở; muốn cập nhật hãy tải lại trang.

<a id="danh-sach-noi-dung"></a>

## 2.2. Danh sách nội dung

Menu **Nội dung**. Các cột: tiêu đề, loại (Phim lẻ, Series, Tập phim), nhà cung cấp, trạng thái, năm phát hành. Bộ lọc: **Loại**, **Trạng thái**, **Media** (lọc theo trạng thái xử lý media, ví dụ "Lỗi media"), và **Nhà cung cấp** (chỉ nhân sự nội bộ có).

Trạng thái của nội dung: *Bản nháp → Chờ QC → Sẵn sàng → Đã xuất bản → Gỡ xuất bản* (xem [Phụ lục A](#phu-luc-trang-thai)).

<a id="tao-noi-dung"></a>

## 2.3. Tạo nội dung mới

1. Ở **Nội dung** bấm **+ Thêm nội dung**.
2. Điền **form chính**:
   - **Tiêu đề mặc định** (bắt buộc, tối đa 500 ký tự): dùng khi chưa có bản dịch cho ngôn ngữ người xem.
   - **Loại nội dung**: *Phim lẻ*, *Series* (nhiều tập) hoặc *Tập phim* (một tập của series). **Không đổi được sau khi tạo.**
   - **Nhà cung cấp**: tài khoản nhà cung cấp tự gán đơn vị của mình (không phải chọn). Tập phim lấy nhà cung cấp theo series.
   - **Năm phát hành** (1888 đến 2100).
   - **Số tập:** Phim lẻ không có trường này (chỉ một tập). Series có **Tổng số tập**. Tập phim có **Thứ tự tập** (nên nhập để tránh trùng thứ tự trong mùa).
   - Với **Tập phim**: chọn **Series chứa tập** (chỉ chọn được nội dung loại Series) rồi tùy chọn **Mùa phim** của series đó.
   - **Đạo diễn**, **Diễn viên** (ô chọn nhiều; với diễn viên nhập thêm **Vai diễn**), **Mô tả mặc định**.
3. Bấm **Tạo mới**. Hệ thống mở trang chi tiết của nội dung vừa tạo ở trạng thái **Bản nháp**; các tab chỉ hiện sau khi tạo.

<a id="trang-chi-tiet-noi-dung"></a>

## 2.4. Trang chi tiết nội dung

Từ trên xuống:

1. **Đầu trang:** tên nội dung, nút **← Danh sách nội dung**, và **Xóa nội dung** (nhà cung cấp không có nút này).
2. **Form chính** (bên trái) với hai ô chỉ đọc: **Trạng thái** (chỉ đổi bằng khung bên phải) và **Trace ID** (mã theo dõi khi gửi sang hệ thống xử lý media). Lưu bằng **Lưu thông tin chung**.
3. **Khung "Quy trình duyệt"** (bên phải): thanh các bước, người phụ trách, danh sách điều kiện và các nút chuyển bước. Xem [2.10](#gui-qc).
4. **Khung "Media & tệp trên kho lưu trữ"**: tình trạng video. Xem [2.8](#khung-media).
5. **Các tab dữ liệu liên quan**, mỗi tab lưu độc lập: Mùa phim (chỉ Series), Thông tin nội dung, Nội dung đa ngôn ngữ, Thể loại phim, Phiên bản nội dung (không có ở Series), Phụ đề, Ảnh bìa, Bản quyền nội dung, Quy trình, Lịch phát hành. Số trên mỗi tab là số bản ghi.

**Khi nào sửa được:** tài khoản nhà cung cấp **chỉ sửa được khi nội dung ở trạng thái Bản nháp**. Sau khi Gửi QC, mọi form chuyển sang chỉ đọc kèm ghi chú *"Nội dung đang ở bước … Nhà cung cấp chỉ được xem"*. Nội dung bị QC **Trả lại** quay về Bản nháp và sửa lại được.

**Cách các tab lưu:** mỗi tab dạng thẻ (mỗi bản ghi một thẻ) có nút **Lưu** và **Xóa** riêng; lưu hoặc xóa thẻ này không ảnh hưởng thẻ khác. Bấm **+ Thêm …** ở đầu tab để thêm thẻ mới, thẻ mới có nhãn "chưa lưu" cho tới khi bấm Lưu.

<a id="tab-thong-tin"></a>

### Tab Thông tin nội dung

Một thẻ **Thông tin chi tiết**: quốc gia sản xuất, **ngôn ngữ gốc** (bắt buộc), thời lượng (phút), độ tuổi tối thiểu, xếp hạng (ví dụ T13, T16, T18) và **thể loại (tag)** (tối đa 20 thẻ, mỗi thẻ tối đa 50 ký tự). Bấm **Lưu**.

<a id="tab-da-ngon-ngu"></a>

### Tab Nội dung đa ngôn ngữ

Mỗi ngôn ngữ một thẻ: **Ngôn ngữ**, **Tiêu đề**, **Mô tả ngắn**, **Mô tả chi tiết**. Một ngôn ngữ chỉ có một thẻ. Điều kiện để Gửi QC thường yêu cầu có bản dịch, và có **bản dịch tiếng Việt**.

<a id="tab-danh-muc"></a>

### Tab Thể loại phim

Gán nội dung vào các thể loại (ví dụ Phim › Hàn Quốc) theo từng ngôn ngữ hiển thị: bấm **+ Thêm thể loại**, chọn **ngôn ngữ trước**, rồi chọn nhiều thể loại (tên thể loại hiển thị theo ngôn ngữ đó; danh mục chưa có bản dịch được đánh dấu "chưa dịch"). Mỗi ngôn ngữ một thẻ, bấm **Lưu**. Danh mục là danh mục dùng chung do [Biên tập viên](#bien-tap-danh-muc) quản lý.

<a id="tab-mua-phim"></a>

### Tab Mùa phim (chỉ Series)

Mỗi mùa một thẻ: **Số mùa**, tiêu đề mùa, năm phát hành, mô tả. Tập phim sau đó chọn mùa của series.

<a id="tab-phien-ban"></a>

### Tab Phiên bản nội dung và tải video gốc

Dành cho Phim lẻ và Tập phim (Series không có media riêng; media nằm ở từng tập).

1. Bấm **+ Thêm phiên bản**. Số phiên bản (v1, v2…) do hệ thống cấp. Mỗi lần mã hóa lại sinh một phiên bản mới.
2. Ở thẻ phiên bản, mục **Video gốc**: chọn tệp rồi bấm **Tải video lên**. Định dạng nhận: mp4, mkv, mov, mxf, ts, m4v, webm, avi.
   - **Video phải có luồng âm thanh.** Video không có âm thanh sẽ bị bước mã hóa từ chối ([Phụ lục B](#phu-luc-loi-media)).
   - Video **lớn hơn 100 MiB** được chia nhiều phần: mất mạng chỉ phải tải lại phần lỗi. Nếu bị ngắt, thẻ hiện *"Có lượt tải dở dang cho «tên tệp»"*: **chọn lại đúng tệp đó** (cùng tên, kích thước, ngày sửa) để tải tiếp phần còn thiếu.
   - Có thanh phần trăm và nút **Hủy tải**. Xem thêm [1.6](#o-tai-tep).
3. Thay video: chọn tệp mới và tải lại (tệp cũ trên kho bị xóa).
4. Thẻ hiện nhãn trạng thái media (*Chưa có nguồn, Nguồn sẵn sàng, Đang xử lý, Sẵn sàng, Lỗi media*) và, nếu lỗi, ô lỗi kèm gợi ý ([Phụ lục B](#phu-luc-loi-media)).

Nhà cung cấp **không** sửa ghi chú, xóa phiên bản hay chọn phiên bản đang phục vụ, và không bấm Transcode: các việc đó thuộc [Kiểm duyệt viên](#qc-phien-ban).

<a id="tab-phu-de"></a>

### Tab Phụ đề

**+ Thêm phụ đề**: chọn **Ngôn ngữ phụ đề** trước, chọn **Loại phụ đề** (Thường, Bắt buộc/forced, SDH cho người khiếm thính), chọn tệp (srt, vtt, ttml, xml, dfxp; tối đa 2 MiB) rồi **Lưu**. Khi sửa, chọn tệp mới để thay tệp cũ; ngôn ngữ và loại không đổi.

<a id="tab-anh-bia"></a>

### Tab Ảnh bìa

**+ Thêm ảnh bìa**: chọn **Loại ảnh** (Poster, Thumbnail, Backdrop), **Ngôn ngữ** (để trống nếu ảnh dùng chung mọi ngôn ngữ), tệp ảnh (jpg, png, webp; tối đa 10 MiB) rồi **Lưu**. Kích thước ảnh được đọc tự động. Ảnh đầu tiên của mỗi loại và ngôn ngữ tự thành **ảnh mặc định**; công tắc **Ảnh mặc định** để đổi. Loại, ngôn ngữ và tệp không đổi được: muốn đổi thì xóa rồi thêm lại. Điều kiện Gửi QC thường yêu cầu có **poster mặc định**.

<a id="tab-ban-quyen-lich"></a>

### Tab Bản quyền nội dung và Lịch phát hành (chỉ xem)

Nhà cung cấp chỉ **xem** hai tab này; người sửa là [Kiểm duyệt viên](#qc-ban-quyen-lich).

<a id="khung-media"></a>

## 2.5. Khung Media & tệp trên kho lưu trữ

Khung này cho biết video của nội dung đã sẵn sàng chưa:

- **Kết luận** ở đầu: *Media đầy đủ* (xanh), *Media chưa đầy đủ* kèm lý do còn thiếu (vàng), hoặc *Series không có media riêng* (xanh).
- **Từng phiên bản:** nhãn trạng thái, nguồn (đã gắn và có trên kho hay không), manifest HLS/DASH, số tệp đầu ra, lỗi nếu có.
- **Xem tệp trên kho:** liệt kê tệp của nội dung trên kho lưu trữ kèm trạng thái.
- **Làm mới:** kết quả xử lý media gửi về CMS qua hàng đợi nên **trang không tự cập nhật**. Sau khi tải video hoặc khi QC bấm Transcode, hãy bấm **Làm mới** (hoặc tải lại trang) để thấy trạng thái mới. Thường vài giây sau khi tải xong, video gốc chuyển từ *Chưa có nguồn* sang *Nguồn sẵn sàng*.

<a id="gui-qc"></a>

## 2.6. Gửi QC (gửi kiểm duyệt)

Khi nội dung đã đủ điều kiện, bấm nút **Gửi QC** ở khung **Quy trình duyệt** (bên phải), ghi chú nếu cần rồi **Xác nhận**. Nội dung chuyển sang **Chờ QC**, bạn không sửa được nữa.

- Danh sách điều kiện hiện ngay trong khung: ✓ đã đạt, ✕ chưa đạt kèm cách khắc phục (ví dụ "Cần tải lên tệp video nguồn", "Cần ít nhất một bản dịch"). Nút **Gửi QC** bị tắt cho tới khi đạt; rê chuột vào nút để xem lý do.
- Phim lẻ và tập phim cần video gốc ở trạng thái sẵn sàng; Series không cần media.
- Sau khi bạn sửa dữ liệu ở các tab, khung này tự cập nhật điều kiện. Nếu vừa tải video, bấm **Làm mới** ở khung Media.

<a id="theo-doi-workflow"></a>

## 2.7. Theo dõi tiến trình và nội dung bị trả lại

- Menu **Workflow** là hàng đợi nội dung theo trạng thái (mặc định *Chờ QC*; đổi bộ lọc **Trạng thái** để xem Bản nháp, Sẵn sàng…). Bấm **Xử lý** để mở nội dung.
- Tab **Quy trình** của nội dung cho thấy **bước hiện tại**, người phụ trách và **lịch sử quy trình** (từ bước, đến bước, người thực hiện, thời điểm, ghi chú). Lịch sử chỉ ghi thêm, không sửa hay xóa.
- Nếu QC **Trả lại**, nội dung về **Bản nháp** kèm ghi chú lý do trong lịch sử. Đọc lý do ở tab Quy trình, sửa rồi Gửi QC lại. Lưu ý: khi nội dung bị trả lại và bạn tải tệp mới, **các tệp cũ trên kho bị xóa** (video gốc, phụ đề, ảnh bìa bị thay).

---

<a id="vai-tro-bien-tap-vien"></a>

# 3. Vai trò: Biên tập viên

Biên tập viên là nhân sự nội bộ. Có **mọi chức năng của Nhà cung cấp** ([Mục 2](#vai-tro-nha-cung-cap): tạo nội dung, các tab, tải tệp, Gửi QC) áp dụng cho **nội dung do mình tạo**, và thêm các chức năng dưới đây. Biên tập viên **không có** menu Workflow, không duyệt, không xuất bản.

Khác biệt khi tạo nội dung: tài khoản nội bộ phải **chọn Nhà cung cấp** (ô chọn có tìm kiếm) vì không gắn sẵn với một đơn vị. Series và Tập phim: tập lấy nhà cung cấp theo series. Danh sách nội dung có thêm bộ lọc **Nhà cung cấp**.

Biên tập viên nội bộ chưa bị khóa sửa ngoài trạng thái Bản nháp (máy chủ vẫn kiểm tra lại mọi thao tác và trả lỗi nếu không hợp lệ).

<a id="xoa-noi-dung"></a>

## 3.1. Xóa nội dung

1. Mở nội dung, bấm **Xóa nội dung** ở đầu trang.
2. Hộp xác nhận nêu rõ: xóa **kèm các tệp trên kho lưu trữ và tài nguyên media** của nội dung (chạy nền, mất khoảng một phút) và **không hoàn tác được**.
3. Bấm **Xóa vĩnh viễn**. Hệ thống quay về danh sách và báo "Đã xóa nội dung".

**Nội dung đã từng xuất bản không xóa được:** nút bị tắt kèm giải thích; chỉ có thể **Gỡ xuất bản** ở khung Quy trình duyệt (xem [QC](#qc-xuat-ban)). Nếu cố xóa, hệ thống trả lỗi và gợi ý Gỡ xuất bản; nội dung còn nguyên. Xóa nội dung không xóa được nếu còn dữ liệu phụ thuộc: thông báo sẽ liệt kê.

<a id="bien-tap-danh-muc"></a>

## 3.2. Thể loại phim

Menu **Danh mục hệ thống → Thể loại phim**: cây phân loại chính thức, ví dụ *Phim › Hàn Quốc*.

- Danh sách có bộ lọc **Cấp** (chỉ thể loại gốc) và nút chuyển **Danh sách / Dạng cây**. Ở dạng cây, chọn **Ngôn ngữ hiển thị** để xem tên theo ngôn ngữ; thể loại chưa dịch có nhãn "Chưa dịch".
- **Thêm:** **+ Thêm mới** → **Tên mặc định** (bắt buộc), **Thể loại cha** (để trống nếu là gốc), **Mô tả** → **Tạo mới**.
- Trang chi tiết có hai tab:
  - **Bản dịch thể loại:** **+ Thêm bản dịch**, chọn **Ngôn ngữ**, **Tên theo ngôn ngữ**, **Mô tả**, **Lưu bản dịch**. Sửa hoặc xóa từng dòng bằng **Sửa/Xóa**.
  - **Thể loại con:** danh sách con trực tiếp (bấm để mở). Muốn thêm thể loại con, tạo thể loại mới và chọn thể loại này làm cha.

<a id="bien-tap-nguoi"></a>

## 3.3. Danh mục đạo diễn và diễn viên

Menu **Danh mục đạo diễn** và **Danh mục diễn viên**: hồ sơ dùng chung cho nhiều nội dung.

- Trường: **Họ tên** (bắt buộc), **Tên khác**, **Quốc tịch**, **Ngày sinh**, **Giới thiệu**, **Khóa tệp ảnh đại diện** (chỉ gồm chữ, số, `/ _ . -`).
- Thêm, sửa, xóa như [1.4](#danh-sach). Xóa bị từ chối nếu người này còn gắn với nội dung.
- Ở trang chi tiết, nút **Phim liên quan** mở danh sách phim (và vai diễn với diễn viên) mà người này tham gia.

<a id="bien-tap-ngon-ngu"></a>

## 3.4. Danh mục ngôn ngữ (chỉ xem)

Menu **Danh mục ngôn ngữ** cho biết các ngôn ngữ dùng cho bản dịch, phụ đề và ảnh. Biên tập viên chỉ **xem**; người thêm, sửa là [Quản trị viên](#admin-danh-muc).

---

<a id="vai-tro-qc"></a>

# 4. Vai trò: Kiểm duyệt viên (QC)

QC thấy **mọi nội dung** (không chỉ của một đơn vị) và là người **duyệt, trả lại, xuất bản, gỡ xuất bản, kích hoạt lại**, bấm **Transcode**, quản lý **bản quyền, lịch phát hành, chính sách quyền xem**. QC dùng lại các màn hình xem nội dung ở [Mục 2](#vai-tro-nha-cung-cap) (danh sách, trang chi tiết, khung Media, tab) và các danh mục ở [Mục 3](#vai-tro-bien-tap-vien) ở chế độ **chỉ xem**.

**Giới hạn sửa nội dung của QC:**

- Nội dung do **tài khoản nội bộ** tạo: QC sửa được các tab thường (thông tin, đa ngôn ngữ, danh mục, mùa, phụ đề, ảnh bìa).
- Nội dung do **đối tác (nhà cung cấp)** tạo: QC **chỉ xem, duyệt hoặc trả lại**. Các tab thường bị khóa, kèm ghi chú *"Nội dung do đối tác tạo: bạn chỉ xem, duyệt hoặc trả lại"*.
- QC không tạo nội dung mới và không xóa nội dung.

<a id="qc-hang-doi"></a>

## 4.1. Hàng đợi và các màn hình tổng hợp

- **Trang chủ** ([2.1](#trang-chu)) hiện thêm khối **Sẵn sàng xuất bản**.
- **Workflow** (hàng đợi, [2.7](#theo-doi-workflow)): mặc định hiện *Chờ QC*, là việc chính của QC mỗi ngày. Bấm **Xử lý** để mở nội dung.
- **Quy trình hiện tại** (nhóm Workflow): bước hiện tại của mọi nội dung. Cột: nội dung, bước (Bản nháp, QC, Duyệt, Xuất bản, Gỡ xuất bản), trạng thái bước (Đang xử lý, Hoàn tất, Trả lại), người phụ trách, cập nhật. Lọc theo **Bước** và **Trạng thái bước**. Chỉ đọc; nút **Mở nội dung** để xử lý.
- **Lịch sử quy trình** (nhóm Workflow): mọi lần chuyển bước theo thời gian (mới nhất trước): nội dung, từ bước, đến bước, người thực hiện (họ tên), ghi chú. Chỉ đọc.
- **Phiên bản nội dung** (nhóm Danh mục hệ thống): phiên bản media của mọi phim: nội dung, số phiên bản, đang phục vụ, trạng thái media (lọc được), tài nguyên media, lỗi nếu có. Chỉ đọc; **Mở nội dung** để thao tác.

<a id="qc-duyet"></a>

## 4.2. Duyệt hoặc Trả lại nội dung

Mở nội dung đang **Chờ QC**; ở khung **Quy trình duyệt** có hai nút:

- **Duyệt:** chuyển sang **Sẵn sàng** (QC bấm tay, không tự duyệt). Nút bị tắt cho tới khi mọi điều kiện đạt; điều kiện chính: media đã xử lý xong (phim lẻ, tập), có thông tin và bản dịch, có poster mặc định. Xem hướng dẫn trong khung và [4.5](#qc-transcode).
- **Trả lại:** chỉ dùng khi nội dung đang **Chờ QC**. **Bắt buộc nhập lý do** ở ô Ghi chú; hộp thoại nhắc rằng nhà cung cấp sẽ tải lại và các tệp cũ sẽ bị xóa. Nội dung về **Bản nháp**.

Mỗi thao tác được ghi vào **lịch sử quy trình** cùng người thực hiện và ghi chú, không sửa được.

<a id="qc-xuat-ban"></a>

## 4.3. Xuất bản, Gỡ xuất bản, Kích hoạt lại

Cũng ở khung Quy trình duyệt, theo trạng thái:

| Trạng thái | Nút | Kết quả |
| --- | --- | --- |
| Sẵn sàng | **Xuất bản** | Đã xuất bản (hiển thị với người xem theo lịch phát hành và bản quyền) |
| Đã xuất bản | **Gỡ xuất bản** | Gỡ xuất bản (ẩn với người xem) |
| Gỡ xuất bản | **Kích hoạt lại (xuất bản)** | Đã xuất bản trở lại |

Điều kiện **Xuất bản** (khung liệt kê ✓/✕, nút bị tắt nếu thiếu): có chính sách quyền xem cho cấp **FREE** ([4.7](#qc-chinh-sach)), có **lịch phát hành** và **bản quyền còn hiệu lực có lãnh thổ** ([4.6](#qc-ban-quyen-lich)), có poster, bản dịch tiếng Việt, media đầy đủ. **Tập phim** chỉ xuất bản được khi **series của nó đã xuất bản**. Nội dung đã từng xuất bản không xóa được ([3.1](#xoa-noi-dung)).

<a id="qc-phien-ban"></a>

## 4.4. Quản lý phiên bản media

Ở tab **Phiên bản nội dung** (ngoài việc thêm phiên bản và tải video như [2.4](#tab-phien-ban)), QC còn:

- Sửa **Ghi chú** của phiên bản (**Lưu ghi chú**) và **Xóa** phiên bản (phiên bản đang phục vụ hoặc đã có media sẵn sàng bị từ chối xóa).
- Chọn **Phiên bản media đang phục vụ** ở ô trên đầu tab rồi **Lưu phiên bản đang phục vụ**. Chỉ chọn được phiên bản có **media đã sẵn sàng**; nếu chưa, hệ thống báo "Phiên bản chưa có media sẵn sàng". Thứ tự đúng: Transcode xong → chọn phiên bản phục vụ → Duyệt.

<a id="qc-transcode"></a>

## 4.5. Transcode (mã hóa video)

Ở khung **Media & tệp trên kho lưu trữ**, mỗi phiên bản có nút **Transcode** (hoặc **Transcode lại** khi đang lỗi).

1. Điều kiện: nội dung đang **Chờ QC**, phiên bản đã có video gốc ở trạng thái *Nguồn sẵn sàng*. Nếu nút bị tắt, rê chuột để xem lý do (ví dụ "Chỉ transcode khi nội dung đang chờ QC", "Chờ Media xác nhận video gốc sẵn sàng", "Đang được xử lý media").
2. Bấm **Transcode**. Hệ thống báo "Đã gửi yêu cầu transcode" và phiên bản chuyển *Đang xử lý*.
3. Kết quả về CMS qua hàng đợi, **trang không tự cập nhật**: thỉnh thoảng bấm **Làm mới**. Video ngắn xong sau vài phút; video dài lâu hơn nhiều.
4. Xong thì phiên bản *Sẵn sàng* kèm HLS/DASH và chất lượng. Sau đó chọn phiên bản phục vụ ([4.4](#qc-phien-ban)) và **Duyệt**.
5. Nếu lỗi, xem [Phụ lục B](#phu-luc-loi-media). Việc mã hóa gọi dịch vụ bên ngoài nên có chi phí; tránh bấm lặp lại khi chưa cần.

<a id="qc-ban-quyen-lich"></a>

## 4.6. Bản quyền nội dung và Lịch phát hành

Chỉ QC và Quản trị viên **sửa** hai mục này (nhà cung cấp, biên tập viên chỉ xem).

- **Tab Bản quyền nội dung:** **+ Thêm bản quyền** → **Bắt đầu lúc**, **Kết thúc lúc** (để trống nếu không giới hạn), **Lãnh thổ** (mã quốc gia 2 chữ cái như VN, US, nhập rồi Enter; để trống là mọi quốc gia) → **Lưu**. Hệ thống phát hành kiểm tra thời hạn và lãnh thổ khi người dùng xem.
- **Tab Lịch phát hành:** **+ Thêm lịch phát hành** → **Phát hành lúc**, **Hết hiệu lực lúc** (tùy chọn) → **Lưu**. Trạng thái lịch (*Chờ, Đang hiệu lực, Đã gỡ*) tính theo thời gian, không nhập tay.
- **Màn tổng hợp:** menu **Lịch phát hành** và **Bản quyền nội dung** cho xem chéo mọi nội dung (lọc theo trạng thái hoặc hiệu lực). Chỉ đọc; nút **Mở nội dung** để sửa ở tab của nội dung.

<a id="qc-chinh-sach"></a>

## 4.7. Chính sách quyền xem

Quy định mỗi cấp người xem (**FREE → VIP → VIP_PRO**, cấp cao kế thừa cấp thấp) được xem nội dung này thế nào.

1. Menu **Chính sách quyền xem**: danh sách nhóm theo nội dung (số chính sách, cấp đã đặt; nội dung chưa có chính sách có nhãn đỏ). Bấm **Mở** để vào một nội dung, hoặc **+ Thêm chính sách** rồi chọn nội dung bằng ô tìm kiếm.
2. Trang chính sách của một nội dung (có nút **← Chính sách quyền xem** và **Mở nội dung**): mỗi cấp quyền là một thẻ.
3. **+ Thêm chính sách** → chọn **Cấp quyền**; công tắc **Được xem** (tắt thì không đặt chất lượng và số tập); **Chất lượng tối đa** (240p đến 2160p, để trống là không giới hạn); **Số tập tối đa** (chỉ với series, để trống là không giới hạn); **Bắt đầu / Kết thúc** hiệu lực; **Trạng thái** (Hoạt động/Ngừng) → **Lưu**. Mỗi cấp chỉ có một chính sách cho một nội dung.
4. Để xuất bản được, nội dung cần có chính sách cho cấp **FREE**.

<a id="qc-cap-quyen-xem"></a>

## 4.8. Cấp quyền xem (chỉ xem)

Menu **Cấp quyền xem** liệt kê các cấp (FREE, VIP, VIP_PRO) với thứ tự và trạng thái. QC chỉ xem; [Quản trị viên](#admin-cap-quyen-xem) thêm hoặc sửa cấp.

<a id="qc-nhieu-nha"></a>

## 4.9. Các danh mục ở chế độ xem

QC mở được Thể loại phim, đạo diễn, diễn viên, ngôn ngữ để xem (form không sửa được, không có nút Thêm, Xóa). Cách dùng xem [Mục 3](#vai-tro-bien-tap-vien).

---

<a id="vai-tro-quan-tri"></a>

# 5. Vai trò: Quản trị viên

Quản trị viên có **toàn quyền**: mọi chức năng của các vai trò trên (xem [Mục 2](#vai-tro-nha-cung-cap), [3](#vai-tro-bien-tap-vien), [4](#vai-tro-qc)) cho **mọi nội dung và mọi nhà cung cấp**, cộng thêm quản trị hệ thống dưới đây.

Khác biệt khi dùng các chức năng đã mô tả:

- Tạo nội dung thay cho một nhà cung cấp: chọn **Nhà cung cấp** ở form chính ([2.3](#tao-noi-dung)).
- Sửa được **mọi** nội dung kể cả do đối tác tạo, không bị khóa theo người tạo ([Mục 4](#vai-tro-qc)).
- Xóa nội dung nháp ([3.1](#xoa-noi-dung)), duyệt, xuất bản, Transcode ([4.2](#qc-duyet) đến [4.5](#qc-transcode)), bản quyền, lịch phát hành, chính sách ([4.6](#qc-ban-quyen-lich), [4.7](#qc-chinh-sach)).

<a id="admin-tai-khoan"></a>

## 5.1. Tài khoản hệ thống

Menu **Tài khoản hệ thống → Tài khoản hệ thống**: nhân sự nội bộ và tài khoản nhà cung cấp đăng nhập CMS. Cột: tên đăng nhập, họ tên, email liên hệ, loại tài khoản, nhà cung cấp, vai trò, trạng thái. Bộ lọc: loại, trạng thái, nhà cung cấp.

**Tạo tài khoản:**

1. **+ Thêm mới**, điền:
   - **Tên đăng nhập:** 3 đến 50 ký tự gồm chữ, số, dấu chấm, gạch dưới, gạch ngang; không phân biệt hoa thường; không đổi sau khi tạo.
   - **Mật khẩu:** lưu dạng băm, không hiển thị lại.
   - **Họ tên**, **Email liên hệ** (chỉ để liên hệ, không dùng đăng nhập).
   - **Loại tài khoản:** *Nhân sự nội bộ* hoặc *Nhà cung cấp*. Tài khoản nhà cung cấp **bắt buộc chọn Nhà cung cấp**; nội bộ thì để trống.
   - **Trạng thái:** Hoạt động hoặc Khóa.
   - **Vai trò:** chọn nhiều vai trò. Chọn theo bảng ở [Tổng quan](#tong-quan-vai-tro).
2. **Tạo mới**.

**Thao tác ở cuối mỗi dòng:**

- 🔒 **Khóa / Mở khóa:** khóa sẽ **thu hồi mọi phiên** đang đăng nhập của tài khoản. Không tự khóa chính mình.
- 🔑 **Đặt lại mật khẩu:** nhập mật khẩu mới; mọi phiên của tài khoản bị thu hồi và mật khẩu không hiển thị lại.
- **Mở** để sửa họ tên, email, vai trò, trạng thái; **Xóa** (cũng thu hồi phiên).

<a id="admin-vai-tro"></a>

## 5.2. Vai trò

Menu **Vai trò**: nhóm quyền gán cho tài khoản. Cột: mã, tên, mô tả, loại (Hệ thống hay Tự tạo), số quyền.

- **Tạo vai trò:** **Mã vai trò** (chữ hoa, số, gạch dưới, bắt đầu bằng chữ; không đổi sau khi tạo), **Tên**, **Mô tả**, rồi chọn **Quyền** trong cây quyền của từng hệ thống (CMS, OTT, Affiliate).
- Thay đổi quyền của vai trò **có hiệu lực ngay** với người đang dùng. Vai trò mặc định (ADMIN, EDITOR, QC, PROVIDER) là vai trò hệ thống; hãy cẩn thận khi sửa.

<a id="admin-quyen-han"></a>

## 5.3. Quyền hạn & Menu

Menu **Quyền hạn & Menu**: danh mục quyền và menu của từng hệ thống.

- Hai chế độ **Danh sách** và **Dạng cây** (cây cha-con). Bộ lọc **Hệ thống** và **Loại** (Menu chức năng hay Quyền API).
- **Thêm quyền hoặc menu:** **Hệ thống** (bỏ trống là CMS, không đổi sau khi tạo), **Mã quyền** (chữ thường, số, dấu chấm, gạch dưới; ví dụ `noi_dung.tao`; mã của hệ thống khác CMS phải bắt đầu bằng mã hệ thống viết thường, ví dụ `ott.user.xem`), **Tên**, **Loại**, **Đường dẫn giao diện** (bắt buộc với menu, ví dụ `/contents`), **Quyền/menu cha** (cùng hệ thống; menu chỉ làm con của menu), **HTTP method** và **Mẫu đường dẫn API** (cho quyền API), **Thứ tự hiển thị**, **Biểu tượng**, **Mô tả**.
- Menu và nhóm menu trên thanh bên của người dùng dựng trực tiếp từ dữ liệu này.

<a id="admin-snapshot"></a>

### Phát lại snapshot quyền

Ở đầu màn **Quyền hạn & Menu** có nút **Phát lại snapshot quyền**: gửi lại bản chụp đầy đủ quyền lên hàng đợi để OTT và Affiliate đồng bộ lại. Chọn **Hệ thống** (để trống là tất cả) và **Loại snapshot**, bấm **Phát lại**; hệ thống báo số thông điệp đã gửi. Dùng khi một hệ thống bị lệch quyền hoặc vừa dựng mới.

<a id="admin-he-thong"></a>

## 5.4. Hệ thống và khóa dịch vụ

Menu **Hệ thống**: CMS là nơi phân quyền tập trung cho CMS, OTT backend, Affiliate API. Mỗi hệ thống có cây quyền riêng và một **khóa dịch vụ** để backend của nó gọi API kiểm tra quyền.

- **Thêm hệ thống:** **Mã hệ thống** (chữ hoa, không đổi), **Tên**, **Mô tả**, **Trạng thái**. *Tạm ngưng* thì menu của hệ thống đó trả lỗi, kiểm tra quyền báo hệ thống bị vô hiệu và khóa dịch vụ bị từ chối.
- **Cấp khóa dịch vụ:** ở cuối dòng bấm biểu tượng khóa → xác nhận **Cấp khóa**. Khóa **chỉ hiện một lần** ngay sau khi cấp: bấm **Sao chép** và lưu ở nơi an toàn; hệ thống chỉ lưu bản băm nên không xem lại được.
- **Cấp lại:** khóa cũ **mất hiệu lực ngay**, backend phải cập nhật khóa mới.
- **Thu hồi khóa dịch vụ:** backend của hệ thống đó không gọi được API kiểm tra quyền nữa.

<a id="admin-phien"></a>

## 5.5. Phiên đăng nhập

Menu **Phiên đăng nhập**: sổ phiên của mọi hệ thống. CMS chỉ ghi định danh phiên, không giữ token. Cột: hệ thống, tài khoản, trạng thái (*Đang hoạt động, Đã kết thúc, Hết hạn*), lý do kết thúc (LOGOUT, PASSWORD_CHANGED, ACCOUNT_LOCKED…), IP, bắt đầu, hết hạn. Lọc theo hệ thống và trạng thái.

**Thu hồi phiên:** biểu tượng cấm ở cuối dòng → **Thu hồi**: phiên bị hủy ở hệ thống tương ứng và access token bị từ chối ngay.

<a id="admin-danh-muc"></a>

## 5.6. Nhà cung cấp và Ngôn ngữ

- **Nhà cung cấp** (nhóm Danh mục hệ thống): studio, creator hoặc đội nội bộ cung cấp phim. Trường: **Tên**, **Loại** (Studio, Creator, Nội bộ), **Thông tin liên hệ**, **Trạng thái** (Hoạt động, Ngừng hợp tác). Cần tạo nhà cung cấp trước rồi mới tạo [tài khoản nhà cung cấp](#admin-tai-khoan) và nội dung của họ.
- **Danh mục ngôn ngữ:** **Mã ngôn ngữ** (dạng `vi`, `en`, `zh-Hans`; không đổi sau khi tạo), **Tên**, **Trạng thái**, **Thứ tự hiển thị**. Ngôn ngữ được dùng cho bản dịch, phụ đề, ảnh bìa nên cân nhắc trước khi ngừng sử dụng.

<a id="admin-cap-quyen-xem"></a>

## 5.7. Cấp quyền xem

Menu **Cấp quyền xem**: **Mã cấp** (chữ hoa, bắt đầu bằng chữ; **không đổi sau khi tạo** vì OTT tham chiếu), **Thứ tự** (số nhỏ là cấp thấp), **Trạng thái**. Cấp cao kế thừa quyền của cấp thấp. Chính sách của từng nội dung gán theo cấp này ([4.7](#qc-chinh-sach)).

---

<a id="phu-luc"></a>

# 6. Phụ lục

<a id="phu-luc-trang-thai"></a>

## A. Các trạng thái và ai làm gì

| Trạng thái nội dung | Ai làm gì | Chuyển tiếp |
| --- | --- | --- |
| **Bản nháp** | Nhà cung cấp, biên tập viên sửa và tải tệp ([Mục 2](#vai-tro-nha-cung-cap)) | **Gửi QC** → Chờ QC |
| **Chờ QC** | QC xem xét ([4.2](#qc-duyet)); chạy Transcode ([4.5](#qc-transcode)) | **Duyệt** → Sẵn sàng; **Trả lại** (bắt buộc lý do) → Bản nháp |
| **Sẵn sàng** | QC chờ xuất bản | **Xuất bản** → Đã xuất bản |
| **Đã xuất bản** | Hiển thị với người xem theo lịch và bản quyền | **Gỡ xuất bản** → Gỡ xuất bản |
| **Gỡ xuất bản** | Ẩn với người xem | **Kích hoạt lại** → Đã xuất bản |

Quy tắc cần nhớ: "Trả lại" chỉ từ *Chờ QC*; nội dung đã từng xuất bản **không xóa được**; tập phim chỉ xuất bản khi series đã xuất bản; series không có media riêng.

<a id="phu-luc-loi-media"></a>

## B. Lỗi xử lý media

Khi phiên bản có nhãn **Lỗi media**, ô lỗi màu đỏ hiện **mã lỗi**, nội dung lỗi đầy đủ và gợi ý. Sau khi sửa nguyên nhân, bấm **Transcode lại** (QC).

| Mã | Nghĩa | Việc nên làm |
| --- | --- | --- |
| `PROVIDER_REJECTED` | Dịch vụ mã hóa từ chối nguồn (không thử lại), ví dụ video **không có luồng âm thanh** | Kiểm tra lại video gốc, tải lại bản đúng rồi Transcode lại |
| `RETRIES_EXHAUSTED` | Hết số lần thử lại (5 lần) | Thử Transcode lại |
| `MEDIA_STUCK` | Quá 24 giờ không có kết quả | Thử Transcode lại |
| `SOURCE_NOT_FOUND`, `INVALID_SOURCE` | Không thấy hoặc không hợp lệ video gốc | Tải lại video gốc |
| `INVALID_REQUEST`, `INTERNAL_ERROR`, `PROFILE_NOT_FOUND`, `DRM_NOT_ENABLED` | Lỗi cấu hình hoặc nội bộ | Thử lại; nếu lặp lại, báo quản trị kèm mã theo dõi |

Mã lạ vẫn hiển thị nguyên văn. Bản ghi lỗi cũ có thể không có mã, chỉ có nội dung.

<a id="phu-luc-su-co"></a>

## C. Xử lý sự cố thường gặp

| Hiện tượng | Nguyên nhân thường gặp | Cách xử lý |
| --- | --- | --- |
| Không thấy một menu | Tài khoản chưa có quyền | Nhờ quản trị viên gán vai trò |
| Nút **Gửi QC / Duyệt / Xuất bản** bị tắt | Còn điều kiện ✕ trong khung Quy trình duyệt | Rê chuột vào nút và đọc dòng ✕ để biết thiếu gì |
| Sau khi tải video, vẫn báo thiếu video gốc | Kết quả media đến qua hàng đợi, trang không tự cập nhật | Bấm **Làm mới** ở khung Media |
| Form không sửa được | Nội dung không ở Bản nháp (nhà cung cấp), hoặc do đối tác tạo (QC), hoặc thiếu quyền | Đọc ghi chú vàng đầu form |
| "Dữ liệu đã bị người khác thay đổi" | Người khác lưu trước | Tải lại, kiểm tra rồi nhập lại thay đổi |
| Không bấm được gì, có thanh "Đang tải tệp" | Đang có tệp tải lên | Chờ xong, hoặc bấm **Hủy tải** |
| Bị đưa về trang đăng nhập | Phiên hết hạn, hoặc tài khoản bị khóa/đổi mật khẩu | Đăng nhập lại; nếu bị khóa, liên hệ quản trị viên |
| Không xóa được nội dung | Đã từng xuất bản, hoặc còn dữ liệu phụ thuộc | Với nội dung đã xuất bản dùng **Gỡ xuất bản** |
