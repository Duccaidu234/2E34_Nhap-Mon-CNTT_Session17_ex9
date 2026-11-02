# BÁO CÁO PHÂN TÍCH KIẾN TRÚC BROWSER PROFILE & QUY TRÌNH AN TOÀN BẢO MẬT

**Đơn vị:** Mạng lưới Y tế CarePulse — Trạm Y tế Phường  
**Đối tượng áp dụng:** Bác sĩ trực chính, Điều dưỡng viên phụ tá, Bác sĩ/Sinh viên thực tập  
**Mục tiêu:** Triệt tiêu 100% xung đột phiên đăng nhập (Session Conflict), bảo mật dữ liệu y tế và tối ưu hiệu suất thao tác lâm sàng.

---

## PHẦN 1: BÁO CÁO PHÂN TÍCH KIẾN TRÚC ĐA PROFILE

### 1. Nguyên nhân sâu xa dẫn đến hiện tượng 'Văng phiên đăng nhập' (Session Conflict)

Khi tất cả nhân sự tại trạm y tế dùng chung 01 Profile trình duyệt mặc định, trình duyệt lưu trữ toàn bộ dữ liệu vào duy nhất một thư mục `Default` nằm trong đường dẫn `User Data` của Chrome (ví dụ: `%LOCALAPPDATA%\Google\Chrome\User Data\Default`).

#### Cơ chế lưu trữ Cookies & Xung đột Token dưới góc độ kỹ thuật:

```
+---------------------------------------------------------------------------------+
|                           CHROME USER DATA DIRECTORY                            |
|                                (Cấu trúc 1 Profile)                             |
|                                                                                 |
|   +-------------------------------------------------------------------------+   |
|   | /Default Directory                                                      |   |
|   |  ├── Cookies (SQLite Database File - Shared Across All Tabs)            |   |
|   |  ├── Local Storage / IndexedDB                                          |   |
|   |  └── Preferences & History                                              |   |
|   +-------------------------------------------------------------------------+   |
+---------------------------------------------------------------------------------+
                                       |
    +----------------------------------+----------------------------------+
    |                                                                     |
    v                                                                     v
[ Tab 1: Bác sĩ trực ]                                 [ Tab 2: Thực tập sinh ]
Gửi Request: Cookie (CarePulse_Token_Doctor)           Đăng nhập tài khoản khác / SSO
Action: Đăng nhập CarePulse EMR                        Action: Đăng nhập PTIT / GitHub / Student EMR
                                                                          |
                                                                          v
                                                       GHI ĐÈ (OVERWRITE) COOKIE SANG:
                                                       (CarePulse_Token_Student)
                                                                          |
    +---------------------------------------------------------------------+
    v
[ Thao tác tiếp theo của Bác sĩ ]
Trình duyệt đính kèm Cookie đã bị ghi đè -> Server phát hiện vô hiệu -> Lỗi HTTP 401/403 -> VĂNG LOGOUT
```

1. **Cơ sở dữ liệu lưu trữ tập trung:** Trình duyệt quản lý Cookie thông qua một tệp cơ sở dữ liệu SQLite duy nhất (`Cookies`). Mọi tab (tab) mở ra trong cùng một Profile đều đọc và ghi dữ liệu vào chung tệp SQLite này.
2. **Cơ chế ghi đè Cookie (Cookie Overwrite):**
   * Bác sĩ trực đang đăng nhập hệ thống CarePulse với Token phiên làm việc của Bác sĩ (`CarePulse_Token_Doctor`).
   * Bạn thực tập sinh mở tab mới trên cùng trình duyệt để truy cập Cổng học tập PTIT/GitHub (có sử dụng chung hạ tầng xác thực SSO) hoặc mở một phiên đăng nhập CarePulse bằng tài khoản sinh viên.
   * Do cùng lưu trữ vào một không gian domain/path trong tệp SQLite, Cookie đăng nhập của sinh viên (`CarePulse_Token_Student`) ngay lập tức **ghi đè (overwrite)** lên Session Cookie hiện tại của Bác sĩ.
3. **Phản hồi vô hiệu từ Server:** Khi Bác sĩ thực hiện thao tác lâm sàng tiếp theo (ví dụ: bấm *Kê đơn* hoặc *Chuyển viện*), trình duyệt gửi đính kèm Session Cookie đã bị ghi đè. Máy chủ CarePulse đối soát thấy Context bị thay đổi hoặc tài khoản sinh viên không có thẩm quyền, trả về phản hồi `HTTP 401 Unauthorized` / `HTTP 403 Forbidden` và lập tức cưỡng chế **văng đăng xuất (Logout)**.

---

### 2. Thiết kế Mô hình 3 Profile chuyên dụng cho Máy tính Trạm khám CarePulse

Giải pháp triệt tiêu tận gốc sự cố là phân tách hệ thống thành 3 **User Data Profiles** riêng biệt (`/Profile 1`, `/Profile 2`, `/Profile 3`). Mỗi Profile hoạt động như một trình duyệt độc lập với tiến trình và bộ nhớ được cách ly tuyệt đối.

```
+---------------------------------------------------------------------------------+
|                         CHROME USER DATA DIRECTORY                              |
+--------------------------+--------------------------+---------------------------+
|                          |                          |                           |
v                          v                          v                           
[ PROFILE 1: WORK ]        [ PROFILE 2: PERSONAL ]    [ PROFILE 3: SANDBOX/TEST ] 
• Thư mục: /Profile 1      • Thư mục: /Profile 2      • Thư mục: /Profile 3       
• Storage: Isolation A     • Storage: Isolation B     • Storage: Isolation C      
• Theme: Dark Blue         • Theme: Teal / Green      • Theme: Amber / Orange     
```

#### Chi tiết cấu hình Mô hình 3 Profile:

| Tiêu chí | 🏥 Profile 1: Work (Medical) | 👤 Profile 2: Personal (Staff) | 🧪 Profile 3: Sandbox / Test |
| :--- | :--- | :--- | :--- |
| **Tên Profile** | `CarePulse_Work` | `Staff_Personal` | `PTIT_Sandbox` |
| **Mục đích sử dụng** | Phục vụ trực khám, kê đơn, tra cứu bệnh án, thao tác hệ thống y tế CarePulse và Cổng BHYT. | Phục vụ nhu cầu cá nhân ngoài giờ làm việc của nhân viên (xem tin tức, MXH, mua sắm). | Phục vụ sinh viên thực tập học tập (Cổng PTIT, GitHub), thử nghiệm ứng dụng mà không ảnh hưởng công việc chính. |
| **Theme thị giác** | **Dark Blue (Xanh thẫm)**<br>Trực quan chuyên nghiệp, dễ nhận diện môi trường làm việc y tế. | **Teal / Green (Xanh ngọc)**<br>Phân biệt rõ môi trường thư giãn/cá nhân. | **Amber / Orange (Cam sáng)**<br>Cảnh báo môi trường thử nghiệm / tạm thời. |
| **Cookies** | Độc lập hoàn toàn. Chỉ lưu trữ duy nhất Session làm việc chính thức của CarePulse và BHYT. | Độc lập hoàn toàn. Lưu Cookie Facebook, Shopee, Gmail cá nhân. | Độc lập hoàn toàn. Tự dọn dẹp hoặc chỉ chứa Cookie học tập PTIT/GitHub. |
| **History (Lịch sử)** | Chỉ chứa URL liên quan đến y tế, hệ thống EMR, tra cứu dược phẩm. | Lưu lịch sử cá nhân; không xuất hiện gợi ý tìm kiếm riêng tư khi chiếu hội chẩn. | Lưu lịch sử tra cứu tài liệu học tập và lập trình. |
| **Bookmarks** | Bộ Bookmark cây thư mục chuẩn 30+ dịch vụ y khoa. | Bookmark cá nhân (giải trí, tin tức, mua sắm). | Bookmark tài liệu học tập, GitHub, Cổng PTIT. |
| **Extensions** | Chỉ cài Extension hỗ trợ y tế (Trình đọc Smartcard BHYT, PDF Viewer). | Tiện ích cá nhân (Adblock, Trợ lý tiếng Việt). | Tiện ích Dev/Test (React DevTools, JSON Viewer). |

---

## PHẦN 2: THIẾT KẾ HỆ THỐNG BOOKMARK & QUY TRÌNH AN TOÀN

### 3. Thiết kế Cấu trúc Cây thư mục Bookmark chuẩn (Ctrl + Shift + O)

Trên Profile `CarePulse_Work`, tổ chức thanh Bookmark bar thành 3 thư mục chính nhằm phân loại và tối ưu hóa thời gian tìm kiếm liên kết dưới 5 giây mỗi ca khám:

```
📁 Bookmarks Bar
├── 📁 01_CarePulse_Clinic
│   ├── 🔗 1. Hệ thống Quản lý Bệnh án Điện tử CarePulse (EMR) - https://carepulse.vn/emr
│   ├── 🔗 2. Cổng Tiếp nhận & Đăng ký Bệnh nhân Trực tuyến - https://carepulse.vn/reception
│   ├── 🔗 3. Hệ thống Quản lý Dược & Kho Vật tư Y tế - https://carepulse.vn/pharmacy-inventory
│   ├── 🔗 4. Báo cáo Thống kê Ca trực & Chuyển tuyến - https://carepulse.vn/reports
│   └── 🔗 5. Lịch Trực Phường & Phân công Lâm sàng - https://carepulse.vn/schedule
├── 📁 02_Pharma_Portal
│   ├── 🔗 1. Cổng Giao dịch Điện tử Bảo hiểm Y tế Việt Nam - https://giamdinhbhyt.vss.gov.vn
│   ├── 🔗 2. Tra cứu Dược thư Quốc gia Việt Nam Online - https://duocthu.org.vn
│   ├── 🔗 3. Ngân hàng Dữ liệu Dược Cục Quản lý Dược (DrugBank) - https://drugbank.vn
│   ├── 🔗 4. Cổng Thông tin Quản lý An toàn Thuốc & ADR Quốc gia - https://adravn.org.vn
│   └── 🔗 5. Tra cứu Danh mục Thuốc BHYT Chi trả - https://baohiemxahoi.gov.vn/drug-list
└── 📁 03_Medical_Tools
    ├── 🔗 1. Công cụ Tính toán Liều lượng Thuốc & Chỉ số Lâm sàng (MDCalc) - https://mdcalc.com
    ├── 🔗 2. Tra cứu Tương tác Thuốc Chuyên sâu (Drugs.com Checker) - https://drugs.com/drug_interactions.html
    ├── 🔗 3. Cổng Thư viện Y khoa Quốc gia (PubMed / Medline) - https://pubmed.ncbi.nlm.nih.gov
    ├── 🔗 4. Trợ lý Phác đồ Điều trị Chuyên khoa (UpToDate Medical) - https://uptodate.com
    └── 🔗 5. Tra cứu Mã Bệnh ICD-10 Bộ Y tế - https://icd10.kcb.vn
```

---

### 4. Hướng dẫn Mẹo Favicon-Only tiết kiệm 300% diện tích thanh công cụ

Mẹo **Favicon-Only** giúp loại bỏ hoàn toàn phần nhãn văn bản (Text Label) của dấu trang, chỉ hiển thị duy nhất biểu tượng trang web (Icon). Kỹ thuật này giúp tiết kiệm không gian gấp 3-4 lần, hiển thị hơn 30+ Bookmark trên cùng một thanh công cụ mà không bị tràn màn hình.

#### Quy trình 3 bước thực hiện:

1. **Bước 1 (Mở Bookmark Manager):** Tại Profile `CarePulse_Work`, nhấn tổ hợp phím **`Ctrl + Shift + O`** để mở trình quản lý dấu trang.
2. **Bước 2 (Thêm Bookmark 4 trang thường dùng):** Truy cập lần lượt 4 trang web:
   * System CarePulse EMR (`https://carepulse.vn/emr`)
   * Cổng BHYT (`https://giamdinhbhyt.vss.gov.vn`)
   * Dược thư Quốc gia (`https://duocthu.org.vn`)
   * Google Dịch Y khoa (`https://translate.google.com`)  
   Nhấn **`Ctrl + D`** tại mỗi trang để lưu vào thanh **Bookmarks Bar**.
3. **Bước 3 (Xóa tên - Favicon-Only):** 
   * Nhấp chuột phải vào từng Bookmark trên thanh dấu trang -> chọn **Chỉnh sửa (Edit)**.
   * Xóa sạch toàn bộ chữ trong ô **Tên (Name)** -> Nhấn **Lưu (Save)**.

```
[Trước khi áp dụng]:
| 🏥 CarePulse EMR | 💳 Cổng Giám định BHYT | 📖 Dược thư Quốc gia | 🌐 Google Dịch | (Chiếm hết diện tích)

[Sau khi áp dụng Favicon-Only]:
| 🏥 | 💳 | 📖 | 🌐 | 📁 01_CarePulse_Clinic | 📁 02_Pharma_Portal | 📁 03_Medical_Tools | (Tiết kiệm 300% không gian)
```

---

### 5. Xử lý Bẫy an ninh & Quy tắc Quản trị Dữ liệu phòng khám

#### 🛑 Nguy cơ Bẫy an ninh & Giải pháp khi có Đoàn thanh tra / Khách mượn máy (30 phút)

* **Phương án lựa chọn:** Kích hoạt **`Guest Profile` (Chế độ Khách)**.
* **Giải thích lý do không chọn `Incognito Mode` (Ẩn danh):**
  * `Incognito Mode` vẫn kế thừa dữ liệu từ Profile đang mở: Khách sử dụng Incognito vẫn có thể xem được toàn bộ danh sách Bookmarks, tiếp cận các Extensions cho phép chạy trong ẩn danh, và được tự động điền (Autofill) lịch sử gõ từ Profile chính.
  * `Guest Profile` khởi tạo một môi trường hoàn toàn cô lập: **Không có Bookmarks, Không Lịch sử, Không Lưu mật khẩu, Không Cookie cũ, Không gợi ý gõ từ trước**. Khi tắt cửa sổ Guest, toàn bộ vết duyệt web của khách bị hủy sạch ngay lập tức.

#### ⚠️ Cảnh báo Nguy cơ khi bật Cloud Sync tài khoản cá nhân trên máy trạm y tế

Nhân sự tuyệt đối **KHÔNG** bật tính năng đồng bộ hóa đám mây (`Google Cloud Sync`) bằng Gmail cá nhân trên Profile công việc (`CarePulse_Work`) vì 2 nguy cơ an ninh nghiêm trọng:

1. **Rò rỉ dữ liệu Y tế & Bệnh nhân (Vi phạm An toàn Thông tin Y tế):**  
   Khi bật Cloud Sync, toàn bộ Session, Lịch sử truy cập hồ sơ bệnh án, Mật khẩu lưu đệm và Dữ liệu EMR của bệnh nhân sẽ bị tự động tải lên Cloud và đồng bộ về máy tính/điện thoại cá nhân của nhân viên ở nhà. Điều này vi phạm nghiêm trọng quy định bảo mật dữ liệu y tế cá nhân.
2. **Lây nhiễm Mã độc & Trộn lẫn Dữ liệu (Cross-Contamination):**  
   Các tiện ích mở rộng (Extensions) cá nhân, mật khẩu rò rỉ từ các trang mạng xã hội hoặc mã độc/spyware từ máy tính cá nhân của nhân viên có thể thông qua Cloud Sync tự động cài đặt ngược vào máy tính trạm y tế, làm đe dọa toàn bộ hệ thống mạng nội bộ phòng khám.