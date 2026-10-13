# Pha 1: Phân tích dự án

Mục tiêu: hiểu đủ về dự án để viết test đúng quy ước và chạy được. Không ghi file nào của dự án trong pha này.

## 1. Ghi nhận trạng thái đầu phiên

Việc này phục vụ kiểm tra thay đổi ngoài phạm vi ở pha kiểm chứng.

- **Dự án là git repo:** chạy `git status --porcelain` và lưu kết quả.
  - Các file đã thay đổi từ trước phiên không phải do bạn gây ra. Không được hoàn tác chúng.
- **Không phải git repo:** ghi nhận điều đó.
  - Từ đây, lưu danh sách mọi file bạn tạo hoặc sửa trong phiên.

## 2. Nhận diện stack

- Làm theo `detection.md`: tìm module trong `scope`, xác định stack của từng module.
- Chỉ đọc playbook của các stack đã nhận diện.
- Stack không được hỗ trợ hoặc không nhận diện được: dừng và báo người dùng.

Nếu `scope` là `changed:<git-ref>`:

- chạy `git diff --name-only <git-ref>` để lấy danh sách file;
- chỉ giữ file production (bỏ file test, cấu hình, tài liệu, file bị xóa);
- dự án không phải git repo hoặc ref không tồn tại thì báo người dùng và hỏi `scope` khác.

## 3. Nhận diện công cụ build và test

Với mỗi module, đọc manifest, cấu hình build và cấu hình test để xác định:

- build tool và wrapper nếu có (`mvnw`, `gradlew`, npm/yarn/pnpm theo lock file, poetry/uv/pip…);
- test framework và phiên bản;
- thư viện mock và assertion;
- công cụ đo coverage;
- lệnh chạy toàn bộ test và lệnh chạy một file hoặc một test đơn lẻ (xem mục Commands của playbook).

Quy tắc:

- **Dự án đã có test framework:** mọi test mới dùng framework đó. Không đưa framework khác vào.
- **Dự án chưa có test framework:** đề xuất bộ công cụ mặc định trong playbook. Ghi vào mục dependency cần thêm của kế hoạch. Chỉ thêm khi người dùng chấp thuận.

## 4. Học quy ước test sẵn có

1. Mỗi module lấy **tối đa khoảng 5 file test** gần đơn vị mục tiêu nhất.
   - Ưu tiên cùng package hoặc thư mục, sau đó tới phần còn lại của module.
   - Không đọc toàn bộ test của dự án.
2. Rút ra từ mẫu:
   - vị trí và cách đặt tên file test;
   - cách đặt tên test case (ví dụ `should...When...`, `test_<hàm>_<tình huống>`, mô tả tiếng tự nhiên);
   - cấu trúc: Arrange-Act-Assert, Given-When-Then, nhóm `describe`/`@Nested`…;
   - style assertion: thư viện và cách viết;
   - cách tạo mock, fixture và dữ liệu test (builder, factory, file fixture);
   - helper, base class và cấu hình test dùng chung có thể tái sử dụng.
3. Các mẫu không thống nhất: chọn quy ước phổ biến nhất trong module mục tiêu và ghi lựa chọn vào kế hoạch.
4. Module chưa có test nào: dùng mục "Test location and naming" và "Test patterns" của playbook.

## 5. Liệt kê và ưu tiên đơn vị mục tiêu

Liệt kê các đơn vị code trong `scope`: class, hàm, component, service, pipe, hook, widget, view model…

**Loại bỏ:**

- code sinh tự động (thư mục `generated`, file có chú thích "generated", code do annotation processor hoặc codegen sinh ra);
- file cấu hình thuần;
- lớp chỉ khai báo dữ liệu không có logic: DTO/POJO chỉ có getter/setter, entity không có hành vi, hằng số, interface không có mặc định.

**Ưu tiên theo thứ tự:**

1. Đơn vị chứa logic nghiệp vụ chưa có test.
2. Đơn vị có test nhưng coverage thấp (khi đo được).
3. Đơn vị đã có test tốt: chỉ bổ sung nhánh còn thiếu nếu `coverage-target` yêu cầu.

**Đơn vị khó test:**

- Ví dụ: phụ thuộc cứng không tiêm được, gọi static/singleton không mock được, tự tạo kết nối mạng hoặc DB bên trong.
- Không sửa code của nó.
- Đánh dấu để đưa vào báo cáo kèm gợi ý refactor.

## 6. Kiểm tra môi trường chạy test

Kiểm tra toolchain cần cho từng module theo playbook, ví dụ:

- JDK và Maven/Gradle (hoặc wrapper);
- Node và trình quản lý package; dependency đã cài chưa;
- Python, môi trường ảo và pytest;
- Flutter SDK;
- Android SDK;
- Xcode (chỉ có trên macOS).

Chạy lệnh kiểm tra phiên bản nhẹ, không chạy build nặng ở bước này.

Nếu thiếu toolchain:

- báo người dùng thiếu gì;
- nói rõ test sẽ không được kiểm chứng bằng cách chạy;
- hỏi người dùng muốn dừng hay tiếp tục sinh test mà không kiểm chứng. Hỏi ngay cả khi `mode=auto`.

## 7. Đo coverage "trước"

Nếu dự án có công cụ coverage và toolchain đầy đủ:

- chạy lệnh coverage theo playbook, chỉ trên phạm vi mục tiêu nếu công cụ cho phép;
- ghi số liệu của phạm vi mục tiêu.

"Có công cụ coverage" gồm cả công cụ gọi được qua dòng lệnh mà không cần sửa cấu hình (ví dụ goal JaCoCo của Maven, `--coverage` của test runner). Đo ở bước này trong mọi `mode`, kể cả `mode=confirm`. Không hoãn sang sau khi người dùng xác nhận: lúc đó test mới đã được thêm nên số liệu không còn là "trước". File sinh ra khi đo được dọn ở điểm dừng (pha 2).

Báo cáo coverage ghi vào thư mục mặc định của công cụ trong dự án, hoặc một thư mục tạm trong dự án. Không ghi ra ngoài dự án: client có thể không cho phép agent xóa ở đó, nên file sẽ bị bỏ lại. Cách dọn dẹp xem ở pha 2 và pha 4.

Nếu không đo được (không có công cụ, lệnh lỗi, hoặc chạy quá lâu): ghi "không đo được" kèm lý do. Không ước đoán.

## Kết quả của pha

Chuyển sang pha 2 khi đã có đủ:

- danh sách module, stack và playbook;
- công cụ build/test/mock/coverage và lệnh chạy;
- quy ước đã nhận diện;
- danh sách đơn vị mục tiêu đã ưu tiên và danh sách đơn vị khó test;
- tình trạng toolchain;
- trạng thái git đầu phiên hoặc ghi chú "không phải git repo";
- coverage "trước" hoặc lý do không đo được.
