# Pha 2: Lập kế hoạch test

Mục tiêu: cho người dùng thấy trước chính xác bạn sẽ làm gì, để họ sửa hoặc chấp thuận trước khi có file nào bị ghi.

## Cách lập kế hoạch

1. Điền mẫu `templates/plan.md` bằng kết quả của pha phân tích. Giữ nguyên thứ tự và tên các mục. Mục không có nội dung thì ghi "Không có". Hiển thị toàn văn kế hoạch theo mẫu, không thay bằng đoạn tóm tắt, kể cả khi `mode=auto`.
   - Tiêu đề các mục chép đúng từ mẫu, có cả số thứ tự (ví dụ `## 4. Đơn vị và test dự kiến`). Không đổi tên, rút gọn hay bỏ mục nào, kể cả mục "8. Xác nhận" khi `mode=auto` (dùng câu dành cho `mode=auto` trong mẫu).
2. **Tham số đã hiểu:**
   - ghi đủ 6 tham số với giá trị thực sẽ dùng;
   - tham số dùng mặc định thì ghi rõ "(mặc định)";
   - giá trị không hợp lệ thì ghi giá trị người dùng đưa, lý do không hợp lệ và giá trị mặc định thay thế.
3. **Quy ước đã nhận diện:** ghi quy ước sẽ theo, kèm file test mẫu làm căn cứ. Nếu phải chọn giữa nhiều quy ước, ghi lựa chọn và lý do.
4. **Đơn vị và test dự kiến:** mỗi đơn vị mục tiêu một dòng, gồm:
   - file test sẽ tạo hoặc sửa;
   - các tình huống chính sẽ test: nhánh chính, biên, lỗi.

   Chỉ sửa file test do agent tạo hoặc thêm test case mới vào file test có sẵn. Không sửa test case có sẵn.
5. **Đơn vị bỏ qua:** gồm đơn vị khó test (kèm lý do) và đơn vị bị loại vì sinh tự động hoặc không có logic.
6. **Thay đổi cấu hình/dependency:**
   - liệt kê chính xác file build/manifest/cấu hình test cần sửa và nội dung thêm vào;
   - những mục này luôn cần người dùng chấp thuận, **kể cả khi `mode=auto`**.
7. **Môi trường:** tình trạng toolchain, trạng thái git đầu phiên, coverage "trước" hoặc lý do không đo được.
8. Nếu `coverage-target` được đặt, ước lượng kế hoạch có khả năng đạt mục tiêu không. Nếu không, nói rõ phần còn thiếu nằm ở đâu.

## Điểm dừng xác nhận

- **`mode=confirm` (mặc định):**
  - hiển thị kế hoạch và **dừng lại**;
  - không ghi file nào cho tới khi người dùng xác nhận;
  - trước khi dừng, xóa file sinh ra khi đo coverage "trước" (thư mục build, báo cáo coverage, cache của trình chạy test) nếu dự án không có chúng từ đầu phiên, theo cùng quy tắc ở pha kiểm chứng. Danh sách đầy đủ của từng stack nằm ở mục "Thư mục/file sinh ra khi chạy" trong playbook. Sau khi xóa, liệt kê lại thư mục gốc và các thư mục test để chắc không còn sót. Người dùng có thể không bao giờ xác nhận, nên dự án phải sạch ngay tại điểm dừng.
  - Người dùng có thể sửa kế hoạch: bỏ đơn vị, đổi quy ước, từ chối dependency. Cập nhật kế hoạch theo yêu cầu, hiển thị lại phần thay đổi, rồi chờ xác nhận lần nữa.
- **`mode=auto`:**
  - hiển thị kế hoạch rồi chuyển ngay sang pha sinh test;
  - ngoại lệ: nếu kế hoạch có thay đổi cấu hình/dependency, vẫn dừng hỏi riêng về các thay đổi đó.
  - Người dùng từ chối thì bỏ các đơn vị phụ thuộc vào chúng và ghi vào báo cáo.

Người dùng từ chối toàn bộ kế hoạch thì chuyển thẳng sang pha báo cáo với ghi chú "không sinh test theo yêu cầu người dùng".
