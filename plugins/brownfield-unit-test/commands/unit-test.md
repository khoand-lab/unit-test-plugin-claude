---
description: Viết unit test cho dự án brownfield (tham số key=value, xem argument-hint)
argument-hint: scope=<đường dẫn|Class#method|changed:<ref>> [mode=confirm|auto] [coverage-target=N] [max-fix-attempts=1..10] [failed-test-policy=remove|disable] [save-report=true|<file>]
---

Dùng skill `brownfield-unit-test` để viết unit test cho dự án hiện tại, làm đúng quy trình 5 pha của skill.

Tham số người dùng truyền vào (dạng `key=value`, có thể trống):

$ARGUMENTS

Nếu không có `scope`, hỏi người dùng trước khi làm bất cứ việc gì khác. Với `mode=confirm` (mặc định), dừng sau pha lập kế hoạch và chờ người dùng xác nhận trước khi ghi file.
