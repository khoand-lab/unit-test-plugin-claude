# Kế hoạch unit test

## 1. Tham số đã hiểu

| Tham số | Giá trị dùng | Ghi chú |
|---|---|---|
| `scope` | <phạm vi> | <ví dụ: 4 file thay đổi so với `main`> |
| `coverage-target` | <% hoặc "không đặt"> | |
| `mode` | <confirm/auto> | <"(mặc định)" nếu không chỉ định> |
| `max-fix-attempts` | <1–10> | <nếu giá trị không hợp lệ: "người dùng đưa `0`, ngoài khoảng 1–10, dùng mặc định 3"> |
| `failed-test-policy` | <remove/disable> | |
| `save-report` | <không lưu / true / đường dẫn> | |

## 2. Dự án

| Module | Stack | Playbook | Build tool | Test framework | Mock/assertion | Coverage |
|---|---|---|---|---|---|---|
| <đường dẫn module> | <stack> | <playbooks/...> | <tool> | <framework + phiên bản> | <thư viện> | <công cụ hoặc "không có"> |

Lệnh chạy test:

- Một file/test: `<lệnh>`
- Toàn bộ module: `<lệnh>`
- Coverage: `<lệnh hoặc "không có">`

## 3. Quy ước đã nhận diện

- Vị trí và tên file test: <quy ước>
- Tên test case: <quy ước>
- Cấu trúc: <AAA / Given-When-Then / nhóm theo ...>
- Assertion: <thư viện và style>
- Mock và fixture: <cách làm>
- Helper tái sử dụng: <danh sách hoặc "Không có">
- File mẫu làm căn cứ: <danh sách file>
- Lựa chọn khi quy ước không thống nhất: <quy ước đã chọn và lý do, hoặc "Không có">

## 4. Đơn vị và test dự kiến

| # | Đơn vị | File test (tạo/sửa) | Tình huống sẽ test |
|---|---|---|---|
| 1 | <Class#method / component> | <đường dẫn> (tạo) | <nhánh chính; biên; lỗi> |

## 5. Đơn vị bỏ qua

| Đơn vị | Lý do | Gợi ý refactor |
|---|---|---|
| <đơn vị> | <khó test: phụ thuộc cứng... / sinh tự động / không có logic> | <gợi ý hoặc "—"> |

## 6. Thay đổi cấu hình/dependency cần chấp thuận

| File | Thay đổi | Lý do |
|---|---|---|
| <pom.xml / package.json / ...> | <nội dung thêm> | <lý do> |

## 7. Môi trường

- Toolchain: <đủ / thiếu ... → test sẽ "chưa kiểm chứng">
- Trạng thái đầu phiên: <git sạch / git có N file đã thay đổi từ trước / không phải git repo, sẽ tự theo dõi file đã ghi>
- Coverage trước: <số liệu của phạm vi mục tiêu / "không đo được: <lý do>">
- Khả năng đạt `coverage-target`: <đánh giá hoặc "không đặt">

## 8. Xác nhận

<Với mode=confirm: "Kế hoạch đang chờ bạn xác nhận. Trả lời đồng ý để bắt đầu sinh test, hoặc cho biết phần cần sửa.">
<Với mode=auto: "Chế độ tự động: bắt đầu sinh test." (nếu có mục 6 thì vẫn hỏi chấp thuận riêng mục 6)>
