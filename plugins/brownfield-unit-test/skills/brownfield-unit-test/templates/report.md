# Báo cáo unit test

## 1. Tóm tắt

- Phạm vi: <scope>
- Đơn vị đã test: <N>/<tổng đơn vị trong kế hoạch>
- Test case: <X> đã kiểm chứng (pass), <Y> đã gỡ, <Z> đã disable, chưa hoàn thành, <W> chưa kiểm chứng
- Coverage: <trước> → <sau> (<công cụ>) / "không đo được: <lý do>"
- `coverage-target`: <đạt / chưa đạt: <phần còn thiếu> / không đặt>

## 2. Test đã tạo

| File test | Tạo/sửa | Đơn vị | Số test case | Trạng thái |
|---|---|---|---|---|
| <đường dẫn> | <tạo/sửa> | <Class#method / component> | <n> | <n pass; n đã gỡ; ...> |

## 3. Coverage

| Phạm vi | Trước | Sau | Công cụ |
|---|---|---|---|
| <module/package/file> | <% hoặc "không đo được"> | <% hoặc "không đo được"> | <công cụ> |

Lý do không đo được (nếu có): <lý do>

## 4. Test fail đã xử lý (`failed-test-policy=<remove/disable>`)

### <Tên test case> — <đã gỡ / đã disable, chưa hoàn thành>

- Đơn vị: <đơn vị>
- Số vòng sửa đã dùng: <n>/<max-fix-attempts>
- Lỗi cuối cùng:

```
<thông báo lỗi>
```

- Mã test (với `remove`) hoặc vị trí test (với `disable`):

```
<mã test đã thử lần cuối>
```

## 5. Nghi vấn bug

| Đơn vị | Hành vi mong đợi | Hành vi quan sát | Mã test minh họa |
|---|---|---|---|
| <đơn vị> | <mong đợi> | <quan sát> | <xem khối mã bên dưới> |

## 6. Đơn vị bỏ qua

| Đơn vị | Lý do | Gợi ý refactor |
|---|---|---|
| <đơn vị> | <lý do> | <gợi ý hoặc "—"> |

## 7. Vấn đề và khuyến nghị

- Lỗi có sẵn trong dự án: <test cũ fail, lỗi build ... hoặc "Không có">
- Thay đổi ngoài phạm vi đã hoàn tác: <file và thay đổi, hoặc "Không có">
- Toolchain: <đủ / thiếu ... → test "chưa kiểm chứng">
- Khuyến nghị: <bước tiếp theo>

## 8. Thay đổi file

| File | Loại | Thay đổi |
|---|---|---|
| <đường dẫn> | <test / helper / cấu hình đã chấp thuận> | <tạo / sửa: thêm n test case> |

<Báo cáo đã lưu tại: <đường dẫn> — nếu có `save-report`>
